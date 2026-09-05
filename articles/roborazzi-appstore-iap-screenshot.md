---
title: "App StoreのIAP審査用スクリーンショットをRoborazziで作るには"
emoji: "📸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Roborazzi", "Android", "iOS", "AppStore", "ComposeMultiplatform"]
published: false
---

# はじめに

個人で作っている猫育成ゲーム（KMP + Compose Multiplatform）のiOS版をApp Storeに出したとき、「コインのアプリ内課金が審査に提出されていない」という理由でリジェクトされました。App Store ConnectでIAPを審査に出そうとしたら、各プロダクトに **App Review Screenshot** が必須になっていて、これを1個ずつ用意する必要がありました。

手で撮ってもいいのですが、Compose Multiplatformで画面はAndroidと共通なので、AndroidのRoborazziで撮れば同じ画面が出せるのでは？と思ってやってみました。

# Roborazziの準備

Roborazziはconvention pluginにまとめて、使いたいモジュールで1行つけるだけにしています。

```kotlin:RoborazziConventionPlugin.kt
class RoborazziConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        target.pluginManager.apply("io.github.takahirom.roborazzi")

        target.extensions.configure<LibraryExtension> {
            testOptions.unitTests.isIncludeAndroidResources = true
        }

        target.extensions.configure<RoborazziExtension> {
            outputDir.set(target.file("src/androidUnitTest/screenshots"))
        }
        // 依存の追加は省略
    }
}
```

# サイズをiPhone相当にする

Appleが要求しているIAP審査用スクショの最低サイズは640×920pxです。Robolectricの`qualifiers`で画面サイズと密度を指定できるので、iPhone 14 Pro相当（390×844dp）を3x（xxhdpi）で出すようにしました。390×3 = 1170、844×3 = 2532なので、1170×2532pxになります。

```kotlin:CoinShopAppReviewScreenshotTest.kt
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [33], qualifiers = "w390dp-h844dp-xxhdpi")
class CoinShopAppReviewScreenshotTest {

    @get:Rule
    val composeTestRule = createComposeRule()
```

`@GraphicsMode(NATIVE)`にしないと影やグラデーションが正しく出ないので必須です。

# 中身は本番と同じ値にする

審査で見られるものなので、サンプルデータは本番のマイグレーションに入っている値と揃えました。

```kotlin:CoinShopAppReviewScreenshotTest.kt
// server/migrations/xxxx_create_coin_packages.sql と一致させる
private val productionCoinPackages = listOf(
    CoinPackage(
        id = "pkg-150",
        name = "150コイン",
        description = "特別おやつ1回+おやつ1回分",
        coinAmount = 150,
        storeProductId = "coin_150",
        localizedPrice = "¥160",
    ),
    CoinPackage(
        id = "pkg-1200",
        name = "1,200コイン",
        description = "一番人気！34%お得",
        coinAmount = 1200,
        storeProductId = "coin_1200",
        badge = "人気",
        recommended = true,
        localizedPrice = "¥1,000",
    ),
    // 500 と 3500 も同様
)
```

コインショップの画面は4つのパックが1画面に収まるので、1枚のスクショで4つのIAPすべてに使い回せました。プロダクトごとに撮り分けなくてよかったのは楽でした。

# Androidで撮ると「Google Play」と書いてある問題

ここで1つ引っかかりました。コインショップの下に「※ 購入はApple ID/Google Playアカウントに請求されます」という注意書きを出しているのですが、RoborazziはAndroidで動くので、そのまま撮るとGoogle Play向けの文言になります。

Appleの審査ガイドライン 2.3.10 は、バイナリやメタデータに他プラットフォームへの参照を含めないよう求めているので、審査用スクショにGoogle Playと書いてあるのはまずそうです。（実際、このアプリは別のリジェクトで 2.3.10 を食らっています）

なので、テストからだけ文言を差し替えられるようにしました。

```kotlin:CoinShopAppReviewScreenshotTest.kt
@Test
fun coinShopScreen_appReview_allPackages() {
    composeTestRule.setContent {
        NekoTheme {
            Box(modifier = Modifier.size(390.dp, 844.dp)) {
                CoinShopContent(
                    uiState = CoinShopUiState(
                        packages = productionCoinPackages,
                        userProfile = sampleUserProfile,
                        isLoading = false,
                    ),
                    billingNoteOverride = "* Purchases will be charged to your Apple ID",
                )
            }
        }
    }
    composeTestRule.onRoot().captureRoboImage()
}
```

あとは記録するだけです。

```bash
cd client && ./gradlew :feature:shop:recordRoborazziDebug
```

`src/androidUnitTest/screenshots/`にPNGが出るので、それをApp Store Connectの各IAPの App Review Screenshot 欄にアップロードしました。

# おわりに

iOS向けのスクショなのにAndroidのテストで撮る、というのは最初どうかなと思ったのですが、Compose Multiplatformで画面が共通なので実質同じものが撮れますし、画面を変えたらスクショも撮り直せるのでよかったです。Simulatorを起動して手で撮るより早いです。

ストア掲載用のスクショも同じやり方でいけそうなので、そのうちやってみようかなと思います。

<!-- TODO: この審査用スクショで実際に審査が通ったかどうかを本人が加筆 -->

# 参考

- https://github.com/takahirom/roborazzi
- https://developer.apple.com/help/app-store-connect/manage-consumable-and-non-consumable-in-app-purchases/create-consumable-or-non-consumable-in-app-purchases
- https://developer.apple.com/app-store/review/guidelines/#accurate-metadata
