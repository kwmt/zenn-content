---
title: "個人開発で5つのターゲットを毎週火曜にリリースする運用にした"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHubActions", "fastlane", "iOS", "Android", "AWS"]
published: false
---

# はじめに

個人で作っている猫育成ゲームは、出すものが5つあります。

- API（Rust / AWS Lambda）
- Admin（管理画面 / Cloudflare Pages）
- Web（LP・利用規約など / Cloudflare Workers）
- iOS（App Store）
- Android（Google Play）

1人でやっていると「まとめてそのうち出す」になりがちで、実際しばらくそうなっていました。なので**毎週火曜に出す**と決めて、そのための手順を書きました。決めてからのほうが結果的に楽だったので、やっていることをめもっとく。

<!-- TODO: 毎週リリースにする前はどういう出し方をしていたか、何が困ったかを本人が加筆 -->

# タグをpushしたらデプロイされる

まず、本番デプロイは全部「タグをpushしたらGitHub Actionsが動く」に統一しています。stagingは`main`にマージしたら自動です。

| 環境 | トリガー | デプロイ先 |
|------|---------|-----------|
| staging (API) | `main` へ push | AWS Lambda (ECR) |
| production (API) | `api/v*` タグ push | AWS Lambda (ECR) |
| production (Admin) | `admin/v*` タグ push | Cloudflare Pages |
| production (Web) | `web/v*` タグ push | Cloudflare Workers |
| Android Closed Testing | `android/v*+*` タグ push | Google Play (Closed Testing) |
| Android Production | `android-production/v*+*` タグ push | Google Play (Production / draft) |
| インフラ | `main` へ push + `infra/terraform/**` | Terraform apply |

タグの名前はこうしています。

```
api/v{YYYY-MM-DD}.{N}                              # 例: api/v2026-02-25.1
admin/v{YYYY-MM-DD}.{N}
web/v{YYYY-MM-DD}.{N}
ios/v{MARKETING_VERSION}+{BUILD_NUMBER}            # 例: ios/v0.1.3+26
android/v{versionName}+{versionCode}               # 例: android/v0.1.0+4
android-production/v{versionName}+{versionCode}
```

バックエンド側は日付ベース（`N`はその日の連番）、モバイルはアプリのバージョン + ビルド番号にしています。モバイルはXcodeプロジェクトや`build.gradle.kts`から取った値をそのまま使うので、タグを見ればどのビルドが本番に出ているか分かります。

タグは必ず`main`上で作ります。AWSのOIDCの信頼ポリシーが`refs/heads/main`と`refs/tags/api/v*`しか許可していないので、それ以外のブランチで作ったタグからデプロイしようとすると認証で落ちます。

# ベイク期間を置かない

テスト配信で1週間寝かせてから本番、という運用にはしませんでした。1人でやっていると、寝かせている間に次の変更が積まれて、結局「今どれが本番と同じなのか」が分からなくなるからです。

代わりに、その週の`main`をそのまま火曜に出して、リスクは次の2つで吸収する方針にしています。

- iOS: TestFlightの内部配布ですぐ配って、自分で主要フローを触ってから本番に昇格する
- Android: 段階ロールアウト（10% → 50% → 100%）で、クラッシュ率とANRを見ながら上げる

DBマイグレーションや壊れると痛い変更がある週だけは、前週のうちに`main`に入れてstagingで数日泳がせてから火曜に乗せる、という例外にしています。

# iOSだけローカルから出す

iOSの本番ビルドと配信だけはGitHub Actionsからではなく、ローカルのfastlaneでやっています。

- TestFlightへのアップロードはActionsからもできる
- 本番（審査提出）はローカルのfastlaneで実行する

本番の昇格は、直前にアップロードしたTestFlightのビルドを**再ビルドせずにpromote**して、`submit_for_review: true`で審査に出します。審査が通ったあとの公開はデフォルト手動（`auto_release: false`）にしていて、即時公開したいときだけ明示的に指定します。

Androidは`release_status: "draft"`でアップロードするので、Play Consoleで手動でロールアウトを開始する必要があります。ここは自動にしないほうが安心かなと思っています。

# 準備だけ自動化する

火曜の朝にやることは、だいたい決まっています。

1. 前回のデプロイタグからの差分を、5つのターゲットそれぞれで見る
2. CIとテストの状態を確認する
3. リリースノートを書く
4. どれを出すか決める

このうち1〜3は毎回同じなので、Claude Codeのスケジュール機能で火曜10時にルーティンを動かして、**リリース対象・差分・実行すべきコマンドをまとめたGitHub issueを起票する**ところまでを自動にしました。デプロイ自体はやりません。

自分がやるのは、起票されたissueを見て、ローカルでデプロイのコマンドを叩くところからです。実デプロイまで自動にしなかったのは、iOSはリモートから出せないのと、本番事故のときの影響が大きいからです。

```
火曜 10:00  ルーティン（自動）  差分検出・リリースノート下書き → issue 起票
火曜 午前    自分              issue を確認してローカルでデプロイ
火曜 午後〜  自分              リリース後の確認、Androidの段階ロールアウト監視
水〜金       -                 通常開発。緊急修正だけ随時 hotfix
```

火曜が祝日の週は前倒しか後ろ倒しにするか、その週はスキップします。

# おわりに

「毎週火曜に出す」と決めただけで、変更を溜め込まなくなったのと、1回のリリースが小さくなって確認が楽になりました。差分が1週間ぶんなら、何かおかしくなってもだいたい心当たりがあります。

準備を自動化して実行は人がやる、という分け方も今のところちょうどいいです。全部自動にすると怖いし、全部手でやると面倒でやらなくなるので。

<!-- TODO: 実際に何週くらい続いているか、リリースで事故ったことがあるかを本人が加筆 -->

# 参考

- https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- https://docs.fastlane.tools/actions/upload_to_app_store/
- https://support.google.com/googleplay/android-developer/answer/6346149
