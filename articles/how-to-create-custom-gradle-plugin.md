---
title: "Custom Gradle Pluginの作り方"
emoji: "🐘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android","gradle"]
published: false
---

# はじめに

新しくAndroid Libraryモジュールを作ると、build.gradle(.kts)の設定が重複してしまう（たとえば`minSdk`や`compileSdk`などのバージョン設定やbuildTypeなど）ので、それらをカスタムGralde Pluginを作って共通化するためにはどうしたらいいかのを記録したものです。

## 前提
以下の説明では、Android Gradle Pluginのバージョン `8.3.0-beta01`を前提としています。


## Pluginをどこに置くか？

以下に記載されていますが、Pluginのソースをいくつかの場所に置くことができます。
https://docs.gradle.org/current/userguide/custom_plugins.html#sec:packaging_a_plugin
以下はPackagin a pluginを翻訳・意訳したものです。

- Build Script
  - 各モジュールのbuild.gradle.kts内にPluginを実装することができます。何もしなくてもプラグインが自動的にコンパイルされるメリットがあるが、定義したbuild script以外のbuild scrpitでは再利用はできません。
- `buildSrc` project
  - rootProjectDir/buildSrc配下にPluginのソースを置くことができます。このPluginはビルドで使用されるすべてのビルドスクリプトから見えるようになりますが、このProjectの外でPluginを再利用することはできません。
- Standalone project
  - プラグイン用に別のプロジェクトを作成することができます。このプロジェクトはJARを生成して公開し、複数のビルドで使用したり、他の人と共有したりすることができます。一般的に、このJARはいくつかのプラグインを含むか、関連するいくつかのタスククラスを1つのライブラリにバンドルします。あるいは、その2つの組み合わせです。

このなかで、nowinandroidでも採用されているStandalone projectでのPluginの作り方を紹介します。


# Custom Gradle Pluginの作り方

## simple gradle plugin
その前にシンプルなPluginの書き方を見ておきたいと思います。

Gradle Pluginを作るには、`Plugin`インターフェースを実装したクラスを作る必要があります。それをprojectにapplyすれば使えるようになります。

```kotlin:build.gradle.kts
class GreetingPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        project.task("hello") {
            doLast {
                println("Hello from the GreetingPlugin")
            }
        }
    }
}
// Apply the plugin
apply<GreetingPlugin>()
```
この例だと、gradle taskに`hello`が追加されたので`hello`タスクを実行できるようになります。
```
❯ ./gradlew hello
> Task :app:hello
Hello from the GreetingPlugin
```


## plugin用のモジュールを作成
シンプルなPluginの作り方がわかったところで、Standalone projectにPluginを置く方法を見ていきたいと思います。

まだ作成してなければまずはplugin用のディレクトリ(ここでは`build-logic`という名前にした)を作成し、その中に以下の内容で`setting.gradle.kts`ファイルを作成します。


```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = "build-logic"
```

※libs.versions.tomlが存在しているとする。


plugin用のディレクトリに、kotlin libraryモジュールを作成します。
- Module name: convention (何でも良いですが、今回は[nowinandroid](https://github.com/android/nowinandroid/tree/main/build-logic/convention)を参考)
- 使用Templates: Java & Kotlin Library

モジュールの作り方はこちらを参照ください。

https://zenn.dev/yasi/articles/how-to-create-new-library-module

作成したモジュールのbuild.gradle.ktsに以下を記入して、Sync Gradleします。

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
   `kotlin-dsl`
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
tasks.withType<KotlinCompile>().configureEach {
    kotlinOptions {
        jvmTarget = JavaVersion.VERSION_17.toString()
    }
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
}

gradlePlugin {
    plugins {
        // ここにこれから作るCustome Gradle Pluginを登録していきます。
    }
}
```
下図のようになっていたら成功です。

![](https://storage.googleapis.com/zenn-user-upload/449fabaf42d6-20240106.png)


## Custom Plugin実装する

appモジュール(applicationモジュール)とAndroid Libraryモジュールのそれぞれのbuild.gradle(.kts)にある、minSdkなどの設定を共通化するPluginを実装してみようと思います。
ただ、appモジュール用のPluginとlibraryモジュール用のPluginはそれぞれ作成する必要があるので、それぞれのPluginの実装を共通化するという方針になります。

- applicationモジュール用のPlugin

```kotlin
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<ApplicationExtension> {
                // TODO
            }
        }
    }
}
```

- libraryモジュール用のPlugin

```kotlin
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
            }

            extensions.configure<LibraryExtension> {
                // TODO
            }
        }
    }
}
```

と、それぞれのpluginを作ってこれだけだと各モジュールで使えないので使えるようにするには、 `build-logic/convention/build.gradle.kts`の `gradlePlugin.plugins`に登録する必要があります。

たとえばこんな感じです。
```kotlin
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "org.example.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidLibrary") {
            id = "org.example.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
    }
}
```

これで各モジュールで使えるようになったので、appモジュール(applicationモジュール)では

```kotlin:app/build.gradle.kts
plugins {
    id("org.example.application")
}
```
libraryモジュールでは

```kotlin:library/build.gradle.kts
plugins {
    id("org.example.library")
}
```
と書けば使えるようになります。
今後新しくライブラリモジュールを追加する際に、同様に書くだけで同じものが適用されるので、build.gradle(.kts)ファイルがとてもスッキリします。


## 設定を共通化する部分の実装

minSdkやcompileSdkなどの設定を次のような感じで実装し、

```kotlin
internal fun Project.configureKotlinAndroid(
    commonExtension: CommonExtension<*, *, *, *, *, *>,
) {
    commonExtension.apply {
        compileSdk = 34

        defaultConfig {
            minSdk = 23
            testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        }

        compileOptions {
            sourceCompatibility = JavaVersion.VERSION_17
            targetCompatibility = JavaVersion.VERSION_17
        }

        buildTypes {
            getByName("release") {
                isMinifyEnabled = true
                proguardFiles(
                    getDefaultProguardFile("proguard-android-optimize.txt"),
                    "proguard-rules.pro"
                )
            }
        }
    }
    tasks.withType<KotlinCompile>().configureEach {
        kotlinOptions {
            jvmTarget = JavaVersion.VERSION_17.toString()
        }
    }
}
```

それぞれのPluginで呼び出して完成です。

- applicationモジュール用のPlugin

```kotlin
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<ApplicationExtension> {
                configureKotlinAndroid(this)
                defaultConfig {
                    targetSdk = 34
                }
            }
        }
    }
}
```

- libraryモジュール用のPlugin

```kotlin
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
            }

            extensions.configure<LibraryExtension> {
                configureKotlinAndroid(this)
            }
        }
    }
}
```

# おわりに

Pluginはこれまで見てきたようにminSdkなどの設定をPlugin化できたり、GreetingPluginで見たようにタスクを登録することもできたり、いろいろPlugin化できてbuild.gradle(.kts)がスッキリしていくので、モジュール共通化のためだけではなく、Fatになったbuild.gradle(.kts)をPlugin化してみてはいかがでしょうか。


# 参考
- https://docs.gradle.org/current/userguide/custom_plugins.html