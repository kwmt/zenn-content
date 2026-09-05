---
title: "KMPアプリにiOSウィジェットとApple Watchアプリを足すときのデータの渡し方"
emoji: "⌚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["KotlinMultiplatform", "iOS", "Swift", "WidgetKit", "watchOS"]
published: false
---

# はじめに

個人で作っている猫育成ゲーム（KMP + Compose Multiplatform）に、ホーム画面のウィジェットとApple Watchアプリを足しました。猫のおなかや機嫌の状態が、アプリを開かなくても見られるようにしたかったからです。

やってみると、同じ「猫の状態を表示する」だけなのに、Android・iOSウィジェット・Watchで全部やり方が違って、データをどう届けるかで悩んだのでめもっとく。

<!-- TODO: ウィジェットとWatchを作ろうと思ったきっかけを本人が加筆 -->

# Kotlinのコードが届く範囲が場所によって違う

まずここが出発点でした。

| 実行される場所 | KMPのコードとローカルDB |
| --- | --- |
| アプリ本体（Android / iOS） | 使える |
| Androidのウィジェット | 使える（更新のたびにアプリプロセスが起動する） |
| iOSのウィジェット拡張 | 使えない（別プロセス。アプリのDBを開けない） |
| Apple Watchアプリ | 使えない（別デバイス） |

Androidのウィジェットは、更新のたびにアプリのプロセスが起きてくるので、普通にRepositoryを注入して使えます。

```kotlin:CatStatusWidgetDataProvider.kt
class CatStatusWidgetDataProvider(
    private val catRepository: CatRepository,
    private val selectedCatRepository: SelectedCatRepository,
    private val statusCatRepository: StatusCatRepository,
    private val careRepository: CareRepository,
    private val nowMillis: () -> Long = { Clock.System.now().toEpochMilliseconds() },
) {
    suspend fun getData(): CatStatusWidgetData? {
        val cats = catRepository.getAll()
        // ...
    }
}
```

アプリ本体と同じコードで同じ値が出るので、ここは特に困りませんでした。

iOSとWatchはそうはいかないので、「アプリ本体が動いているうちにスナップショットを書き出しておく」方式にしました。

# iOSウィジェット: App GroupのUserDefaultsに置く

ウィジェット拡張とアプリでデータを共有する方法はいくつかありますが、量が少ないのでApp GroupのUserDefaultsにJSONを1本置くだけにしました。

書き出す側はKotlin（`iosMain`）です。`NSUserDefaults(suiteName:)`はKotlin/Nativeからそのまま呼べます。

```kotlin:WidgetDataBridge.kt
private const val APP_GROUP_ID = "group.jp.example.app"
private const val WIDGET_DATA_KEY = "widget_cat_data"

val userDefaults = NSUserDefaults(suiteName = APP_GROUP_ID)
if (userDefaults != null) {
    userDefaults.setObject(json, forKey = WIDGET_DATA_KEY)
} else {
    Napier.e("[Widget] NSUserDefaults(suiteName: $APP_GROUP_ID) が null。App Group 設定を確認してください")
}
```

`suiteName`が間違っていたりApp Groupの設定が漏れていたりすると、例外ではなく`null`が返ってくるだけなので、ログを出しておかないと気づけません。ここは最初ハマりました。

書いたあとは`WidgetCenter.shared.reloadAllTimelines()`を呼んでウィジェットに再読み込みさせます。

猫の状態はお世話するたびに変わるので、Flowをそのまま購読して書くと書きすぎになります。`debounce`を入れました。

```kotlin:WidgetDataBridge.kt
.debounce(1000L)
```

# スナップショットには「生値」を入れる

ここが一番ハマったところなのですが、スナップショットに **表示用に計算済みの値** を入れると、ウィジェット側とアプリ本体で表示がずれます。

このアプリは猫のステータスが時間で減っていく作りで、DBには「最終更新時点の生値 + その時刻」を持っていて、表示のときに経過時間ぶんを引く、という設計になっています。ウィジェットはアプリが起動していない間もタイムラインを進めるので、減衰の計算材料のほうを渡して、ウィジェット側で計算させる必要がありました。

```kotlin:WidgetDataBridge.kt
@Serializable
data class WidgetCatData(
    val catName: String,
    val conditionLabel: String,
    val hungerPercent: Float,
    // ...
    /**
     * 上記ステータス値の基準時刻（エポック秒）。
     * ウィジェット拡張はアプリが起動していない間もタイムラインを更新するため、
     * この時刻からの経過分をウィジェット側で追加減衰させる。null なら追加減衰しない。
     */
    val statBaseEpochSeconds: Long? = null,
    /**
     * 各ステータスが 1 ポイント減るまでの分数（**実効値**）。0 以下なら減衰しない。
     * オフライン倍率と復帰ボーナス半減を反映済みなので、Swift 側はこの分数で割るだけでよい。
     */
    // ...
)
```

この「同じ計算をKotlinとSwiftの両方に持つ」ところは、総当たりのテストを書いたら3つバグが出たので、そっちは別記事に書きました。

<!-- TODO: kmp-swift-parity-test を公開したら、ここに https://zenn.dev/yasi/articles/kmp-swift-parity-test を1行で置く -->

もうひとつ、スナップショットのフィールドを増やすときは必ずデフォルト値を付けます。ウィジェット拡張やWatchアプリのバージョンがアプリ本体より古いことがあって、古いフォーマットのJSONが残っていてもデコードで落ちないようにするためです。

# Apple Watch: 別デバイスなので認証がいる

Watchはアプリ本体からペイロードを送るのに加えて、Watch自身がAPIを叩きに行きます（お世話をWatchからできるようにしたので）。つまりWatch側もアクセストークンとリフレッシュトークンを持つことになります。

ここで1つバグを作りました。**リフレッシュトークンが失効するとWatchが詰む**というものです。

- `WatchSessionService.isAuthenticated`は`true`にする箇所しかなく、`false`に戻る経路がなかった
- リフレッシュ失敗のエラーは投げられるだけで、誰も捕まえて再認証につないでいなかった
- 再ペアリングを促す画面は`isAuthenticated == false`のときに出るので、初回起動時にしか出ない

結果、Watchは「認証済み」の顔をしたまま古いデータを表示し続けて、全部のリクエストが黙って失敗する。ユーザーからは何が起きているか分からないし、アプリを消すまで復帰できない、という状態でした。

しかもテストは`testRefreshToken_failsClearsAuth`という名前で存在していて、通っていました。中身が名前どおりのことを検証していなかった、というオチです。

直し方はシンプルで、リフレッシュトークンが拒否されたら認証情報を捨てて未認証に戻します。

```swift:WatchAPIClient.swift
/// 認証情報を破棄する。
///
/// `baseUrl` はサーバ設定であって認証情報ではないので残す。
/// トークンを空にすることで [isConfigured] が false になり、UI 側は再ペアリングを促す状態に戻せる。
private func clearCredentials() {
    accessToken = ""
    refreshToken = ""
}
```

ポイントは、**破棄するのは401 / 403のときだけ**にしたことです。

```swift:WatchAPIClient.swift
/// リフレッシュトークン自体が拒否された（401 / 403）場合は保持している認証情報を破棄する。
/// 破棄しないと [isConfigured] が true のまま失効したトークンでリクエストを投げ続け、
/// Watch が「認証済みなのに何も動かない」状態から復帰できなくなる。
///
/// 5xx やレスポンス不正など**一時的な失敗では破棄しない**。
/// サーバ障害のたびにユーザーを締め出してしまうため。
```

もともとリフレッシュの失敗を一律で同じエラーにしていたので、無条件に破棄すると、サーバーが一時的に落ちただけで全ユーザーのWatchが再ペアリング待ちになってしまいます。復帰の流れはこうなりました。

```
リフレッシュが401 → 認証情報を破棄 → isConfigured=false
  → isAuthenticated=false → 再ペアリング画面を表示
  → ユーザーがiPhoneアプリを開く → WCSessionで新トークンが届く → 自動復帰
```

テストのほうも、名前どおりトークンが空になることを検証する内容に直して、500のときは認証情報を保持することを確認するテストを足しました。

# おわりに

ウィジェットとWatchは「アプリの一部」のつもりで作り始めたのですが、実際は別プロセス・別デバイスで、データも認証も自分で持つ小さいアプリでした。アプリ本体が持っている前提（DBが読める、ログイン済み）が全部なくなるので、そのつもりで設計しないとハマるなと思います。

Androidのウィジェットだけやたら楽だったのが印象的でした。

# 参考

- https://developer.apple.com/documentation/widgetkit
- https://developer.apple.com/documentation/watchconnectivity
- https://developer.android.com/develop/ui/views/appwidgets
