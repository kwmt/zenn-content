---
title: "App Storeの審査で2回連続リジェクトされたときのメモ"
emoji: "🐱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["iOS", "AppStore", "KotlinMultiplatform", "RevenueCat"]
published: false
---

# はじめに

個人で作っているねこ育成ゲーム（クライアントはKMP + Compose Multiplatform、サーバーはRust、課金はRevenueCat）のiOS版を初めてApp Storeに出したら、2回連続でリジェクトされました。
指摘されたガイドラインと直し方を忘れそうなのでメモしておきます。

提出したのは v0.2.2 (build 50) です。Appleからの指摘文はそのまま貼って、下に自分の訳を置いてます（アプリ名と商品名だけ伏せてます）。

<!-- TODO: リジェクト通知を見たときの状況（メール？App Store Connect？何をしてたか）を本人が加筆 -->

# 1回目のリジェクト（5月5日、build 50）

指摘は5件でした。1件ずつ書いていきます。

## Guideline 5.1.1(iv): ATTの事前説明に「あとで設定する」ボタンがあった

> Guideline 5.1.1(iv) - Legal - Privacy - Data Collection and Storage
>
> The app includes App Tracking Transparency permissions requests, but it encourages or directs users to accept tracking. Specifically, the app directs the user to accept tracking in the following way(s):
>
> - A message appears before the permission request, and the user can close the message and delay the permission request with the "Set up later" button. The user should always proceed to the permission request after the message.
>
> Permission requests give users control of their personal information.

> アプリにはApp Tracking Transparencyの許可リクエストが含まれていますが、ユーザーにトラッキングを許可するよう促したり誘導したりしています。具体的には、以下の方法でユーザーをトラッキング許可に誘導しています。
>
> - 許可リクエストの前にメッセージが表示され、ユーザーは「あとで設定する」ボタンでメッセージを閉じて許可リクエストを先送りできます。メッセージの後は必ず許可リクエストに進む必要があります。
>
> 許可リクエストは、ユーザーが自分の個人情報をコントロールするためのものです。

直し方も書いてありました。

> To resolve this issue, please revise the permission request process in the app to not include an exit button on the message before the tracking request.

> この問題を解決するには、トラッキングリクエスト前のメッセージに離脱ボタンを含めないよう、アプリの許可リクエストの流れを修正してください。

オンボーディングの5ページ目に「広告についてのおねがい」という説明ページを置いていて、「次へ進む」ボタンを押すと `ATTrackingManager.requestTrackingAuthorization` が呼ばれてシステムダイアログが出る、という作りでした。その「次へ進む」の下に「あとで設定する」というテキストリンクを置いていて、これを押すとATTダイアログを出さずに次のページに進めるようになっていました。これがダメだったようです。

<!-- TODO: なぜ「あとで設定する」を置いたのか（ユーザーに強制したくなかった等）を本人が加筆 -->

対応は「あとで設定する」を消すだけ...ではなく、TopBarの戻る矢印で前のページに戻ってから別ルートで進む、という抜け道もあったのでそっちも塞ぎました。ATTダイアログで拒否された後に出す「スキップ」は、すでにシステムダイアログを通過しているので残してます。

```diff kotlin:OnboardingPagerScreen.kt
+    // ATT 事前モーダル状態（page 5 かつ attDenied=false）では戻る矢印を非表示にする。
+    // App Store Review Guideline 5.1.1(iv): ATT システムダイアログ表示前に許可リクエストへの
+    // 進行を回避できる動線（戻る含む）を設けることは禁止されている。
+    val isAttPrePermission = pagerState.currentPage == 5 && !uiState.attDenied
     Scaffold(
         topBar = {
-            if (pagerState.currentPage > 0) {
+            if (pagerState.currentPage > 0 && !isAttPrePermission) {
```

```diff kotlin:OnboardingPagerScreen.kt
-            if (pagerState.currentPage == 5) {
+            // ATT 事前モーダル状態（attDenied=false）では skip 導線を出さない。
+            // App Store Review Guideline 5.1.1(iv) により、ATT システムダイアログ表示前に
+            // ユーザーが許可リクエストへ進むのを回避できる動線を設けることは禁止されている。
+            // 拒否後（attDenied=true）の「スキップ」はシステムダイアログを既に通過しているため許容。
+            if (pagerState.currentPage == 5 && uiState.attDenied) {
```

```diff xml:values-ja/strings.xml
     <string name="onboarding_att_next">次へ進む</string>
-    <string name="onboarding_att_skip">あとで設定する</string>
```

ちなみにATTダイアログを出すところはKotlin/NativeからそのままiOSのAPIを呼んでます。

```kotlin:IosAttPermissionRequester.kt
override suspend fun requestPermission(): AttPermissionResult = suspendCoroutine { continuation ->
    ATTrackingManager.requestTrackingAuthorizationWithCompletionHandler { status ->
        val result = when (status) {
            ATTrackingManagerAuthorizationStatusAuthorized -> AttPermissionResult.Granted
            ATTrackingManagerAuthorizationStatusDenied,
            ATTrackingManagerAuthorizationStatusRestricted,
            ATTrackingManagerAuthorizationStatusNotDetermined -> AttPermissionResult.Denied
            else -> AttPermissionResult.Denied
        }
        continuation.resume(result)
    }
}
```

## Guideline 2.1(b): コインを買おうとしたらエラーになった

> Guideline 2.1(b) - Performance - App Completeness
>
> The In-App Purchase products in the app exhibited one or more bugs which create a poor user experience. Specifically, an error occurred when we tried to purchase the coins.
>
> Review device details:
> - Device type: iPhone 17 Pro Max
> - OS version: iOS 26.4.2

> アプリ内課金の商品に、ユーザー体験を損なうバグが1つ以上ありました。具体的には、コインを購入しようとしたときにエラーが発生しました。

コード見ると、原因は1ヶ月前に自分で入れたバグでした...！？

サブスクリプションの購入が動かなくて直した修正で、`RevenueCatPurchaseService.purchase()` を「`subscription` offeringからだけ商品を探す」ように変えていました。

```diff kotlin:RevenueCatPurchaseService.kt
     override suspend fun purchase(productIdentifier: String): StorePurchaseResult {
-        val packages = cachedPackages ?: client.fetchCurrentOfferingPackages()
+        // サブスクリプション商品は "subscription" offering に含まれる
+        val packages = fetchSubscriptionPackages()
         val packageInfo = packages.firstOrNull { it.productIdentifier == productIdentifier }
-            ?: return StorePurchaseResult.Error("商品が見つかりません")
+            ?: return StorePurchaseResult.Error("商品が見つかりません: $productIdentifier")
```

RevenueCatのCurrent Offeringは `coin_shop` で、コイン商品（`coin_150` / `coin_500` / `coin_1200` / `coin_3500`）はそっちに入っているので、`subscription` offeringだけ見ても見つからず、コイン購入は必ず「商品が見つかりません」で失敗する状態になっていました。サブスクを直してコインを壊してた、というやつです。しかもコメントには「全 offerings から指定 ID で取得する」と書いてあるのに実装は `subscription` しか見てない、という矛盾つき。

修正は全offeringsのパッケージを集めてから `productIdentifier` で探すようにしただけです。

```diff kotlin:RevenueCatPurchaseService.kt
-    /**
-     * "subscription" offering からパッケージを取得する。
-     * Current Offering は "coin_shop" のため、全 offerings から指定 ID で取得する。
-     */
-    private suspend fun fetchSubscriptionPackages(): List<PackageInfo> {
+    /**
+     * 全 offerings からパッケージを集約して返す。
+     * Current Offering は "coin_shop"、サブスクリプション商品は "subscription" offering に
+     * 含まれているため、両方をカバーする必要がある。
+     */
+    private suspend fun fetchAllPackages(): List<PackageInfo> {
         val offerings = client.fetchOfferings()
-        Napier.d(tag = "RevenueCat") {
+        Napier.d(tag = TAG) {
             "Offerings: ${offerings.map { o -> "${o.identifier} (${o.availablePackages.map { it.productIdentifier }})" }}"
         }
-        val subscriptionOffering = offerings.firstOrNull { it.identifier == SUBSCRIPTION_OFFERING_ID }
-        Napier.d(tag = "RevenueCat") {
-            "Subscription offering packages: ${subscriptionOffering?.availablePackages?.map { it.productIdentifier }}"
-        }
-        return subscriptionOffering?.availablePackages ?: emptyList()
+        return offerings.flatMap { it.availablePackages }
     }
```

あわせて `fetchOfferings()` のネットワーク例外も `try` の中で拾うようにして、失敗したときは `Napier.e` でサーバーにログを送るようにしました（それまでは例外が `try` の外で起きて沈黙してました）。coin_shop / subscription 両方のofferingから探せるかのユニットテストも足してます。

## Guideline 2.1(b) Information Needed: サブスクの商品がアプリ内で見つからない

> Guideline 2.1(b) - Information Needed
>
> We have started the review of the app, but we are not able to continue because we cannot locate the In-App Purchases, such as （サブスク3プランの商品名）, within the app at this time.

> アプリの審査を開始しましたが、（サブスク3プランの商品名）などのアプリ内課金がアプリ内で見つからないため、審査を続行できません。

これは `feat_subscription` というFeature Flagでサブスクへの導線を全部隠していたのが原因でした。コインショップのバナー、パートナー追加の上限ダイアログ、フレンド一覧のバッジ、全部このフラグでガードしていたので、審査担当者からはサブスクに辿り着けなかったわけです。

じゃあadminからフラグをONにすればいいかというと、そうもいかなくて

- サーバー側のバリデーションが最低2 variantsを要求していて `{"enable": 1}` が弾かれる
- 既存ユーザーのassignmentが永続化されていて、target_ratioを変えても再アサインされない
- クライアントに1時間のキャッシュがある

という構造的な問題があって、100% enableにする運用自体が難しいことがわかりました。なのでフラグごと撤廃して、サブスクの導線は常時表示にしました。

<!-- TODO: App Store Connect の返信フォームで「サブスクに辿り着く手順」をどう返信したか本人が加筆 -->

## Guideline 2.1(b): コインのIAPが審査に提出されていない

> Guideline 2.1(b) - Performance - App Completeness
>
> We are unable to complete the review of the app because one or more of the In-App Purchase products have not been submitted for review.
>
> Specifically, the app includes references to coins but the associated In-App Purchase products have not been submitted for review.

> 1つ以上のアプリ内課金の商品が審査に提出されていないため、アプリの審査を完了できません。
>
> 具体的には、アプリにはコインへの参照が含まれていますが、対応するアプリ内課金の商品が審査に提出されていません。

これはコードの問題ではなくApp Store Connectの操作の話で、バイナリとIAPは別々に審査提出する必要がある、というのを知りませんでした。バイナリだけ提出して、コインの消耗型IAP 4つ（`coin_150` / `coin_500` / `coin_1200` / `coin_3500`）は「審査の準備ができました」のまま放置していた、というやつです。

再提出のときにSubmissionにこの4つを追加して、ステータスが「審査待ち」になったのを確認しました。

各IAPには App Review Screenshot が必須なので、コインショップ画面のスクショをRoborazziで1170×2364pxで生成するテストを足して、4つのIAPで使い回しました。

## Guideline 2.3.10: バイナリにGoogle Playへの参照が残っていた

> Guideline 2.3.10 - Performance - Accurate Metadata
>
> The app or metadata includes information about third-party platforms that may not be relevant for App Store users, who are focused on experiences offered by the app itself.
>
> Revise the app's binary to remove Google Play references.

> アプリまたはメタデータに、App Storeのユーザーには関係のない可能性があるサードパーティプラットフォームの情報が含まれています。
>
> アプリのバイナリを修正して、Google Playへの参照を削除してください。

これはKMPあるあるかもしれません。commonMainの `composeResources` に

```xml
<string name="coinshop_billing_note">※ 購入はApple ID/Google Playアカウントに請求されます</string>
```

という文言を入れていたのと、`core/data` のcommonMainに

```kotlin
object StoreUrl {
    private const val ANDROID_URL = "https://play.google.com/store/apps/details?id=jp.example.app"
    private const val IOS_URL = "https://apps.apple.com/app/idXXXXXXXXXX"
```

のようにPlay StoreのURLを定数で持っていたので、iOSのklibにも文字列リテラルとして埋まっていました。

対応は文字列もURLも `expect/actual` でプラットフォームごとに分けて、commonMainにはGoogle Playの文字を一切置かないようにしました。

```diff kotlin:StoreUrl.kt
-object StoreUrl {
-    private const val ANDROID_URL = "https://play.google.com/store/apps/details?id=jp.example.app"
-    private const val IOS_URL = "https://apps.apple.com/app/idXXXXXXXXXX"
-
+/**
+ * 各プラットフォームのアプリストア（App Store / Google Play）の URL。
+ * App Store Review Guideline 2.3.10 に従い、iOS バイナリには Google Play URL を含めないため、
+ * URL 文字列リテラルは `androidMain` / `iosMain` の actual に分離している。
+ */
+expect object StoreUrl {
     val current: String
-        get() = forPlatform(currentPlatform)
-
-    fun forPlatform(platform: String): String =
-        if (platform == "android") ANDROID_URL else IOS_URL
 }
```

```kotlin:iosMain/StoreUrl.ios.kt
actual object StoreUrl {
    actual val current: String = "https://apps.apple.com/app/idXXXXXXXXXX"
}
```

文字列リソースのほうは、composeResourcesも `androidMain` / `iosMain` に置けるので、そこに分けました。

```kotlin:commonMain/BillingNote.kt
@Composable
expect fun billingNoteText(): String
```

```kotlin:iosMain/BillingNote.ios.kt
@Composable
actual fun billingNoteText(): String =
    stringResource(Res.string.coinshop_billing_note_apple)
```

```xml:iosMain/composeResources/values-ja/strings.xml
<string name="coinshop_billing_note_apple">※ 購入は Apple ID アカウントに請求されます</string>
```

直したあと、iOS用のprepared resource（`.cvr`）と全モジュールのiOS klibを `play.google.com` / `Google Play` でgrepしてヒットゼロになったのを確認しました。

## 再提出

ここまでの修正を入れた build 52 を再提出しました（コインのIAP 4つも一緒に）。

# 2回目のリジェクト（5月7日、build 52）

5月7日に結果が来て、今度は3件。1回目で指摘されたATT・Google Play・IAP未提出は消えていたので、そこは通ったようです。

## Guideline 3.1.2(c): EULAへのリンクがない

> The submission did not include all the required information for apps offering auto-renewable subscriptions.
>
> The following information needs to be included in the App Store metadata:
> - A functional link to the Terms of Use (EULA). If you are using the standard Apple Terms of Use (EULA), include a link to the Terms of Use in the App Description. If you are using a custom EULA, add it in App Store Connect.

> 自動更新サブスクリプションを提供するアプリに必要な情報が、提出内容に含まれていませんでした。
>
> 以下の情報をApp Storeのメタデータに含める必要があります。
> - 利用規約（EULA）への有効なリンク。Apple標準の利用規約（EULA）を使う場合は、App Descriptionに利用規約へのリンクを含めてください。カスタムEULAを使う場合は、App Store Connectに追加してください。

アプリ内のサブスク画面には利用規約とプライバシーポリシーのリンクを置いていたのですが、App Store側のメタデータにはなかった、という指摘です。

自前の利用規約（自分のサイトに置いてます）をカスタムEULAとしてApp Store Connectに登録する方針にしました。

<!-- TODO: 5/8 の再提出時点で App Store Connect 側に何をしたか（標準 EULA のリンクを Description に入れた？カスタム EULA を登録した？）を本人が加筆 -->

で、実際にカスタムEULAとして登録しようとしたら、ダイアログに「カスタムEULAはAppleの最低条件を満たしている必要があります」と出ていることに気づきました。自分の規約は免責事項や知財帰属、解約条件はカバーしていたのですが、Apple Developer Program License AgreementのSchedule 1/2で要求されている

- Appleが契約当事者ではないことの明示
- ライセンスの範囲（デバイス毎、譲渡不可）
- 維持・サポートは運営者の責任
- 製品クレームへの対応責任
- 法令遵守（米国禁輸国でないことの表明）
- Appleが第三受益者であることの明示

が足りていませんでした。なので利用規約に §2 の個人利用限定の条項と、§15 としてApple向けの追加条項を10個足しました。

```diff html:web/src/pages/terms.ts
-  <h2>15. お問い合わせ</h2>
+  <h2>15. Apple App Store からダウンロードされた本アプリへの追加条項</h2>
+  <p>本アプリを Apple App Store からダウンロードしてご利用される場合、本規約に加えて以下の条項が適用されます。本規約と以下の条項に矛盾がある場合、以下の条項が優先します。</p>
+
+  <h3>15-1. 契約の当事者</h3>
+  <p>本規約は、ユーザーと運営者との間の契約であり、Apple Inc.（以下「Apple」）は本契約の当事者ではありません。運営者のみが本アプリおよびそのコンテンツに関する責任を負います。</p>
+
+  <h3>15-2. ライセンスの範囲</h3>
+  <p>運営者は、ユーザーに対し、Apple Media Services Terms and Conditions の Usage Rules に従って、ユーザーが所有または管理する Apple ブランドの製品上で本アプリを使用する、限定的・譲渡不可・非独占的なライセンスを付与します。</p>
+
+  <h3>15-3. 維持・サポート</h3>
+  <h3>15-4. 製品保証</h3>
+  <h3>15-5. 製品クレーム</h3>
+  <h3>15-6. 知的財産権</h3>
+  <h3>15-7. 法令遵守</h3>
+  <h3>15-8. 連絡先</h3>
+  <h3>15-9. 第三者規約への準拠</h3>
+  <h3>15-10. Apple の第三受益者</h3>
+
+  <h2>16. お問い合わせ</h2>
```

（15-3 以降の本文は長いので省略してます）

おまけで、規約バージョンを `2026-05-17.2` に上げたら `users.agreed_terms_version` が `VARCHAR(10)` で `value too long for type character varying(10)` になってゲスト登録が落ちる、というのを踏みました。

## Guideline 2.1(b): IAPの購入でまたエラー

> The In-App Purchase products in the app still exhibited one or more bugs which create a poor user experience. Specifically, an error occurred when we tried to make a purchase.
>
> Review device details:
> - Device type: iPhone 17 Pro Max
> - OS version: iOS 26.4.2

> アプリ内課金の商品に、依然としてユーザー体験を損なうバグが1つ以上ありました。具体的には、購入しようとしたときにエラーが発生しました。

「still」がつらい。

build 52 にはコイン購入の修正は入っていたはずなので、1回目と同じ原因ではなさそうなのですが、この時点ではエラーの内容がわからず原因を特定できませんでした。

<!-- TODO: 2 回目の IAP エラーの原因が結局何だったか（Sandbox で再現したか、Paid Apps Agreement 等）を本人が加筆 -->

なので次に同じことが起きたときにすぐ切り分けられるように、RevenueCatのエラーコードを分類してanalyticsに送るようにしました。

| code | 分類 |
|------|------|
| 5 (ProductNotAvailable), 23 (Configuration) | `PRODUCT_NOT_FOUND` |
| 10 (Network), 35 (OfflineConnection) | `NETWORK_ERROR` |
| その他 | `STORE_ERROR` |
| SDK 例外 | `UNKNOWN` |

これで「Apple側の設定を直すべきか」「端末側のネットワークか」くらいはBigQueryから追えるようになったはず。

## Guideline 3.1.2(c): 1日あたりの価格が月額より目立っていた

> One or more auto-renewable subscriptions are marketed in the purchase flow in a way that may mislead or confuse users about the subscription terms or pricing. Specifically:
> - The auto-renewable subscription displays the daily calculated pricing for the subscription more clearly and conspicuously than the billed monthly amount.
>
> Next Steps
> - Revise the auto-renewable subscription purchase flow to ensure that the billed amount is the most clear and conspicuous pricing element in the layout. Any other pricing elements, including free trial, introductory pricing, and calculated pricing information, must be displayed in a subordinate position and size to the total billed amount.

> 1つ以上の自動更新サブスクリプションが、購入フローにおいてサブスクリプションの条件や価格についてユーザーを誤解させたり混乱させたりする可能性のある方法で訴求されています。具体的には
> - 自動更新サブスクリプションが、月額の請求金額よりも1日あたりの計算価格を明確かつ目立つように表示しています。
>
> 次のステップ
> - 請求金額がレイアウトの中で最も明確で目立つ価格要素になるように、自動更新サブスクリプションの購入フローを修正してください。無料トライアル、導入価格、計算価格を含む他の価格要素はすべて、請求総額より下位の位置とサイズで表示する必要があります。

プランカードの月額価格が `h3` で、申し込みボタンのラベルが「1日たった◯円」の太字だったので、たしかに1日あたりのほうが目立ってました。

月額価格を `h1` のBoldでアクセントカラーにして、1日あたり価格は区切り線の下に小さく `textMuted` で出すようにしました。ボタンのラベルも「ライトプランに申し込む」のようなプラン名に変更してます。

```diff kotlin:PlanCard.kt
                 Text(
                     text = plan.price,
-                    style = NekoTheme.typography.h3,
+                    style = NekoTheme.typography.h1,
                     color = accentColor,
                 )
```

```diff kotlin:PlanCard.kt
+        // 1日あたり価格（補助情報。月額価格より小さく・控えめに表示。
+        // Apple Guideline 3.1.2(c) 準拠で月額より目立たない位置・サイズ・色）
+        if (plan.dailyPrice.isNotEmpty()) {
+            Text(
+                text = stringResource(Res.string.feature_subscription_daily_price_caption, plan.dailyPrice),
+                style = NekoTheme.typography.small,
+                color = NekoTheme.colors.textMuted,
+                textAlign = TextAlign.Center,
+                modifier = Modifier.fillMaxWidth(),
+            )
+            Spacer(modifier = Modifier.height(12.dp))
+        }
```

# その後

2回目の3件を直した build 53 を5月8日に作りました。利用規約の §15 は5月17日に足したので、そっちは build 56 からです。

<!-- TODO: build 53 以降の審査結果（何回目で通ったか、通った日、審査にかかった日数）を本人が加筆 -->

# おわりに

コードのバグはコイン購入の1件だけで、あとはATTの導線、KMPの共通リソースに混ざったGoogle Play、IAPの提出漏れ、EULA、価格の見せ方と、コードの外側の話が多かったです。特にバイナリとIAPを別々に提出するのとカスタムEULAの最低条件は、リジェクトされるまで知らなかったので、初めて出す人は先に見ておくといいかなと思います。

<!-- TODO: 感想と次にやること（Android 版のリリース？サブスク周りの改善？）を本人が加筆 -->

# 参考

- https://developer.apple.com/app-store/review/guidelines/
- https://developer.apple.com/app-store/review/guidelines/#accurate-metadata
- https://developer.apple.com/app-store/user-privacy-and-data-use/
