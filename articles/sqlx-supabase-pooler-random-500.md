---
title: "SupabaseのTransaction poolerでsqlxがランダムに500を返していたときのメモ"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Supabase", "sqlx", "AWSLambda"]
published: false
---

# はじめに

個人で作っているアプリのサーバー（Rust / Axum 0.8 / sqlx 0.8 / PostgreSQL。AWS Lambdaにコンテナイメージで乗せてます。DBはSupabase）で、stagingのAPIがたまに500を返す、という状態がしばらく続いてました。

<!-- TODO: 気づいたきっかけ（アプリを触ってて気づいたのか、ログを見て気づいたのか）と、最初に何を疑ったかを本人が加筆 -->

ログを見るとSupabaseのpooler（Supavisor）とsqlxのprepared statementの相性問題だったので、直すまでの流れをメモしておきます。対応したのは2026年5月13日〜14日です。

# 前提の構成

SupabaseへはSupavisor（Shared pooler）経由でつないでいて、接続用のURLを2つ持ってました。

| 接続方式 | ポート | 用途 |
| --- | --- | --- |
| Transaction pooler | 6543 | `DATABASE_URL`（通常のDB操作） |
| Session pooler | 5432 | `DIRECT_DATABASE_URL`（マイグレーション） |

Supabaseのドキュメント https://supabase.com/docs/guides/database/connecting-to-postgres にTransaction modeはこう書いてあるので、Lambdaならこっちだろうと思ってアプリ用はTransaction poolerにしてました。

> This is ideal for serverless or edge functions, which require many transient connections.

> 多数の一時的な接続を必要とするserverlessやedge functionに最適。

ちなみにprepared statementの衝突自体は今回が初めてではなくて、2月に一度踏んでます。そのときのコミット。

```
🐛(api): Supavisor 対応で prepared statement キャッシュを無効化

- statement_cache_capacity(0) で sqlx のプリペアドステートメントキャッシュを無効化
- Supavisor (transaction mode) で「prepared statement already exists」エラーを解消
```

```rust:server/crates/api/src/lib.rs
        // Supavisor (transaction mode) ではプリペアドステートメントが接続間で衝突するため無効化
        .statement_cache_capacity(0);
```

その翌日には`sqlx::migrate!()`もTransaction poolerだと動かなかったので、マイグレーションだけSession poolerにつなぐようにしてます。

```
🐛(server): マイグレーション用に直接DB接続をサポート

Supavisor transaction mode (port 6543) では sqlx::migrate!() が内部で
使用する prepared statement が衝突するため、DIRECT_DATABASE_URL 環境変数で
session mode pooler (port 5432) 経由の直接接続を指定可能にした。
```

なので「Transaction poolerだとprepared statementが衝突する」のは知ってたし、`statement_cache_capacity(0)`で対策済みのつもりでした。

# 症状

stagingで複数のエンドポイントがランダムに500を返す。issueに書いた当時の記録がこれです。

> - 2026-05-13 09:30 JST: `GET /api/rankings/friendship?period=weekly&limit=100` が 500
> - 2026-05-12 19:54 JST: `/api/cats` `/api/subscription/status` `/api/missions/daily` `/api/login-bonus/status` `/api/banners` が同時に 500

レスポンスはこれだけ。

```json
{"error":"database error","error_code":"internal_server_error"}
```

毎回ではなくランダムなので、アプリ側でサーバーの生メッセージをそのまま出さないようにする対応を先に入れて、API側の調査は別issueにしました。

# ログを見る

CloudWatch Logsを`aws logs tail /aws/lambda/<function-name>`で追うと、こういうエラーが混ざってました。

```
prepared statement "sqlx_s_3" already exists
prepared statement "sqlx_s_4" does not exist
unnamed prepared statement does not exist
bind message supplies 3 parameters, but prepared statement "" requires 1
bind message supplies 0 parameters, but prepared statement "" requires 8
error occurred while decoding column "id": invalid length: expected 16 bytes, found 4
error occurred while decoding column "breed_id": invalid length: expected 16 bytes, found 36
```

`prepared statement "sqlx_s_3" already exists`は2月に見たやつと同じです。`statement_cache_capacity(0)`を入れてるのに...！？

それより気になるのが下の2行で、`expected 16 bytes, found 4`はUUID（16バイト）のカラムを読もうとしたら4バイトの整数が返ってきてます。つまり別のクエリの結果を受け取ってる。`bind message supplies 3 parameters, but prepared statement "" requires 1`も、自分がParseしたのとは別のstatementにBindしようとしてる形です。

issueにはこう整理してました。

> これは AWS Lambda + SQLx + PgBouncer (Supavisor transaction mode) の有名な競合:
>
> 1. SQLx は PostgreSQL の Extended Query Flow を使う（unnamed prepared statement 含む）
> 2. Lambda が同時実行されると複数インスタンスが同じ Pooler 接続を再利用
> 3. プリペアドステートメントの状態が混線し、別クエリの結果が返ってくる / バインドパラメータ数が合わない

# statement_cache_capacity(0)を入れてるのになぜ？

sqlxのドキュメント https://docs.rs/sqlx/0.8.6/sqlx/postgres/struct.PgConnectOptions.html#method.statement_cache_capacity にはこうあります。

> Sets the capacity of the connection's statement cache in a number of stored distinct statements. Caching is handled using LRU, meaning when the amount of queries hits the defined limit, the oldest statement will get dropped. The default cache capacity is 100 statements.

> 接続のstatement cacheの容量を、保持するstatementの数で設定する。キャッシュはLRUで、上限に達したら一番古いstatementが捨てられる。デフォルトは100。

これはあくまで「名前付き」のprepared statement（ログに出てる`sqlx_s_3`みたいなやつ）を接続にキャッシュして使い回すかどうかの話です。

一方でPostgreSQLのExtended Queryプロトコルには「無名」のprepared statementというのがあって、名前が空文字（`""`）のやつです。 https://www.postgresql.org/docs/current/protocol-flow.html

> An unnamed prepared statement lasts only until the next Parse statement specifying the unnamed statement as destination is issued. (Note that a simple Query message also destroys the unnamed statement.)

> 無名のprepared statementは、次に無名statementを宛先にしたParseメッセージが来るまでしか生きない。（単純なQueryメッセージでも無名statementは破棄される）

> If successfully created, a named prepared-statement object lasts till the end of the current session, unless explicitly destroyed.

> 名前付きのprepared statementは、作成に成功したら明示的に破棄しない限りセッションの終わりまで生きる。

ログの`prepared statement "" requires 1`の`""`がまさに無名のほうです。PRにはこう書きました。

> `statement_cache_capacity(0)` で名前付き prepared statement は無効化済みだったが、
> PostgreSQL Extended Query Flow が使う**無名 ("") prepared statement** はクライアントから
> 制御不能なため根本解決にならず、上記が解消できなかった。

Supabase側のドキュメントも改めて見ると、Transaction modeの説明にはっきり書いてありました。

> Transaction mode does not support prepared statements.

> Transaction modeはprepared statementをサポートしていない。

SupavisorのFAQ https://supabase.com/docs/guides/troubleshooting/supavisor-faq-YyP5tI にはTransaction modeとSession modeの違いがこう書いてあります。

> In transaction mode, a client is allowed to make a single query before being sent back to the figurative 'waiting room'.

> transaction modeでは、クライアントは1つクエリを実行したら、いわば「待合室」に送り返される。

> In session mode, once the pooler assigns a direct connection, it stays with that client until voluntarily surrendered.

> session modeでは、poolerが接続を割り当てたら、クライアントが自分から手放すまでその接続はそのクライアントのもの。

つまり名前付きのキャッシュを切ったところで、sqlxがprepared statementを使ってること自体は変わらないので、Transaction modeでは根本的にダメ、ということのようです。多くのクエリに`.persistent(false)`もつけてました（issueでは「部分対応」扱い）が、これも無名のprepared statementを使う設定なので同じことです。

ちなみにsqlx 0.8.6のソース（`sqlx-postgres/src/connection/executor.rs`）を見ると、名前付きか無名かを決めてるのは`statement_cache_capacity`ではなく`persistent`のほうでした。

```rust
    let id = if persistent {
        let id = conn.inner.next_statement_id;
        conn.inner.next_statement_id = id.next();
        id
    } else {
        StatementId::UNNAMED
    };
```

`persistent`のドキュメント https://docs.rs/sqlx/0.8.6/sqlx/query/struct.Query.html#method.persistent

> If `true`, the statement will get prepared once and cached to the connection's statement cache. If queried once with the flag set to `true`, all subsequent queries matching the one with the flag will use the cached statement until the cache is cleared. If `false`, the prepared statement will be closed after execution. Default: `true`.

> `true`ならstatementは一度prepareされて接続のstatement cacheに入り、以降同じクエリはキャッシュされたstatementを使う。`false`ならprepared statementは実行後にcloseされる。デフォルトは`true`。

なので`statement_cache_capacity(0)`だけだと、`.persistent(false)`をつけてないクエリは名前付きの`sqlx_s_N`をキャッシュせずに毎回作る、という動きになります。

<!-- TODO: 要確認。上の sqlx ソースの読みは下書き時に AI が sqlx-postgres 0.8.6 のソースで確認したもので、リポジトリ内のコメント「unnamed PS だけ使う設定 (capacity=0) なら名前衝突は発生しない」と食い違う。本人が確認して、この段落を残すか消すか決める -->

# 対応案

issueには「A. アプリもDIRECT_DATABASE_URL（Session pooler）を使う」「B. Lambda concurrency = 1に制限」「C. sqlxのsimple_queryモードに切り替え」「D. Supabase側のPoolerをsession modeに切り替え」の4案を書いて、Aにしました。

> ### A. アプリケーションも DIRECT_DATABASE_URL (Session Pooler) を使う
> - 接続が同一セッションに固定されるので prepared statement 競合がなくなる
> - 制約: pool size の上限 (Supabase Free: 15)、IPv4 専用
> - Lambda concurrency と pool size のバランス調整が必要
>
> 実用的には **A**（DIRECT_DATABASE_URL の利用）が最も筋が良いと考えている。マイグレーションでは既に DIRECT を使っているので、メインクエリも切り替えるだけ。

# Session poolerに切り替える

`create_app`でプールを作るところを、`DIRECT_DATABASE_URL`があればそっちを使うようにしました。

```diff rust:server/crates/api/src/lib.rs
 pub async fn create_app(config: Config) -> Router {
-    let connect_options = config
-        .database_url
+    // Supavisor の transaction pooler (DATABASE_URL, port 6543) では SQLx の
+    // unnamed prepared statement が Lambda 同時実行時に他コネクションと混線し、
+    // "bind message supplies N parameters, but prepared statement \"\" requires M" 等の
+    // ランダムな 500 を引き起こす。Session pooler (DIRECT_DATABASE_URL, port 5432) なら
+    // クライアント側コネクションが単一の Postgres バックエンドに固定されるため安全。
+    // DIRECT_DATABASE_URL が未設定の環境（ローカル開発等）は従来通り DATABASE_URL を使う。
+    let (pool_url, using_session_pooler) = match config.direct_database_url.as_deref() {
+        Some(url) => {
+            tracing::info!("Using DIRECT_DATABASE_URL (Session pooler) for application pool");
+            (url, true)
+        }
+        None => {
+            tracing::warn!(
+                "DIRECT_DATABASE_URL not set; falling back to DATABASE_URL. Prepared statements may conflict if this points at a transaction pooler."
+            );
+            (config.database_url.as_str(), false)
+        }
+    };
+
+    let mut connect_options = pool_url
         .parse::<sqlx::postgres::PgConnectOptions>()
-        .expect("Invalid DATABASE_URL")
+        .expect("Invalid database URL")
         .ssl_mode(if std::env::var("SSL_REQUIRE").is_ok() {
             sqlx::postgres::PgSslMode::Require
         } else {
             sqlx::postgres::PgSslMode::Prefer
-        })
-        // Supavisor (transaction mode) ではプリペアドステートメントが接続間で衝突するため無効化
-        .statement_cache_capacity(0);
+        });
+
+    // Session pooler に繋いでいる時は prepared statement を再利用しても安全。
+    // フォールバックで transaction pooler に当たってしまったときの保険として
+    // キャッシュを無効化する。
+    if !using_session_pooler {
+        connect_options = connect_options.statement_cache_capacity(0);
+    }
```

この時点では「Session poolerなら名前付きのprepared statementも安全」と思っていたので、Session poolerのときは`statement_cache_capacity(0)`を外してます（これが後で誤りだったとわかります）。

SQSから起動するpush通知用のLambda（`push_worker.rs`）にも同じロジックを入れました。

# デプロイしたけど直らない

マージしてstagingのログを見ると...

```
WARN  DIRECT_DATABASE_URL not set; falling back to DATABASE_URL.
ERROR database error: prepared statement "sqlx_s_6" already exists
ERROR Migration failed: prepared statement "sqlx_s_3" already exists
```

`DIRECT_DATABASE_URL not set`...！？

Lambdaの環境変数はTerraformで渡してるのですが、`DIRECT_DATABASE_URL`をそもそも配線してませんでした。そのときは「マイグレーションが正常実行されているので設定済みの前提」と思っていたのですが、実際は未設定でした。

<!-- TODO: DIRECT_DATABASE_URL 未設定でこれまでマイグレーションがどう動いていたか（気にしてなかったなら「気にしてなかった」でよい）を本人が加筆 -->

```diff hcl:infra/terraform/variables.tf
+variable "direct_database_url" {
+  description = "Supabase Session pooler URL (port 5432) used by application pool to avoid SQLx prepared statement conflicts under Supavisor transaction mode."
+  type        = string
+  sensitive   = true
+}
```

```diff hcl:infra/terraform/lambda.tf
   environment {
     variables = {
       DATABASE_URL                  = var.database_url
+      DIRECT_DATABASE_URL           = var.direct_database_url
       JWT_SECRET                    = var.jwt_secret
```

push worker側の`sqs.tf`も同様です。GitHub Actions側は`TF_VAR_direct_database_url`をenvironment secretから流すようにしました。値はSupabaseのDashboard → Connect → Session poolerの接続文字列で、port 5432のこういう形です。

```
postgresql://<user>:<password>@<host>:5432/postgres
```

もう1つ、このPRで`statement_cache_capacity(0)`をSession poolerでも常に適用するように戻しました。PRの説明。

> このときは「Session pooler なら named PS も安全」と判断したが、これは誤り。
> Supabase Session pooler (Supavisor) でも backend connection が後続セッションに
> recycle されるため、過去セッションが作った `sqlx_s_N` が新セッションの
> `PREPARE` で `"already exists"` を引き起こす。**unnamed PS のみを使う設定（capacity=0）
> なら名前衝突しない**ため、こちらを常時適用する。

<!-- TODO: Session pooler で "already exists" を実際に観測したのか（staging のログ or ローカル検証）、それとも #674 の時点の判断だったのかを本人が加筆 -->

最終的にプールを作るところはこうなりました。

```rust:server/crates/api/src/lib.rs
    // statement_cache_capacity(0) は Session pooler / Transaction pooler のどちらでも必須。
    // - Transaction pooler: 接続が transaction 単位で別 backend に切り替わるため named PS が衝突
    // - Session pooler: backend connection が後続セッションに recycle されるため、
    //   過去セッションが作成した sqlx_s_N が新セッションに残り `PREPARE` 時に
    //   "prepared statement \"sqlx_s_N\" already exists" になる。
    //   unnamed PS だけ使う設定 (capacity=0) なら名前衝突は発生しない。
    let _ = using_session_pooler; // 現状はログ用途のみ
    let connect_options = pool_url
        .parse::<sqlx::postgres::PgConnectOptions>()
        .expect("Invalid database URL")
        .ssl_mode(if std::env::var("SSL_REQUIRE").is_ok() {
            sqlx::postgres::PgSslMode::Require
        } else {
            sqlx::postgres::PgSslMode::Prefer
        })
        .statement_cache_capacity(0);
```

# 今度はEMAXCONNSESSION

Session poolerに切り替えたあと、stagingで30並列のburstテストをすると今度は別のエラーが出ました。

```
ERROR (EMAXCONNSESSION) max clients reached in session mode - max clients are limited to pool_size: 15
```

Supabase FreeのSession poolerは15接続が上限で、Lambda 1つあたり`max_connections(5)`だったので、4インスタンス以上同時に動くと超えます。Lambdaは1インスタンスで1リクエストしか処理しないし、ハンドラの中で並列にDBを叩いてる箇所もない（`tokio::join!`や`for_each_concurrent`をgrepして確認）ので、1に減らしました。

```diff rust:server/crates/api/src/lib.rs
+    // Lambda は 1 instance = 1 リクエスト処理なので接続は 1 で十分。
+    // Supabase Free の Session pooler 上限 (15) を 1 Lambda 5 接続で食い潰すと、
+    // 複数 Lambda 同時実行時に EMAXCONNSESSION で 500 になる (検証時に観測)。
     let mut pool_options = sqlx::postgres::PgPoolOptions::new()
-        .max_connections(5)
+        .max_connections(1)
         .min_connections(0)
         .acquire_timeout(std::time::Duration::from_secs(60));
```

PRに書いたビフォーアフター。

| | Before (5) | After (1) |
|---|---|---|
| 同時実行可能 Lambda 数 (Session pooler 15 内) | 3 | **15** |
| 1 instance のリクエスト処理速度 | 同じ | 同じ |
| EMAXCONNSESSION エラー | 4 instance〜 | 16 instance〜 |

push workerのほうは`for_each_concurrent(10, ...)`の中でDBを触るので`max_connections(2)`のままです。

# マイグレーションの注意

`sqlx::migrate!()`はTransaction poolerだと動きません。これは2月の時点で踏んでたので、マイグレーションだけ別のプールを作ってSession pooler経由で実行してます。

```rust:server/crates/api/src/lib.rs
        // Supavisor transaction mode (port 6543) では sqlx::migrate!() が内部で使う
        // prepared statement が衝突するため、マイグレーション専用に別の接続を使う
        // 注: Render.com Free plan は IPv4 のみ対応のため、Supabase Direct 接続 (IPv6) は使用不可
        // DIRECT_DATABASE_URL には Session pooler (port 5432, IPv4 対応) を設定すること
```

Direct connection（IPv6のみ）は使ってないので、Session poolerが実質唯一の選択肢でした。Supabaseのドキュメントでもこう書いてあります。

> This is only recommended as an alternative to a Direct Connection when connecting from an IPv4-only network.

> （Session modeは）IPv4のみのネットワークから接続するときの、Direct Connectionの代替としてのみ推奨。

あと、Lambdaが複数同時にコールドスタートするとマイグレーション用の接続でも15の上限に当たるので、そこはリトライしてます。

```rust:server/crates/api/src/lib.rs
            // Session pooler の接続上限（pool_size: 15）に達する場合があるためリトライする。
            // Lambda 複数インスタンスが同時にコールドスタートすると接続が競合する。
```

# おわりに

SupabaseのドキュメントにはTransaction modeは「serverlessに最適」とも「prepared statementは非対応」とも書いてあって、sqlxはprepared statement前提なので、この組み合わせだとSession pooler一択だったようです。`statement_cache_capacity(0)`で対策した気になってたのがよくなかったかなと思います。
Session poolerは接続数の上限が15（Free）なので、Lambdaが16インスタンス以上同時に動いたらまたEMAXCONNSESSIONになるはずで、そのときはまた考えます。

<!-- TODO: 切り替え後に他に困ったこと（レイテンシの変化、本番での再発有無など）があれば本人が加筆 -->

# 参考

- https://supabase.com/docs/guides/database/connecting-to-postgres
- https://supabase.com/docs/guides/troubleshooting/supavisor-faq-YyP5tI
- https://docs.rs/sqlx/0.8.6/sqlx/postgres/struct.PgConnectOptions.html#method.statement_cache_capacity
- https://docs.rs/sqlx/0.8.6/sqlx/query/struct.Query.html#method.persistent
- https://www.postgresql.org/docs/current/protocol-flow.html
