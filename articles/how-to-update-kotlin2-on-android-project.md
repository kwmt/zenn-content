---
title: "Android ProjectでKotlin2.0にアップデートするには"
emoji: "💨"
type: "tech"
topics: ["Android","Kotlin"]
published: true
published_at: 2024-06-19 08:00
---

# はじめに

個人のAndroid Projectでkotlin2.0に上げたときにやった作業のメモです。
前提として、上げる前の環境は
- kotlin 1.9.24
- AGP 8.5
- compose compier 1.5.14
- ksp導入済み

あたりでしょうか。
比較的新しいバージョンだと思うので、これらのバージョンではない場合は、この記事以外にも作業が必要かもしれません。


# kotlin 2.0にアップデートする


## kotlinを2.0.0にする

```diff toml
--- a/gradle/libs.versions.toml
+++ b/gradle/libs.versions.toml
@@ -2,7 +2,7 @@
 barcode-scanning = "17.2.0"
 camera-camera2 = "1.3.4"
 camera-view = "1.3.4"
-kotlin = "1.9.24"
+kotlin = "2.0.0"
 ksp = "1.9.24-1.0.20"
 coroutine = "1.8.0"
 appcompat = "1.7.0"
```

## compose pluginの導入
### `org.jetbrains.kotlin.plugin.compose` を top levelの build.gradle.ktsに追加する。


```diff toml
--- a/gradle/libs.versions.toml
+++ b/gradle/libs.versions.toml
@@ -116,6 +116,7 @@ junit5-android-test-runner = { module = "de.mannodermaus.junit5:android-test-run
 [plugins]
 android-application = { id = "com.android.application", version.ref = "agp" }
 kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
+compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
 google-services = { id = "com.google.gms.google-services", version.ref = "google-services" }
 firebase-crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "firebase" }
 ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

```diff toml
--- a/build.gradle.kts
+++ b/build.gradle.kts
@@ -23,6 +23,7 @@ buildscript {
 plugins {
     alias(libs.plugins.android.application) apply false
     alias(libs.plugins.kotlin.android) apply false
+    alias(libs.plugins.compose.compiler) apply false
     alias(libs.plugins.google.services) apply false
     alias(libs.plugins.firebase.crashlytics) apply false
     alias(libs.plugins.parcelize) apply false
```
### Composeを使うモジュールのbuild.gradle.ktsにpluginを適用する

```diff toml
+++ b/core/presentation/build.gradle.kts
@@ -1,6 +1,7 @@
 plugins {
     id("com.android.library)
     id("org.jetbrains.kotlin.android")
+    alias(libs.plugins.compose.compiler)
     id("androidx.navigation.safeargs")
     alias(libs.plugins.ksp)
     id("kotlin-parcelize")
```


### kotlinCompilerExtensionVersion は不要なので削除

```diff kotlin
ConfigureAndroidCompose.kt
@@ -11,10 +11,5 @@ internal fun CommonExtension<*, *, *, *, *, *>.configureAndroidCompose(project:
         buildFeatures {
             compose = true
         }
-
-        composeOptions {
-            kotlinCompilerExtensionVersion =
-                project.libs.findVersion("compose-compiler").get().toString()
-        }
     }
 }
```

### compose-compiler バージョンも不要になったので削除

```diff toml
 libs.versions.toml
  compose = "1.6.8"
-compose-compiler = "1.5.14"
 lifecycle = "2.8.2"
 navigation = "2.7.7"
```

## kspのバージョンを上げる

> ksp-1.9.24-1.0.20 is too old for kotlin-2.0.0. Please upgrade ksp or downgrade kotlin-gradle-plugin to 1.9.24.

と出るので、kspのバージョンを上げる

```diff toml
+++ b/gradle/libs.versions.toml
@@ -3,7 +3,7 
 kotlin = "2.0.0"
-ksp = "1.9.24-1.0.20"
+ksp = "2.0.0-1.0.22"
 coroutine = "1.8.0"
```

## (必要なら)composeCompiler オプションを設定

compose compiler pluginを適用したモジュールの build.gradle.ktsで、composeCompilerオプションを設定できます。


```diff kotlin
 android { }
+composeCompiler {
+    enableStrongSkippingMode = true
+
+    reportsDestination = layout.buildDirectory.dir("compose_compiler")
+}
```


composeCompilerオプションの設定できるプロパティ一覧はこちら　

https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-compiler.html#compose-compiler-options-dsl

### convention pluginでの書き方

nowinandroidが参考になるかもです。

https://github.com/android/nowinandroid/blob/85129e4660f7a27c7081f4ac21169d19db89fbb6/build-logic/convention/src/main/kotlin/com/google/samples/apps/nowinandroid/AndroidCompose.kt#L54-L71

# 参考

https://developer.android.com/develop/ui/compose/compiler?hl=ja
