---
title: "detektのカスタムルールでモジュール間の依存を禁止するには"
emoji: "🚧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kotlin", "detekt", "gradle", "KotlinMultiplatform", "Android"]
published: false
---

# はじめに

個人で作っている猫育成ゲームのクライアントはKMPのマルチモジュール構成で、こんな感じに分けています。

```
core/
  domain/        # ドメインモデル・リポジトリのインターフェース（純粋Kotlin）
  data/          # Room, Ktor, Repository実装
  designsystem/  # テーマと共通コンポーネント
feature/
  home/ shop/ care/ friend/ ...
```

`feature`は`core:domain`のインターフェースだけを見て、`core:data`の実装は見ない、というルールにしています。ただ、こういうのは決めても忘れます。1人で作っていても、しばらく経つと普通に`import`してしまう。

なので、detektのカスタムルールを書いてビルドで落とすようにしました。書き方をめもっとく。

# ルールを書く

detektのカスタムルールは`Rule`を継承して、見たいPSIのvisitメソッドをoverrideするだけです。今回は`import`文を見たいので`visitImportDirective`です。

```kotlin:NoDataLayerDependencyInFeatureRule.kt
class NoDataLayerDependencyInFeatureRule(config: Config = Config.empty) : Rule(config) {

    override val issue = Issue(
        id = "NoDataLayerDependencyInFeature",
        severity = Severity.Defect,
        description = "Feature モジュールは core:data に依存できません。core:domain のインターフェースを使用してください。",
        debt = Debt.TWENTY_MINS,
    )

    override fun visitKtFile(file: KtFile) {
        val packageName = file.packageFqName.asString()
        if (!packageName.startsWith("jp.example.app.feature")) return

        super.visitKtFile(file)
    }

    override fun visitImportDirective(importDirective: KtImportDirective) {
        val importPath = importDirective.importPath?.pathStr ?: return

        if (importPath.startsWith("jp.example.app.core.data")) {
            report(
                CodeSmell(
                    issue = issue,
                    entity = Entity.from(importDirective),
                    message = "Feature モジュールは core:data に依存できません。" +
                        "core:domain のインターフェースを使用してください。" +
                        " (import: $importPath)",
                ),
            )
        }
    }
}
```

ポイントは`visitKtFile`のところで、パッケージ名が`feature`で始まらないファイルは`super`を呼ばずに抜けています。こうすると配下の`visitImportDirective`も呼ばれないので、対象外のモジュールを丸ごとスキップできます。

`message`には「代わりに何を使えばいいか」を書いておくと、あとから自分で見たときに助かります。

# ルールセットに登録する

`RuleSetProvider`を作って、ルールを詰めます。

```kotlin:ArchitectureRuleSetProvider.kt
class ArchitectureRuleSetProvider : RuleSetProvider {
    override val ruleSetId: String = "architecture"

    override fun instance(config: Config): RuleSet = RuleSet(
        ruleSetId,
        listOf(NoDataLayerDependencyInFeatureRule(config)),
    )
}
```

これをServiceLoaderから見えるようにするために、`resources/META-INF/services/`にファイルを置きます。ファイル名がインターフェース名、中身が実装クラスのFQCNです。

```:src/main/resources/META-INF/services/io.gitlab.arturbosch.detekt.api.RuleSetProvider
jp.example.app.detekt.ArchitectureRuleSetProvider
```

ここを忘れると、ルールのクラスは存在するのに何も起きない、という状態になります。`RuleSetProvider`の`listOf`に足し忘れても同じです。

:::message
ルール自体のユニットテスト（`lint()`して`findings`の数を見るやつ）は書けるので、ルールのロジックはテストで担保できます。ただ「そのルールが実際にプロジェクトに適用されているか」はテストされないので、ここは別で確認する必要があります。
:::

# 設定ファイル

detektのデフォルトルールは全部offにして、自分のルールセットだけonにしています。

```yaml:config/detekt/detekt.yml
# Disable all default rule sets - only use custom architecture rules
complexity:
  active: false

coroutines:
  active: false

# ...

# Custom architecture rules
architecture:
  active: true
  NoDataLayerDependencyInFeature:
    active: true
```

デフォルトのルールも有効にすると既存コードの指摘が大量に出て、結局`@Suppress`だらけになりそうだったので、「アーキテクチャの約束事だけ機械に見てもらう」という割り切りにしました。

# convention pluginから配る

新しいモジュールを作るたびにdetektの設定を書くのは面倒なので、convention pluginに寄せています。

```kotlin:ComposeMultiplatformConventionPlugin.kt
class ComposeMultiplatformConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        target.pluginManager.apply("example.kmp.library")
        target.pluginManager.apply("org.jetbrains.compose")
        target.pluginManager.apply("org.jetbrains.kotlin.plugin.compose")
        target.pluginManager.apply("io.gitlab.arturbosch.detekt")

        // ... compose の依存追加は省略

        target.extensions.configure(DetektExtension::class.java) {
            config.setFrom(target.rootProject.files("config/detekt/detekt.yml"))
            buildUponDefaultConfig = true
        }

        target.dependencies.add(
            "detektPlugins",
            target.project(":detekt-rules"),
        )
    }
}
```

これで、新しいfeatureモジュールを作るときにconvention pluginを1行つけるだけで、Composeの依存もdetektのカスタムルールも一緒についてきます。あとは

```bash
cd client && ./gradlew detekt
```

で全モジュールをチェックできます。

# おわりに

「featureからdataを触らない」みたいなルールは、レビューで指摘するより機械に見てもらうほうが早いなと思いました。ルール自体は`import`の文字列を見ているだけなので、思ったより簡単に書けます。

<!-- TODO: 実際にこのルールに引っかかったことがあるか（自分でやらかしたか）を本人が加筆 -->

<!-- TODO: 文字列リソースのキー名を見るルール（StringResourceKeyNamingRule）は実装もユニットテストもあるが、
     ArchitectureRuleSetProvider の listOf と detekt.yml のどちらにも登録されていないので現状 1 度も動いていない。
     いま登録すると既存キー 1084 件のうち 122 件（catprofile_*, feedback_* など）が違反になって detekt が落ちるので、
     有効化するなら先にキーのリネームが要る。この記事では
     「ルールを書いても登録を忘れると動かない」実例として書き足すか、触れないか決める -->

# 参考

- https://detekt.dev/docs/introduction/extensions/
- https://detekt.dev/docs/introduction/configurations/
