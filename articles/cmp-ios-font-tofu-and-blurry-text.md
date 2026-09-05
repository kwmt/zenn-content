---
title: "Compose MultiplatformのiOSで文字が豆腐になったり滲んだりしたときのメモ"
emoji: "🔤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["KotlinMultiplatform", "ComposeMultiplatform", "iOS", "Kotlin"]
published: false
---

# はじめに

個人で作っている猫育成ゲーム（KMP + Compose Multiplatform）で、iOSだけ文字の表示がおかしくなる不具合を2つ踏んだ。

1. 一部の日本語が豆腐（▮）になる
2. 別アプリから復帰すると文字が二重に滲む

どちらもAndroidでは起きなくてiOSだけだったので、原因と直し方をめもっとく。踏んだときのCompose Multiplatformは1.10.0（今は1.11.1に上げているが、そこで再現するかは確認できてない）。

<!-- TODO: 2つのバグにどうやって気づいたか（実機を触ってて？スクショを見て？）を本人が加筆 -->

# 一部の日本語が豆腐（▮）になる

## 症状

Bottom Barのラベルやカテゴリチップで、一部の日本語が▮になる。全部の文字が化けるわけではなくて一部だけ。しかも常時ではなく、起動直後やタブ切り替え直後といった特定のタイミングで出る。

<!-- TODO: 豆腐になった画面のスクショがあれば ![](zenn-user-upload の URL) を置く -->

## 原因

`FontFamily.Default` のフォールバックチェーンがiOSで失敗しているようだった。Androidの`FontFamily.Default`はNoto Sans CJKへ正しくフォールバックするので、同じコードでもAndroidでは起きない。

## 対策

iOSだけシステムフォント（Hiragino Sans）を明示的に指定することにした。`expect/actual`でプラットフォームごとに`FontFamily`を分ける。

```kotlin:commonMain/NekoFontFamily.kt
/**
 * アプリ全体で使用する日本語対応の FontFamily。
 *
 * iOS では Compose Multiplatform の `FontFamily.Default` のフォールバックチェーンが
 * 特定のタイミング（起動直後・タブ切り替え直後など）で失敗し、一部の日本語文字が
 * 豆腐（▮）として描画される不具合があるため、iOS のみシステムフォント（Hiragino Sans）
 * を明示的に指定する。Android では `FontFamily.Default` が正しく日本語へフォールバックする
 * ため、そのまま使用する。
 */
expect val NekoFontFamily: FontFamily
```

Androidはそのまま。

```kotlin:androidMain/NekoFontFamily.android.kt
// Android では FontFamily.Default が Noto Sans CJK へ正しくフォールバックするため、そのまま使用する。
actual val NekoFontFamily: FontFamily = FontFamily.Default
```

iOSはweightごとに`SystemFont`を並べる。

```kotlin:iosMain/NekoFontFamily.ios.kt
// iOS では Compose Multiplatform 1.10.0 の FontFamily.Default の CJK フォールバックが
// タイミング次第で失敗し、特定の日本語文字が豆腐（▮）として描画される不具合がある。
// iOS 標準の日本語フォントである Hiragino Sans を明示的に指定して回避する。
// SystemFont は Skia の legacyMakeTypeface 経由でフォント名解決される（同期ロード、FOUT なし）。
//
// 注: 現状アプリで Italic を使用していないため Normal style のみ登録している。将来 Italic を
// 使う場合は FontStyle.Italic 分も追加すること（追加しない場合 iOS だけ silent fallback する）。
@OptIn(ExperimentalTextApi::class)
actual val NekoFontFamily: FontFamily = FontFamily(
    SystemFont(identity = "Hiragino Sans", weight = FontWeight.Normal, style = FontStyle.Normal),
    SystemFont(identity = "Hiragino Sans", weight = FontWeight.Medium, style = FontStyle.Normal),
    SystemFont(identity = "Hiragino Sans", weight = FontWeight.SemiBold, style = FontStyle.Normal),
    SystemFont(identity = "Hiragino Sans", weight = FontWeight.Bold, style = FontStyle.Normal),
)
```

`SystemFont`は`ExperimentalTextApi`なので`@OptIn`が要る。

あとはこれを`TextStyle`に効かせないと意味がないので、自前のTypographyの全`TextStyle`に`fontFamily`を足した。

```diff kotlin:NekoTypography.kt
 data class NekoTypography(
     val scoreDisplay: TextStyle = TextStyle(
+        fontFamily = NekoFontFamily,
         fontSize = 56.sp,
         fontWeight = FontWeight.SemiBold,
     ),
     val h1: TextStyle = TextStyle(
+        fontFamily = NekoFontFamily,
         fontSize = 32.sp,
         fontWeight = FontWeight.Bold,
     ),
```

これが20個以上あるので全部に足している。1つでも漏れるとそのスタイルだけ豆腐になるはず。

`SystemFont`で登録していないweightやstyleは、iOSだけ黙ってフォールバックする（エラーにならない）ので、Italicを使い始めるときは追加を忘れないようにコメントを残した。

# 別アプリから復帰すると文字が滲む

## 症状

ゲームのリザルト画面で、AdMobの広告を見てアプリがバックグラウンドに行って復帰すると、文字が二重に滲む。

<!-- TODO: 滲んだ画面のスクショがあれば ![](zenn-user-upload の URL) を置く -->

## 原因

リザルト画面の最外の`Column`を`graphicsLayer { alpha = animAlpha.value }` で常時ラップしていた。これがあるとiOS（Skiko）ではオフスクリーンレンダリングレイヤーが保持され続けるので、一度バックグラウンドに行って復帰したときにキャッシュ済みのbitmapが破損して、文字が二重に滲む、ということらしい。

そもそも`graphicsLayer`は画面表示時の400msのフェードインのために付けていたもので、フェードインが終わったあとも付けっぱなしにしておく必要はなかった。

## 対策

フェードインが終わったら`Modifier`から外して、通常のオンスクリーン描画に戻すようにした。

```diff kotlin:ResultScreen.kt
     Column(
         modifier = Modifier
             .fillMaxSize()
-            .graphicsLayer { alpha = animAlpha.value }
+            // フェードイン中だけ graphicsLayer を適用する。
+            // 常時付与したままにすると iOS (Skiko) でオフスクリーンレイヤーが残り、
+            // 広告視聴などでアプリがバックグラウンドから復帰したときに
+            // キャッシュされた bitmap が破損して文字が二重に滲む不具合が発生する。
+            .then(
+                if (fadeInComplete) {
+                    Modifier
+                } else {
+                    Modifier.graphicsLayer { alpha = animAlpha.value }
+                },
+            )
             .background(NekoTheme.colors.background.copy(alpha = 0.7f))
```

`fadeInComplete`はアニメーション完了時に立てるフラグ。

```kotlin:ResultScreen.kt
val animAlpha = remember { Animatable(0f) }
var fadeInComplete by remember { mutableStateOf(false) }
// ...
animAlpha.animateTo(1f, animationSpec = tween(durationMillis = 400))
fadeInComplete = true
```

# おわりに

2つとも、共通コードは何も間違ってないのにiOSだけ表示が壊れるやつでした。`FontFamily.Default`と`graphicsLayer`という、Androidだと何も考えずに使っているものが原因なので、Androidだけ見てると気づけないなと思います。CMPで両OS出すなら、iOSの実機を触る時間はちゃんと取ったほうがよさそうです。

# 参考

- https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-multiplatform-resources.html
