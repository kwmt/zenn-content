---
title: "RevenueCatでサブスクとコイン課金を両方入れて分かったこと"
emoji: "💳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["RevenueCat", "iOS", "Android", "KotlinMultiplatform", "AppStore"]
published: false
---

# はじめに

個人で作っている猫育成ゲーム（KMP + Compose Multiplatform、サーバーはRust）に、月額のサブスクリプションと、消耗型のコイン購入の両方を入れました。課金まわりはRevenueCatを使っています。

「サブスクだけ」「単発課金だけ」ならそこまで難しくないと思うのですが、両方入れて、しかもプランのアップグレード・ダウングレードがあると、考えることがけっこう増えました。実装しながら分かったことをめもっとく。

<!-- TODO: RevenueCat を選んだ理由（StoreKit / Play Billing を直接使わなかった理由）を本人が加筆 -->

# 1. offeringを分けると、商品の探し方に気をつける

RevenueCatではコインとサブスクをそれぞれ別のofferingにしています。

- `coin_shop`（Current Offering）: `coin_150` / `coin_500` / `coin_1200` / `coin_3500`
- `subscription`: 月額プラン3つ

ここで一度やらかしていて、購入処理が「`subscription` offeringからだけ商品を探す」実装になっていたせいで、コインの購入が必ず「商品が見つかりません」で失敗する状態になっていました。しかもそれに気づいたのがApp Storeの審査でリジェクトされたときです。

商品IDから探すときは、Current Offeringだけでも特定のofferingだけでもなく、**全offeringのパッケージを集めてから**探すようにしました。このあたりの詳しい話は審査リジェクトの記事に書いています。

<!-- TODO: app-store-rejected-twice を公開したら、ここに https://zenn.dev/yasi/articles/app-store-rejected-twice を1行で置く -->

# 2. stagingと本番でproduct IDを分ける

サブスクの動作確認を本番のproduct IDでやりたくないので、staging / ローカルでは`stg_`プレフィックスのproduct IDを使うようにしました。RevenueCatのAPIキーも環境ごとに分けています。

サーバー側のWebhookも`stg_`付きのproduct IDを受け取れるようにする必要があります。ここを忘れると、stagingで購入してもサーバーのティアが上がらない、ということになります。

# 3. サーバーはDBのtierをそのまま信じない

契約状態はRevenueCatのWebhookで受けてDBに書いています。ただ、Webhookが必ず届く保証はないので、DBの`tier`カラムをそのまま返していると、期限切れのユーザーに有料特典が残り続けます。

なので、返すときに毎回「実効ティア」を計算するようにしました。

```rust:subscription.rs
pub fn effective_tier(
    tier: &str,
    status: &str,
    expires_at: Option<DateTime<Utc>>,
    grace_until: Option<DateTime<Utc>>,
    now: DateTime<Utc>,
) -> (String, String) {
    let parsed_tier = Tier::from_db_str_or_free(tier);

    // 猶予期間中は元の tier を維持する
    if let Some(grace) = grace_until {
        if grace > now {
            return (tier.to_string(), status.to_string());
        }
    }

    // expires_at が過去なら free にダウングレード
    if let Some(exp) = expires_at {
        // ...
    }
}
```

猶予期間（billing grace period）の間は元のティアを維持します。カード決済に失敗しただけで即座に機能が止まると、ユーザーからすると理不尽なので。

ステータスを返すAPIだけでなく、猫を作るAPIやテーマカラーを変えるAPIなど、ティアで制限がかかるところは全部この関数を通すようにしました。「DBのtierを直接見ている場所」が1つでも残っていると、そこだけ期限切れが効かなくなります。

# 4. ダウングレードは即時に切り替えられない

これは実装方針そのものが間違っていた話です。

iOSで「Premiumに加入中にLightを購入しても、Premiumの制限が外れない」というバグを見つけて、直そうとして調べたら、そもそも**Appleのサブスクリプショングループ内のダウングレードは「次回更新時に切り替わる」仕様**でした。現在の期間中は上位ティアの特典を維持することが求められていて、即時切り替えはStoreKit上で実装できません。

それまでの実装は、購入が成功したら楽観的更新で即座にLightに切り替えていました。つまり

- 課金はPremium分を払い続けているのに、Lightの制限がかかる（不利益変更）
- App Review Guideline 3.1.2(b) のシームレスなアップグレード / ダウングレード要件に反する可能性がある

という状態でした。動かないバグを直そうとしたら、方針が間違っていたことが分かった、というやつです。

Apple Music / Netflix / Spotify / Notion / iCloud+ あたりがどうしているかを見ても、みんな「次回更新時に切替」で、期間中は上位の特典が続く作りになっていました。なので、そちらに合わせて、ダウングレードの購入時は「次回更新日から新しいプランになります」という確認ダイアログを出すようにしました。

# 5. ダウングレードで超過したぶんはソフトロックにする

このアプリはプランによって飼える猫の数の上限が変わります。ダウングレードすると上限を超えるので、超えたぶんをどうするかを決める必要がありました。

消すのは論外なので、「ロック状態」にしてお世話やゲームができないようにして、データはそのまま持っておく方式にしました。再アップグレードすれば自動で解除されます。

ロックするのは**新しい順**です。長く育てている猫を守りたいので、あとから増やした猫からロックされます。

```
CatResponse に is_locked: bool を追加
compute_is_locked_flags() で「新しい順」から超過分をロック判定
（古参の猫を保護する顧客フレンドリー設計）
```

サーバー側の実装で気をつけたのは次の2点です。

- お世話のAPIにもロックのガードを入れる（一覧のフラグだけだとAPIを直接叩けば操作できてしまう）
- 猫の数が上限以下ならCOUNT 1回で早期returnする（大半のユーザーはここで抜ける）

並び順は`created_at`だけだと同時刻の猫で順序が揺れるので、`(created_at, id)`のタプルで比較して安定させています。

あとティアで制限しているものにテーマカラーがあるのですが、こちらは「DBの値は消さずにレスポンスだけマスクする」ようにしました。GETで値を書き換えると冪等でなくなるのと、再アップグレードしたときに元の色に戻ってほしいからです。

# おわりに

サブスクは「買ったら有効、期限が切れたら無効」くらいの単純な話だと思っていたのですが、猶予期間、Webhookの取りこぼし、ダウングレードの扱い、と考えることが多かったです。特にダウングレードは、ストアの仕様に合わせないとガイドライン違反にもなりうるので、実装する前に他のアプリがどうしているか見ておいたほうがよかったなと思います。

<!-- TODO: 実装してみての感想と、次にやりたいこと（トライアル、年額プランなど）を本人が加筆 -->

# 参考

- https://www.revenuecat.com/docs/offerings/overview
- https://developer.apple.com/app-store/subscriptions/
- https://developer.apple.com/app-store/review/guidelines/#subscriptions
