---
title: "Fleet IDEを使ってKotlin Multiplatformを始めてみた"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin","Fleet"]
published: true
---

# はじめに

Kotlin Multiplatformを試してみたくて最初はIntelij IDEA CEを使おうかと思ってたんですが、Kotlin Multiplatform用？のFleetというIDEがあると知って、Fleetを使ってKotlin Multiplatformを始めてみようかと思います。

主にこちらの記事を読みながらメモしていく感じの記事になるかと思います。

https://www.jetbrains.com/help/kotlin-multiplatform-dev/fleet.html



# Fleetを使ってKotlin Multiplatformを試してみる

わかりやすかったので、以下は↑のページの冒頭を部分を翻訳しただけです。


> この記事では、JetBrains FleetでKotlinマルチプラットフォームプロジェクトを操作するための基本を紹介します。
Fleetは共同開発のために設計されたモダンなIDEです。このFleetはKotlin Multiplatformに優れたエクスペリエンスを提供するよう努めています。最初のステップとして、FleetはKotlin MultiplatformをmacOSにもたらします。
Fleetを使えば、Android、iOS、デスクトッププラットフォームをターゲットとしたマルチプラットフォームプロジェクトを素早く開いて実行することができます。Fleetのスマートモードは、適切なコード処理エンジンを選択します。
iOSをターゲットにしている場合、ナビゲーション、リファクタリング、デバッグはプロジェクトで使用されている言語に関係なく利用できます。これにより、言語が混在するコードベースの管理が容易になります。FleetはSwiftをフルサポートしているので、アプリケーションのネイティブ部分も簡単に拡張・保守できます。


:::message
現在、FleetはKotlin MultiplatformをmacOSでのみサポートしているので、このチュートリアルを完了するにはMacが必要です。
:::


## Fleetのインストール
Toolbox App でのみ公開されてるとのことなので、Toolboxをインストールして、そこからFleetを探してインストールします。
https://www.jetbrains.com/ja-jp/fleet/download/#section=mac

![](https://storage.googleapis.com/zenn-user-upload/809e4ec40d30-20240323.png)

## プロジェクトを新規作成する
プロジェクトを作るには[Kotlin Multiplatform Wizard](https://kmp.jetbrains.com/?_gl=1*fas060*_ga*MTY3MDY5MjgwOS4xNzAzNTE3MDY4*_ga_9J976DJZ68*MTcxMTE0OTMxNS41LjEuMTcxMTE1MDYxNi41OC4wLjA.&_ga=2.142849589.1877468280.1711118271-1670692809.1703517068)というサイトを使うみたいです。
とりあえず全部にチェックしてダウンロードします。

![](https://storage.googleapis.com/zenn-user-upload/087c08f35b5b-20240323.png)

zipを解凍して適当なディレクトリに移動します。


## Fleetを起動してプロジェクトを開く
Fleetを起動します。

ダウンロードしたプロジェクトをフォルダを選択してプロジェクトをOpenします。

![](https://storage.googleapis.com/zenn-user-upload/ad9574a3f344-20240323.png)
（https://www.jetbrains.com/help/kotlin-multiplatform-dev/fleet.html#get-started-with-fleet から画像参考）

Android,iOSをやっていると、build.gradleだったりworkspaceだったりをファイルを選択してプロジェクトを開くので、意外とフォルダを選択することはわからなかったりするのでちょっと強調しておきます。

プロジェクトを開いた状態のFleet画面がこんな感じ

![](https://storage.googleapis.com/zenn-user-upload/df5aefbf6e36-20240323.png)


いきなりこんなエラーが出たが、これはJDK17じゃないからですかね。
```
* What went wrong:
An exception occurred applying plugin request [id: 'com.android.application', version: '8.2.0']
> Failed to apply plugin 'com.android.internal.application'.
   > Could not create an instance of type com.android.build.gradle.internal.dsl.ApplicationExtensionImpl$AgpDecorated.
      > Could not create an instance of type com.android.build.gradle.internal.dsl.LintImpl$AgpDecorated.
         > Could not generate a decorated class for type LintImpl$AgpDecorated.
            > com/android/tools/lint/model/LintModelSeverity has been compiled by a more recent version of the Java Runtime (class file version 61.0), this version of the Java Runtime only recognizes class file versions up to 55.0
```

```
java -version
```
が11とかになってるはずなので、jenvなど使って17にすればよい。ちなみに今の環境だと、

```
❯ jenv versions
* system
  17.0
  17.0.9
  openjdk64-17.0.9
```  
となっていたので、
```
jenv local 17.0.9
```

とした。すると、エラーなくいろいろダウンロードしてくれてるのでしばらく待ちます。

![](https://storage.googleapis.com/zenn-user-upload/1d4eff9d9b68-20240323.png)

10分ちょいかかってようやく完了。

![](https://storage.googleapis.com/zenn-user-upload/5b7dac32e62e-20240323.png)



Smart Modeというのがある。
![](https://storage.googleapis.com/zenn-user-upload/2240d28b0d81-20240323.png)
これを有効にするとFleetはIDEとして使えるようなり、無効にするとシンプルなエディタとして動作するとのこと。

## プロジェクトを実行する

⌘RやRunボタンで、実行するターゲットを選択することができます。

![](https://storage.googleapis.com/zenn-user-upload/d1905c460768-20240323.png)


それぞれ実行すると、次のように実行されます。


![](https://storage.googleapis.com/zenn-user-upload/f38f34c8f4a1-20240323.png)



今回は、iOS17.0だと認識してくれなかったので、iOS17.4をインストールしたら認識してくれました。

![](https://storage.googleapis.com/zenn-user-upload/870538266d0c-20240323.png)



## SwiftからKotlinに値を渡すには

UIはJetpack Composeで書かれており、それは以下のファイルに記述があります。

`composeApp/src/commonMain/kotlin/App.kt`

`App` Composable関数がありますが、App関数は

`composeApp/src/iosMain/kotlin/MainViewController.kt`

で呼ばれています。

![](https://storage.googleapis.com/zenn-user-upload/068f19ec2026-20240323.png)


さらに`MainViewController`関数は `iosApp/iosApp/ContentView.swift`内の `makeUIViewController` 関数で呼ばれています。

![](https://storage.googleapis.com/zenn-user-upload/d4bc6ce01204-20240323.png)

`MainViewController`関数の引数にわたすようにすることができて、ドキュメントでは次のような例が書かれていました。

Swift側で関数を定義してそれを使った結果のStringを`MainViewController`に渡すというものでした。

```swift
diff --git a/iosApp/iosApp/ContentView.swift b/iosApp/iosApp/ContentView.swift
index 3cd5c32..8d1ebac 100644
--- a/iosApp/iosApp/ContentView.swift
+++ b/iosApp/iosApp/ContentView.swift
@@ -4,10 +4,18 @@ import ComposeApp

 struct ComposeView: UIViewControllerRepresentable {
     func makeUIViewController(context: Context) -> UIViewController {
-        MainViewControllerKt.MainViewController()
+        MainViewControllerKt.MainViewController(text: sumArray(input: [10,20,30,40]))
     }

     func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
+
+    private func sumArray(input: [Int]) -> String {
+        var total = 0
+        for item in input {
+            total += item
+        }
+        return "Total value is \(total)"
+    }
 }
 ```

 もちろん、このままだとエラーになるので、`MainViewController`側も修正します。

```kotlin
 diff --git a/composeApp/src/iosMain/kotlin/MainViewController.kt b/composeApp/src/iosMain/kotlin/MainViewController.kt
index fa143d4..8e95e38 100644
--- a/composeApp/src/iosMain/kotlin/MainViewController.kt
+++ b/composeApp/src/iosMain/kotlin/MainViewController.kt
@@ -1,3 +1,3 @@
 import androidx.compose.ui.window.ComposeUIViewController

-fun MainViewController() = ComposeUIViewController { App() }
+fun MainViewController(text: String) = ComposeUIViewController { App(text) }
```

さらにAppにtextを渡しているので、App側も修正します。

```kotlin
diff --git a/composeApp/src/commonMain/kotlin/App.kt b/composeApp/src/commonMain/kotlin/App.kt
index d332107..26138fd 100644
--- a/composeApp/src/commonMain/kotlin/App.kt
+++ b/composeApp/src/commonMain/kotlin/App.kt
@@ -18,15 +18,15 @@ import kotlimmultiplatformsample.composeapp.generated.resources.compose_multipla
 @OptIn(ExperimentalResourceApi::class)
 @Composable
 @Preview
-fun App() {
+fun App(text: String? = null) {
     MaterialTheme {
         var showContent by remember { mutableStateOf(false) }
+        val greeting = remember { Greeting().greet() }
         Column(Modifier.fillMaxWidth(), horizontalAlignment = Alignment.CenterHorizontally) {
             Button(onClick = { showContent = !showContent }) {
-                Text("Click me!")
+                Text(text ?: greeting)
             }
             AnimatedVisibility(showContent) {
-                val greeting = remember { Greeting().greet() }
                 Column(Modifier.fillMaxWidth(), horizontalAlignment = Alignment.CenterHorizontally) {
                     Image(painterResource(Res.drawable.compose_multiplatform), null)
                     Text("Compose: $greeting")
```

これでエラーはなくなったので実行すると、次のようになります。


![](https://storage.googleapis.com/zenn-user-upload/2801319c6c69-20240323.png =400x)

上部のButtonのテキストが、Swiftで作った関数の結果が表示されてることがわかるかと思います。

## リファクタリング

Kotlinの関数がSwiftで使われてるときにFleetのリファクタリング機能を使ってリネームできるとのことなのでやってみた。（これちょっと感動）

https://youtu.be/cwehdiljgU0

Contextメニューからでもできるし、^Rでも可能。

## デバッガー

> The Fleet debugger can automatically step between Swift and Kotlin code.

ほー！

Swift側とKotlin側でBreakPointをつけてDebug実行してみました。

https://youtu.be/VOOColUXAk4

ステップ実行できた！
ただ、Swift側で値の中身が見れたけどKotlinでの中身が見れそうになかった。。今後に期待か！？

# おわりに

Fleetを使いながらKotlin Multiplatformがどんな感じかという触りは触れれたかなと思うので、自分の中では一旦満足です！


## 今回試したプロジェクトはこちら

https://github.com/kwmt/KotlinMultiPlatformSample