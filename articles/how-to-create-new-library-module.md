---
title: "Android Studioでライブラリモジュールの作り方"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android","gradle"]
published: true
---

最近よくモジュールを作るのでメモしておく。

# 新しくライブラリモジュールを作る
Android StudioのProjectツリーでproject rootを選択し、⌘+Nを押します。（その状態で`m`をタイプすると検索できます）

![](https://storage.googleapis.com/zenn-user-upload/7f2b527fd12b-20240106.png)


`Module`を選択すると、`Create New Module`ダイアログが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/b5ccdf559c6e-20240106.png)

Templates[^1]で`Android Library`を選択し、右側で必要事項を入力します。

- Module name: 作りたい適切なモジュール名
- Package name: パッケージ名(自分は applicationId + module name とすることが多いです)
- その他はデフォルトで良いかと思います。

入力を終えてたら、Finishをクリックします。すると新しく作ったモジュールが作られます。


![](https://storage.googleapis.com/zenn-user-upload/4d5e5ec18b89-20240106.png)

あとこのとき変更されるものとしては、`settings.gradle.kts`が次のように変更されます。

```diff kotlin
diff --git a/settings.gradle.kts b/settings.gradle.kts
index 0b6e301..a0b7ddd 100644
--- a/settings.gradle.kts
+++ b/settings.gradle.kts
@@ -21,3 +21,4 @@ 
 include(":app")
+include(":sample")
```

この状態で Sync Gradleすると新しく作ったモジュールが使える状態になりました。


[^1]: Templatesで私がよく使うテンプレートを紹介
    - `Android Library`
      - appモジュール(applicationモジュール)から使いたいモジュールを作りたいとき利用
    - `Phone & Tablet`
      - 同じプロジェクト内に別のアプリを作りたい場合に利用
      - たとえば、nowinandroidだと通常アプリとは別に[カタログアプリ](https://github.com/android/nowinandroid/tree/main/app-nia-catalog)で使われています。
    - `Benchmark`
      - [ベンチマーク](https://developer.android.com/topic/performance/benchmarking/benchmarking-overview?hl=ja)を行うときに利用


# 作ったモジュールを使う

たとえばappモジュールから先程作ったsampleモジュールを参照できるようにするには、appモジュールのbuild.gradle(.kts)を開き、dependenciesで
`implementation(project(":sample"))`を追加します。

```diff kotlin
--- a/app/build.gradle
+++ b/app/build.gradle
@@ -22,6 +22,7 @@ android {
 dependencies {
+    implementation(project(":sample"))
```

これはstringで指定する方法ですが、Gralde7.0以降であれば、タイプセーフにかくことができます。
そのためにはまず、settings.gradle(.kts)に
```kotlin
enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")
```
を追加します[^2]。
appモジュールのbuild.gradle(.kts)に戻り、`projects.<module name>`と書き直すことができます。

```diff kotlin
 dependencies {
+    // implementation(project(":sample"))
+    implementation(projects.sample)
```

これでappモジュールからsampleモジュールを参照できるようになったので、確認のためsampleモジュールに適当なクラスを作成し、

![](https://storage.googleapis.com/zenn-user-upload/96ac8c3bf4de-20240106.png)

appモジュールで使えることが確認できました。(以下の図はappモジュールにあるMainActivity内)

![](https://storage.googleapis.com/zenn-user-upload/d84bf57a6a75-20240106.png)


[^2]: https://docs.gradle.org/7.0/release-notes.html#new-features-and-usability-improvements

# モジュールの削除方法

モジュールを削除するのも意外と困るのでメモしておきます。

単純に削除したいモジュールを右クリックしてDeleteを探すだけだと無いですし、⌘+Deleteで消すことができません。

![](https://storage.googleapis.com/zenn-user-upload/28e3cf4a63ae-20240107.png)

Deleteできるようにするには、`settings.gradle(.kts)`に書かれてる `include(":sample")` のような消したいモジュールのincludeを削除して、Sync Gradleします。完了したらもう一度消したいモジュールを右クリックしてみると、`Delete`が存在するので、クリックすると削除できます。（⌘+Deleteでも削除可能になります）


![](https://storage.googleapis.com/zenn-user-upload/3119f14d9b6d-20240107.png)
