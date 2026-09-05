---
title: "sqlxでトランザクション中にプールからもう1本コネクションを取ってハングした話"
emoji: "🔒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "sqlx", "PostgreSQL", "AWSLambda"]
published: false
---

# はじめに

個人で作っている猫育成ゲームのサーバー（Rust + Axum + sqlx、AWS Lambda で動かしてます）で、報酬を受け取るAPIが応答しなくなりました。クライアント側は受け取りダイアログが閉じないまま、という状態です。

エラーも出ないでただ止まる、というやつだったので、原因が分かるまで少し時間がかかりました。同じことをやりそうなのでめもっとく。

<!-- TODO: どうやって気づいたか（自分で触ってて？ユーザーからの報告？）を本人が加筆 -->

# 前提: コネクションプールは1本

Lambdaで動かしているので、コネクションプールのサイズは1にしています。

```rust:lib.rs
// Lambda は 1 instance = 1 リクエスト処理なので接続は 1 で十分。
// Supabase Free の Session pooler 上限 (15) を 1 Lambda 5 接続で食い潰すと、
// 複数 Lambda 同時実行時に EMAXCONNSESSION で 500 になる
let mut pool_options = sqlx::postgres::PgPoolOptions::new()
    .max_connections(1)
    .min_connections(0)
    .acquire_timeout(std::time::Duration::from_secs(60));
```

Lambdaは1インスタンスが1リクエストしか処理しないので、1本で足ります。DB側の接続上限を食い潰さないためにも、少ないほうが都合がいいです。

これが今回の伏線でした。

# 症状

`POST /api/rewards/{slug}/claim` が応答しない。エラーにもならず、ただ返ってこない。

コードはこんな流れになっていました。

1. トランザクションを開始する
2. 報酬が「ルール型」なら、トランザクションの中で受け取り資格をもう一度判定する
3. 資格があればコインを付与して、受け取り済みにする
4. コミット

2番の判定をしている関数がこれです。

```rust:rewards.rs
async fn eval_rule<'e, E>(
    executor: E,
    db: &sqlx::PgPool,
    user_id: Uuid,
    rule_key: &str,
) -> Result<bool, AppError>
where
    E: sqlx::PgExecutor<'e>,
{
    match rule_key {
        RULE_BETA_GRATITUDE => {
            let thresholds = fetch_thresholds(db).await?;
            is_beta_eligible(executor, user_id, &thresholds).await
        }
        _ => Ok(false),
    }
}
```

`executor`と`db`の2つを受け取っています。呼び出し側はこう。

```rust:rewards.rs
let eligible = eval_rule(&mut *tx, &state.db, auth.user_id, key).await?;
```

トランザクションは`executor`として渡しているのですが、しきい値を取る`fetch_thresholds(db)`のほうは`&PgPool`を受け取っていて、プールから**別のコネクションを取りに行きます**。

# 原因

プールのコネクションは1本しかなくて、その1本はトランザクションが握っています。そこで`fetch_thresholds`が「コネクションを1本ください」と言うので、プールは空きが出るまで待ちます。でも空きが出るのはトランザクションが終わったときで、トランザクションは`fetch_thresholds`の結果を待っている。

```
TX（コネクション#1を保持）
  └─ fetch_thresholds → プールに「もう1本ください」
                          └─ 空くのを待つ（空くのはTXが終わったとき）
```

自分で自分を待っている状態です。`acquire_timeout`を60秒にしているので、正確には60秒後にタイムアウトするはずですが、体感としてはハングでした。

プールが5本とか10本あれば、たまたま空きがあって動いてしまうこともあると思います。プール1本の環境だと100%再現するので、逆に分かりやすかったかもしれません。

同じ判定を使っている`GET /api/rewards/claimable`のほうは、トランザクションを張っていないので普通に動いていました。「一覧では出るのに、受け取ろうとすると固まる」という症状だったのはこれが理由です。

# 対策

`&PgPool`を受け取るのをやめて、`&mut PgConnection`を受け取るようにしました。こうすると関数の中で勝手に別のコネクションを取りに行けなくなります。

```diff rust:rewards.rs
-async fn eval_rule<'e, E>(
-    executor: E,
-    db: &sqlx::PgPool,
+/// しきい値取得・eligibility 判定の全クエリを **同一コネクション**（`conn`）で実行する。
+/// claim はトランザクション中に呼ぶため、ここでプール（`&PgPool`）から別コネクションを
+/// 取りに行くと、TX がコネクションを保持したまま空きを待ち続けてデッドロック（応答ハング）
+/// する。必ず渡された 1 本のコネクション上で完結させること。
+async fn eval_rule(
+    conn: &mut sqlx::PgConnection,
     user_id: Uuid,
     rule_key: &str,
-) -> Result<bool, AppError>
-where
-    E: sqlx::PgExecutor<'e>,
-{
+) -> Result<bool, AppError> {
     match rule_key {
         RULE_BETA_GRATITUDE => {
-            let thresholds = fetch_thresholds(db).await?;
-            is_beta_eligible(executor, user_id, &thresholds).await
+            let thresholds = fetch_thresholds(&mut *conn).await?;
+            is_beta_eligible(&mut *conn, user_id, &thresholds).await
         }
         _ => Ok(false),
     }
 }
```

孫の関数（`fetch_thresholds` / `fetch_threshold` / `is_beta_eligible`）も全部`&mut PgConnection`に変えて、`&mut *conn`で下に渡していきます。これで判定に必要なクエリが全部1本のコネクションで完結します。

呼び出し側は、トランザクション内ならトランザクションのコネクションをそのまま渡します。

```diff rust:rewards.rs
-        let eligible = eval_rule(&mut *tx, &state.db, auth.user_id, key).await?;
+        let eligible = eval_rule(&mut *tx, auth.user_id, key).await?;
```

トランザクションを張っていない一覧のほうは、自分で1本取ってきて、使い終わったら返します。

```rust:rewards.rs
// 1 本のコネクションを取得し eval_rule 内で使い回す（TX ではないが
// eval_rule の規約に合わせ単一コネクションで完結させる）。判定後に解放する。
let mut conn = state.db.acquire().await?;
let eligible = eval_rule(&mut conn, auth.user_id, key).await?;
drop(conn);
```

# 学んだこと

ジェネリックな`E: PgExecutor`で受け取る書き方自体は悪くないのですが、今回は`executor`と`db`の**両方**を受け取っていたのが罠でした。呼び出し側は`&mut *tx`を渡しているので「トランザクションで実行される」と思い込んでいたのに、中で一部のクエリだけプールから別コネクションを取っていた、という状態です。

引数に`&PgPool`があったら、その関数はトランザクション中に呼べないと思ったほうがよさそうです。今は「トランザクション中から呼ばれる可能性がある関数は`&mut PgConnection`だけを受け取る」というのを規約にして、コメントにも書いておきました。

# おわりに

エラーが出ずにただ固まる系は、どこを見ればいいのか分からなくなるので苦手です。今回はプールが1本だったおかげで確実に再現したので、まだ分かりやすいほうでした。プールが多い環境で「たまに固まる」になってたら、もっと沼だったろうなと思います。

# 参考

- https://docs.rs/sqlx/latest/sqlx/trait.Executor.html
- https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html
