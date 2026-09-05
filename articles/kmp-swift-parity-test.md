---
title: "KotlinとSwiftで同じ減衰計算を書いたら、総当たりテストで３つバグが出た話"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kotlin", "KotlinMultiplatform", "Swift", "iOS"]
published: false
---

# はじめに

個人で作っているアプリ（KMP + Compose Multiplatformの猫育成ゲーム）で、ウィジェットに表示している猫のステータスの数値が、時間が経っても全然動かないという問題がありました。

<!-- TODO: #960 に気づいた経緯（自分で見つけた？誰かに言われた？）を本人が加筆 -->

これを直していく中で、猫のステータスの減衰計算がKotlin（KMPの共通コード）、iOSウィジェット（Swift）、Apple Watch（Swift）の3か所に存在することになって、「Kotlin側とSwift側で本当に同じ結果になるのか？」を総当たりで比較したらバグが3つ出てきたので、そのメモです（実装はClaude Codeにやらせてます）。

# そもそもなぜ数値が動かなかったのか

猫のステータス（満腹度・幸福度・清潔度・元気）は、ローカルDBに「最終更新時点の生値 + `lastStatUpdateAt`」で保存していて、時間経過による減衰は表示するときに`StatDecayCalculator.applyDecay()`で計算する設計になっています。

Kotlin側の計算の中身はこれです。

```kotlin:StatDecayCalculator.kt
fun calculateDecay(stored: Int, elapsedMinutes: Long, decayMinutes: Long, minimum: Int): Int {
    if (decayMinutes <= 0) return stored
    val decayed = (elapsedMinutes / decayMinutes).toInt()
    return (stored - decayed).coerceAtLeast(minimum)
}
```

「経過分数 ÷ 減衰間隔（分）」を切り捨てた数だけ引いて、下限でクランプ。それだけです。

画面側はこれを適用していたんですが、ウィジェット用のデータを作る箇所だけ適用していなくて、DBの生値をそのまま表示していました。そのときの原因説明をそのまま貼ります。

> UI 層（`CareViewModel`, `GameSelectViewModel`）は適用していましたが、ウィジェットのデータ生成箇所は適用しておらず、DB の生値をそのまま表示していました。DB の値はお世話・`statTick` 時にしか変わらないため、**何時間経ってもウィジェットの数値が動きません**でした。

# Androidは簡単、iOSは面倒

Androidは、ウィジェット更新のたびにアプリのプロセスが起動してDBを読めるので、`CatStatusWidgetDataProvider`で`applyDecay`を適用するだけで終わりました。

iOSはそうはいかなくて、ウィジェット拡張（NekoWidgetExtension）からはアプリのローカルDBが読めず、アプリ本体がApp GroupのUserDefaultsに書き込んだスナップショット（JSON）しか参照できません。なのでKotlin側で減衰を適用してからスナップショットを書いても、アプリを起動しない限り数値が固まったままになります。

しかもKMP側のFramework（`SharedApp`）はiosAppターゲットにしか組み込んでいなくて、ウィジェット拡張からは使えない構成でした。Watchはもっと無理で、そもそもFrameworkを`iosArm64`と`iosSimulatorArm64`向けにしかビルドしてません。

<!-- TODO: Framework をウィジェット拡張にリンクしなかった（できなかった？）理由を本人が加筆。サイズ？ビルド時間？ -->

そこで、スナップショットに「減衰の計算材料」（基準時刻、各ステータスの減衰間隔、下限）を含めて、拡張側のSwiftで将来時刻ぶんのタイムラインエントリ（15分刻み × 6時間 = 24エントリ）を先に計算する方針にしました。つまり、Kotlinの`StatDecayCalculator`と同じ計算をSwiftにも書く、ということです。

（減衰でコンディションが変わると猫の画像とラベルも変わるので、コンディション判定`CatCondition.evaluateFromStats`の移植`CatConditionEvaluator`もSwift側に必要になりました）

ここから、同じ計算を2言語で書くことによるバグが出てきます。

# バグ1: Floatのまま引き算すると約24%で1ずれる

最初、Swift側の減衰はスナップショットの値が0..1のFloat（`hungerPercent`など）なので、そのままFloatで引き算していました（このコードはコミットに残ってないです）。

Kotlin側は整数（0..100のポイント）で計算しているので一致するのか？と思って、0..100 × 0..100の全組み合わせで実測してみたら...

> **Float 減算による 1 ポイントずれ**: 当初 Swift 側の減衰をパーセント（Float）のまま引き算していたが、
> 0..100 × 0..100 の全組み合わせを実測したところ **約24%（5151通り中1238通り）で表示が1ずれる**ことが判明。
> Kotlin と同じく整数ポイントで計算するよう変更した

（作業メモ`tasks/todo.md`より）

Floatの誤差で`xx.99999...`になったところを`Int()`で切り捨てて1落ちる、よくあるやつです。Kotlinと同じく整数ポイントに変換してから計算するように直しました。直した時点のSwift側がこれです。

```swift:WidgetCatData.swift
// Float のまま引き算すると誤差で 1 ポイントずれることがあるため、
// Kotlin 側 `StatDecayCalculator` と同じく整数ポイントで計算する。
func decay(_ percent: Float, _ decayMinutes: Int64?) -> Int {
    let stored = Int((percent * 100).rounded())
    guard let decayMinutes, decayMinutes > 0 else { return stored }
    return max(stored - Int(elapsedMinutes / decayMinutes), minimum)
}
```

ついでに、表示側の`Int(percent * 100)`も同じFloatの誤差で53%が52、59%が58と表示されるバグが元からあったので、`Int((percent * 100).rounded())`に直しました（iOS・Android両方）。

# バグ2: KotlinとSwiftで減衰が二重にかかる

最初のコミットでは、Kotlin側で`applyDecay`した減衰済みの値と、書き込み時刻を基準時刻としてスナップショットに載せていました。

```kotlin:WidgetDataBridge.kt（修正前）
val nowEpochSeconds = Clock.System.now().epochSeconds
val cat = StatDecayCalculator.applyDecay(storedCat, nowEpochSeconds * 1000L, careConfig)
// ...
hungerPercent = (cat.hunger / 100f).coerceIn(0f, 1f),
// ...
statBaseEpochSeconds = nowEpochSeconds,
```

Swift側はこの基準時刻からの経過ぶんをさらに減衰させます。一見正しそうなんですが、コードレビューで重大指摘が来ました。作業メモから引用します。

<!-- TODO: レビューは何にやらせたか（Claude Code のレビュー用エージェント？人？）を本人が加筆 -->

> **[重大] iOS で減衰の floor 除算が二重に入りアプリ本体と最大1ポイントずれる**
> 当初はウィジェット拡張に「減衰済みの値 + 書き込み時刻」を渡していたため、
> `floor(e1/d) + floor(e2/d) ≤ floor((e1+e2)/d)` により剰余が切り捨てられ恒常的に高く出ていた。
> コンディション境界に乗るとラベルと猫画像まで食い違う。

なるほど...。Kotlin側で「最終更新から書き込みまでの経過`e1`」を`d`で割って切り捨て、Swift側で「書き込みから表示までの経過`e2`」を`d`で割って切り捨てると、それぞれの余りが捨てられるので、アプリ本体が1回で計算する`floor((e1+e2)/d)`より減りが小さくなることがあります。

具体例だと（後述のテストコードから）、保存値80・減衰間隔20分で、最終更新の50分後にアプリが書き込み、その30分後（最終更新から80分後）にウィジェットが表示する場合、

- アプリ本体: 80 - floor(80/20) = 76
- 二重適用: Kotlinで 80 - floor(50/20) = 78 → Swiftで 78 - floor(30/20) = 77

で1ポイントずれます。あとで書いたメモには「約50%の確率で1ポイントずれる」と書いてあります。

対策は、スナップショットには**減衰前の生値と`lastStatUpdateAt`**を載せて、floor除算をSwift側の1回だけにすること。

```diff kotlin:WidgetDataBridge.kt
-            val nowEpochSeconds = Clock.System.now().epochSeconds
-            val cat = StatDecayCalculator.applyDecay(storedCat, nowEpochSeconds * 1000L, careConfig)
+            val nowMillis = Clock.System.now().toEpochMilliseconds()
+            val statBaseEpochSeconds = storedCat.lastStatUpdateAt?.let(::parseEpochSecondsOrNull)
+
+            // 書き込み時点のコンディションは減衰後の値で判定する
+            // （拡張側が減衰できない＝lastStatUpdateAt が無いときの表示値にもなる）
+            val decayedCat = StatDecayCalculator.applyDecay(storedCat, nowMillis, careConfig)
+            val condition = CatCondition.evaluate(decayedCat)
 
-            val condition = CatCondition.evaluate(cat)
...
-                hungerPercent = (cat.hunger / 100f).coerceIn(0f, 1f),
+                hungerPercent = (storedCat.hunger / 100f).coerceIn(0f, 1f),
...
                 imageUrl = conditionImageUrls[condition.apiValue],
-                statBaseEpochSeconds = nowEpochSeconds,
+                statBaseEpochSeconds = statBaseEpochSeconds,
```

Kotlin側の`applyDecay`は、書き込み時点のコンディションラベルの判定（`lastStatUpdateAt`がなくてSwift側で減衰できないときのフォールバック表示用）にだけ使う形になりました。

# バグ3: 経過0以下で下限クランプが効かない

ここまで直したあと、「ウィジェットの表示値とアプリ本体の計算が本当に一致するか」を総当たりで比較しました。そのときの検証内容から。

> **ウィジェット表示値 vs アプリ本体の一致**: 保存値 0..100 × 経過 -120..2880分（時計ずれによる未来時刻含む） × 減衰間隔5種 × 下限2種の **1,011,010 通りを総当たりで比較し mismatch 0**

mismatch 0になるまでに、1,640件の不一致が出ました。

> 2. 経過時間が0以下のとき `statMinimum` のクランプが効かず**1,640件の不一致** → early return を廃止

Swift側にこう書いていたのが原因です。

```swift:WidgetCatData.swift（修正前）
let elapsedMinutes = Int64(date.timeIntervalSince1970 - TimeInterval(baseEpochSeconds)) / 60
guard elapsedMinutes > 0 else { return self }
```

経過が0以下（端末の時計ずれで基準時刻が未来になっているとか）なら減衰しないので早期returnしていたんですが、Kotlin側は経過を0にクランプしたうえで`calculateDecay`を通すので、保存値が下限`statMinimum`より小さいときは下限まで引き上げられます。Swift側は早期returnでそれをスキップしていたので、「保存値 < 下限 かつ 経過 ≤ 0」の組み合わせが全部ずれてました。

直したのがこれ。

```diff swift:WidgetCatData.swift
-        let elapsedMinutes = Int64(date.timeIntervalSince1970 - TimeInterval(baseEpochSeconds)) / 60
-        guard elapsedMinutes > 0 else { return self }
+        // App Group の値が壊れていると Int64(_:) がトラップして拡張ごとクラッシュするため
+        // exactly: で安全に変換する
+        let elapsedSeconds = date.timeIntervalSince1970 - TimeInterval(baseEpochSeconds)
+        guard let elapsedSecondsInt = Int64(exactly: elapsedSeconds.rounded(.towardZero)) else { return self }
+        // 端末時計のずれ等で基準時刻が未来になっても巻き戻さない（Kotlin の coerceAtLeast(0) と同じ）。
+        // 経過が 0 でも statMinimum のクランプは効かせる必要があるため early return しない。
+        let elapsedMinutes = max(elapsedSecondsInt / 60, 0)
```

3つとも「アプリ本体とウィジェットで数値が1違う」という出方なので、目視ではまず気づかないと思います。

# 総当たりをテストとして残す

ここまでの総当たりは使い捨てのスクリプトで、リポジトリには残っていませんでした。ウィジェット拡張にはユニットテストのターゲット自体がなかったためです（`project.pbxproj`の手編集が必要でリスクが高いのでこのときは見送りました）。

なので別途`NekoWidgetTests`ターゲットを追加して、総当たりを`WidgetStatDecayParityTests`として固定しました。ファイルの冒頭コメントがそのまま経緯になってます。

```swift:WidgetStatDecayParityTests.swift
/// ウィジェットの減衰計算が iPhone アプリ本体（Kotlin `StatDecayCalculator`）と
/// 一致することを総当たりで検証する。
///
/// このファイルには Kotlin `StatDecayCalculator.calculateDecay` / `applyDecay` の
/// **Swift 参照実装**を手で写している。参照実装は Kotlin 側の変更に自動追随しないため、
/// `core:domain` の `StatDecayCalculator` や `CatCondition.evaluateFromStats` を変更したら
/// 必ずこのファイルの参照実装と期待値も更新すること。
///
/// このウィジェット実装では、この種の総当たり検証でしか見つからないバグを 3 件検出している。
/// 1. Float のまま減算して約 24% の組み合わせで 1 ずれる
/// 2. Kotlin と Swift で減衰が二重に合成され約 50% の確率で 1 ポイントずれる
/// 3. 経過が 0 以下のとき `statMinimum` のクランプが効かず 1,640 件の不一致
```

やり方は、Kotlinの`calculateDecay`をSwiftに写した「参照実装」をテストの中に置いて、

```swift
private func kotlinCalculateDecay(
    stored: Int, elapsedMinutes: Int64, decayMinutes: Int64, minimum: Int
) -> Int {
    if decayMinutes <= 0 { return stored }
    let decayed = Int(elapsedMinutes / decayMinutes)
    return max(stored - decayed, minimum)
}
```

本物の`WidgetCatData.decayed(at:)`と総当たりで比較するだけです。保存値0..100 × 減衰間隔9種 × 下限3種 × 経過分数（負値含む）を回してます。

```swift
func testDecayMatchesKotlinReferenceImplementation() {
    var checked = 0
    for storedPoints in 0...100 {
        for decayMinutes in decayMinutesCases {
            for minimum in minimumCases {
                let data = makeData(
                    storedPoints: storedPoints,
                    decayMinutes: decayMinutes,
                    minimum: minimum
                )
                for elapsed in elapsedMinutesCases {
                    let now = base + elapsed * 60
                    let actual = data.decayed(
                        at: Date(timeIntervalSince1970: TimeInterval(now))
                    )
                    let expected = kotlinCalculateDecay(
                        stored: storedPoints,
                        elapsedMinutes: kotlinElapsedMinutes(
                            baseEpochSeconds: base, nowEpochSeconds: now
                        ),
                        decayMinutes: decayMinutes,
                        minimum: minimum
                    )
                    // メッセージは autoclosure なので失敗時のみ組み立てられる
                    // （毎回文字列補間すると総当たりが極端に遅くなる）
                    XCTAssertEqual(
                        points(actual.hungerPercent), expected,
                        "hunger stored=\(storedPoints) decay=\(decayMinutes) min=\(minimum) elapsed=\(elapsed)"
                    )
                    // ...happiness / cleanness / energy も同様
                    checked += 1
                }
            }
        }
    }
    XCTAssertGreaterThan(checked, 10_000)
}
```

経過分数（`elapsedMinutesCases`）は境界の前後（7, 8, 9, 15, 16, 17...）と0、負値（-1, -60, -10000。未来時刻）を混ぜてます。

バグ2の二重適用も、「減衰済みの値を送る設計だとずれる」ことを回帰テストにしてあります。さっきの76 / 77 / 78の例はここから取りました。

```swift
// 誤った設計: Kotlin 側で減衰済みの値（50 分ぶん）を書き込み、基準時刻を書き込み時刻にする
let preDecayed = kotlinCalculateDecay(
    stored: storedPoints,
    elapsedMinutes: syncElapsed,
    decayMinutes: decayMinutes,
    minimum: minimum
)
XCTAssertEqual(preDecayed, 78) // 80 - (50/20 = 2) = 78
var doubleApplied = makeData(
    storedPoints: preDecayed, decayMinutes: decayMinutes, minimum: minimum
)
doubleApplied.statBaseEpochSeconds = syncEpoch
let wrong = doubleApplied.decayed(at: displayDate)
// 78 - (30/20 = 1) = 77 → 本体の 76 と 1 ポイントずれる
XCTAssertEqual(points(wrong.hungerPercent), 77)
XCTAssertNotEqual(points(wrong.hungerPercent), expected)
```

コンディション境界だとラベルまで変わります（`testDoubleAppliedDecayCanFlipConditionLabel`）。保存値82・減衰間隔20分で60分後に表示すると、正しくは79で「元気」なのに、41分時点で減衰済みの80を送ると残り19分ぶんが切り捨てられて80のまま「絶好調」になる、というやつです。

このテストの結果は`xcodebuild test -only-testing:NekoWidgetTests ✅ 39 passed / 0 failed`でした。

# Apple Watchにも同じバグがあった

この修正をやっている途中で、Watch側（`WatchDataBridge.kt`と`ComplicationTimelineProvider.swift`）にもまったく同じバグが残っていることに気づいて、別issueに分けて対応しました。こっちもWatchへ送るペイロードに生値を載せていて、Complicationは単一エントリのタイムラインだったので、iPhoneが起動していない間は値が完全に固まってました。

修正はウィジェットと同じ方針です。`WatchSyncPayload`のdocコメントに理由をそのまま書いてあります。

```kotlin:WatchSyncPayload.kt
/**
 * iPhone から Apple Watch に送る同期ペイロード。
 *
 * `xxxPercent` には **減衰前の生値**（ローカル DB の値をそのまま 0..1 に正規化したもの）を入れる。
 * 減衰済みの値を入れると Watch 側の減衰と切り捨てが二重に入り、アプリ本体と最大 1 ポイント
 * （コンディション境界ではラベルと画像まで）ずれるため（実際に踏んだバグ）。
 *
 * Watch 側は [statBaseEpochSeconds] からの経過時間と実効減衰間隔を使って自前で減衰させる。
 *
 * 新しいフィールドは必ずデフォルト値付きで追加すること。
 * Watch アプリのバージョンが iPhone 側より古い／App Group に旧フォーマットの JSON が
 * 残っている状況でもデコードが失敗しないようにするため。
 */
```

追加したフィールドはこれです。

```kotlin:WatchSyncPayload.kt
    /**
     * 上記ステータス値の基準時刻（エポック秒）= 猫の `lastStatUpdateAt`。
     * 書き込み時刻ではなく最終更新時刻を基準にすることで、アプリ本体と同じ計算結果になる。
     * null なら Watch 側では減衰しない。
     */
    val statBaseEpochSeconds: Long? = null,
    /**
     * 各ステータスが 1 ポイント減るまでの分数（**実効値**）。0 以下なら減衰しない。
     * オフライン倍率（statOfflineMultiplier）と復帰ボーナス半減を反映済みなので、
     * Swift 側はこの分数で経過分を割るだけでよい。
     */
    val hungerDecayMinutes: Long = 0,
    // ...happiness / cleanness / energy も同様
    /** 減衰の下限値（0..100 のポイント）。 */
    val statMinimum: Int = 0,
```

ここでSwift側の減衰計算がNekoWidgetとNekoWatchで2重（Kotlinを入れると3重）になるのを避けるため、`Shared/CatStatDecay.swift`に集約して、NekoWidgetExtension / NekoWatch / NekoWatchWidgetExtensionの3ターゲットで共有するようにしました。計算式はさっき総当たり済みのものをそのまま移しただけなので、ウィジェット側の挙動は変わってません。

Watch側にも同じ総当たりの`WatchStatDecayParityTests`を置いてあります。ちなみに`NekoWatchTests`はmainでコンパイルが通らず、CIもxcodebuildのテストを回していなかったので、しばらく前から一度も動いていなかったことがここで判明しました...。

# おまけ: クライアント全体がサーバの2倍速で減っていた

これは総当たりとは別で、ここまでの調査中に`CareConfig.statOfflineMultiplier`がどこからも参照されていないのに気づいたのがきっかけです。

サーバ（Rust）は猫一覧取得時に`is_active = false`で減衰を適用するので、実効レートが`decay_minutes × statOfflineMultiplier`（=2）になるんですが、クライアントの`StatDecayCalculator`は常にアクティブレートで計算していました。そのときの例だと、

| | 計算 | 表示 |
|---|---|---|
| クライアント予測 | 90 − 360/10 | **54** |
| サーバ実測（`is_active=false`） | 90 − 360/20 | **72** |

アプリを開いてサーバと同期すると数値が跳ね上がる、という現象になります。減衰計算はサーバのRustも含めると4か所目でした。

この修正で`effectiveDecayMinutes()`（`base × statOfflineMultiplier × (半減中なら 2)`）を唯一の計算元にして、ウィジェット・Watchに渡す減衰間隔もこの実効値にしました。こっちにもサーバの計算式の参照実装と直積で比較する`StatDecayServerParityTest`を置いてあります。

# おわりに

1ポイントのずれは目視ではまず気づけないので、総当たりで比較しないと3つとも見つからなかったと思います。ただ、テスト内の参照実装はKotlin側の変更に自動追随しないので、「`StatDecayCalculator`を変えたらSwift側も直す」とコメントで書いてある状態なのは変わっていません。

<!-- TODO: 次にやりたいこと（Framework を拡張から使えるようにする？参照実装の自動生成？）を本人が加筆 -->

<!-- TODO: リポジトリが private なので「参考」に載せる URL がない。公開できる URL があれば本人が「# 参考」を追加 -->
