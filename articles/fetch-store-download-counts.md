---
title: "App StoreとGoogle Playのダウンロード数を自動で取ろうとしてはまった話"
emoji: "📊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "AppStoreConnect", "GooglePlay", "AWSLambda"]
published: false
---

# はじめに

個人で作っている猫育成ゲームで、毎日のダウンロード数をSlackに流したくなりました。ストアの管理画面を毎日見に行くのが面倒だったからです。

サーバー（Rust）に取得処理を書いて、EventBridgeのスケジュール（`cron(0 21 * * ? *)` = JST 6:00）で毎日叩くようにしたのですが、動くまでに4回はまったのでめもっとく。

<!-- TODO: そもそもなぜ毎日の数字を自動で取りたくなったか（何を判断したかったか）を本人が加筆 -->

# 1. Googleのトークン交換でgrant_typeがアンダースコアだった

まずAndroid側が動きませんでした。サービスアカウントのJWTをアクセストークンに交換するところで400です。

```
400 unsupported_grant_type
（Invalid grant_type: urn:ietf:params:oauth:grant_type:jwt-bearer）
```

RFC 7523の正しいURNは

```
urn:ietf:params:oauth:grant-type:jwt-bearer
```

で、`grant-type`の部分は**ハイフン**です。書いたコードは`grant_type`（アンダースコア）になっていました。プッシュ通知（FCM）用のトークン交換は別のファイルにあって、そっちはハイフンで正しく動いていました。

定数に切り出して、両方から使うようにしました。

```rust
const JWT_BEARER_GRANT_TYPE: &str = "urn:ietf:params:oauth:grant-type:jwt-bearer";
```

同じ文字列を2箇所に書いたら、片方だけ間違っていても気づけないな、と反省しました。

# 2. Play Developer Reporting APIにインストール数はなかった

トークン交換が通ったら、今度は404になりました。

Google Play Developer Reporting API（`playdeveloperreporting`）を使って`storePerformanceReport:query`を叩いていました。ただ、そもそもこのAPIはクラッシュやANRなどの**アプリの健全性メトリクス専用**で、インストール数やダウンロード数のメトリクスを持っていません。存在しないメソッドを叩いていたので404、というわけです。

トークン交換が壊れていた間はそこで止まっていたので、1つ目を直したことでようやくこの問題が見えた、という順番でした。

Googleの公式手順どおり、Play ConsoleがCloud Storageのバケット（`pubsite_prod_rev_<developer-id>`）に出している月次のインストールレポートCSVを読む方式に変えました。

```
stats/installs/installs_<package>_<YYYYMM>_overview.csv
```

やることはこうなります。

- OAuthのスコープを`playdeveloperreporting`から`devstorage.read_only`に変更する
- Cloud Storage XML APIでCSVを取得する
- **UTF-16（BOM付き）**なのでデコードする
- 対象日の Daily Device Installs（なければ Daily User Installs）を取る

CSVがUTF-16なのは知らないと引っかかると思います。あとサービスアカウントにバケットの読み取り権限（`roles/storage.objectViewer`など）を付ける必要があります。

# 3. iOSのSKUはBundle IDではなかった

iOS側はApp Store Connectの`salesReports` APIでレポートを取ってきて集計しています。こちらはエラーにならず、ただ**毎日0**でした。

原因は、レポートの`SKU`列をBundle IDだと思って絞り込んでいたことでした。

```
SKU が jp.example.app の行だけを集計する
→ 一致する行がないので全行が除外される
→ 常に0
```

`SKU`はApp Store Connectで自分が設定する任意の文字列で、Bundle IDとは別物です。というわけで、SKUでの絞り込みは撤廃して、ダウンロード種別（`1` / `1F` / `1T` / `F1`）のUnitsを全部合算するようにしました。アップデートや課金の行は除外されるように、種別で見ています。

複数アプリを出していて絞り込みたい場合のために、環境変数で任意のSKUを指定したときだけ絞れる、という逃げ道も残しています。

あと、同じことで悩まないように、レポートに出てくる `(SKU, Product Type Identifier)` のdistinctな組み合わせをログに出すようにしました。CloudWatchで実際の値が確認できるので、種別の漏れにも気づけます。

なお、そこに至る前に403も食らっていて、そちらはコードではなくApp Store Connect APIキーのロール（Sales / Finance）の権限不足でした。ダウンロード数を取るにはレポート用の権限があるキーが要ります。

# 4. Playのレポートは11日くらい遅れて出る

Android側もエラーにならず、`app_downloads`テーブルに1行も入っていませんでした。ログを見ると

```
date not found in installs report
```

対象日を「今日の3日前」にしていたのですが、Google PlayのインストールCSVは生成が大きく遅れていて、実測で**取得日の11日前くらいまで**しか行がありませんでした。なので毎回「その日の行がない」で失敗していました。

日付をもっと前にずらす手もありますが、遅延が一定とも限らないので、「対象日以前で確定している最新の日を採用する」ようにしました。

- 対象日の行があればそれを使う
- なければ対象日以前で一番新しい行を使い、**その実際の日付で**記録する
- 月初は前月のCSVも見る必要があるので、対象月と前月の両方をbest-effortで取る（404はその月が未生成として無視）
- 対象日と違う日を採用したときは、Slackの通知に `(latest available YYYY-MM-DD)` と付ける

記録する日付を「取ろうとした日」ではなく「実際に採用した日」にするのが大事で、ここを間違えると、遅延で同じ数字が別の日付に何回も入ることになります。

# おわりに

両ストアともAPIはあるものの、iOSは「SKUって何」、Androidは「そもそもそのAPIにインストール数がない」で、ドキュメントを読み直す時間のほうが長かったです。しかも失敗の仕方が「エラーになる」ではなく「毎日0が入る」「1行も入らない」だったので、動いていないことにしばらく気づけませんでした。

外部のレポート系APIを叩くときは、取れた値が0でも「取得成功」にせず、何を採用したかログに残しておくのが結局いちばん早いなと思います。

# 参考

- https://developer.apple.com/documentation/appstoreconnectapi/download_sales_and_trends_reports
- https://support.google.com/googleplay/android-developer/answer/6135870
- https://datatracker.ietf.org/doc/html/rfc7523
