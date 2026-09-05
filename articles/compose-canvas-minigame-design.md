---
title: "Compose MultiplatformのCanvasでミニゲームを4つ作ったときの設計メモ"
emoji: "🎮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["KotlinMultiplatform", "ComposeMultiplatform", "Kotlin", "JetpackCompose"]
published: false
---

# はじめに

個人で作っている猫育成ゲームに、Compose MultiplatformのCanvasで書いたミニゲームが4つ（魚キャッチ、毛糸、おひるね、ダッシュ）入っています。1つ目を作ったあとに共通のインターフェースへ切り出して、残り3つはその形に沿って作りました。

Composeでゲームループを回すと、毎フレームrecompositionが走らないようにするところが一番の悩みどころだったので、そのあたりをめもっとく。

<!-- TODO: そもそもなぜミニゲームをCanvasで書くことにしたか（ゲームエンジンを使わなかった理由）を本人が加筆 -->

# 共通インターフェース

`core:game-engine`モジュールに置いているのはこれだけです。

```kotlin:GameEngine.kt
interface GameEngine {
    @Composable
    fun Render()
    fun update(deltaTime: Float)
    fun onInput(event: InputEvent)
}

sealed class InputEvent {
    data class Tap(val x: Float, val y: Float) : InputEvent()
    data class Drag(val x: Float, val y: Float, val dx: Float, val dy: Float) : InputEvent()
}
```

状態遷移のほうはこれだけ。

```kotlin:GameLogic.kt
interface GameLogic<T> {
    fun update(state: T, deltaTime: Float): T
}
```

`GameLogic`はComposeにもCanvasにも依存しない純粋関数なので、そのままユニットテストが書けます。当たり判定やスコア計算のテストはここに寄せています。

# 状態を2層に分ける

Composeでゲームを書くときに困るのが、`mutableStateOf`にゲーム状態を全部持たせると毎フレームrecompositionが走ってしまうことです。60fpsで動かしたいのに、毎フレーム画面全体が再構成されると重い。

なので状態を2層に分けました。

```kotlin:FishGameEngine.kt
// ゲーム状態本体 — Compose Snapshotシステムで追跡しない（毎フレームの変更通知を回避）
private var _state = FishGameState(timeRemaining = config.gameDuration)

// フレームカウンタ — Canvas drawスコープで読み取りredrawのみトリガー
private var _frameVersion by mutableStateOf(0)

// UIオーバーレイ用の個別State — 値が実際に変わった時のみrecomposition発生
var phase by mutableStateOf(GamePhase.Ready)
    private set
var score by mutableStateOf(0)
    private set
var displaySeconds by mutableStateOf(config.gameDuration.toInt())
    private set
```

- 魚の座標などのゲーム状態本体（`_state`）はただの`var`。Snapshotで追跡しないので、毎フレーム書き換えてもrecompositionは起きない
- 代わりに`_frameVersion`という`Int`のStateをインクリメントする
- スコアや残り秒数など、画面の上に重ねるUIで使う値だけ個別の`mutableStateOf`にする

`_frameVersion`を「Canvasのdrawスコープの中でだけ読む」のがポイントで、こうするとrecompositionではなくredrawだけがトリガーされます。

```kotlin:FishGameEngine.kt
/** Canvas drawスコープ内で呼ぶ。frameVersionを読み取ることでredrawをサブスクライブ */
fun readStateForDraw(): FishGameState {
    _frameVersion // Snapshotシステムへの読み取り登録
    return _state
}

@Composable
override fun Render() {
    val textMeasurer = rememberTextMeasurer()
    Canvas(modifier = Modifier.fillMaxSize()) {
        drawGameScene(readStateForDraw(), textMeasurer)
    }
}
```

`_frameVersion`という行がぽつんと置いてあるだけなので、消されないようにコメントを書いています。

UI用のStateを同期するところは、値が変わったときだけ代入するようにしています。`mutableStateOf`は同じ値なら通知しないので厳密には`if`は要らないはずですが、意図をはっきりさせる意味で書いてます。

```kotlin:FishGameEngine.kt
// UI用Stateを同期（mutableStateOfは同値なら通知しない）
val s = _state
if (phase != s.phase) phase = s.phase
if (score != s.score) score = s.score
val newSeconds = s.timeRemaining.toInt()
if (displaySeconds != newSeconds) displaySeconds = newSeconds
```

残り時間は`Float`で持っていますが、UIには秒（`Int`）で出すので、秒が変わったときだけrecompositionが起きます。

# ゲームループ

`withFrameNanos`でフレームごとに`update`を呼ぶだけです。

```kotlin:FishGameScreen.kt
// Game loop — deltaTimeを1フレーム分(1/60秒)でキャップ
// GCスパイクがあっても魚は1フレーム分以上動かない（ジャンプではなく一瞬停止に見える）
LaunchedEffect(phase) {
    if (phase == GamePhase.Playing) {
        val maxDt = 1f / 60f // 16.67ms — 1フレーム分が上限
        var lastFrameTime = withFrameNanos { it }
        while (true) {
            val frameTime = withFrameNanos { it }
            val rawDelta = (frameTime - lastFrameTime) / 1_000_000_000f
            lastFrameTime = frameTime
            engine.update(rawDelta.coerceAtMost(maxDt))
        }
    }
}
```

`deltaTime`をそのまま渡すと、GCなどで100ms空いたときに魚が一気に飛ぶので、1フレーム分でキャップしています。キャップすると代わりに一瞬止まって見えますが、ワープするよりはマシかなと。

# 入力

`pointerInput`で拾って、`InputEvent`に変換してエンジンに渡します。座標は画面サイズで割って0〜1に正規化しているので、エンジン側は画面サイズを知らなくていいです。

```kotlin:FishGameScreen.kt
.pointerInput(phase) {
    if (phase == GamePhase.Playing) {
        detectHorizontalDragGestures { _, dragAmount ->
            val normalizedDx = dragAmount / canvasWidth
            engine.onInput(InputEvent.Drag(0f, 0f, normalizedDx, 0f))
        }
    }
}
```

# ロジックは純粋関数にする

`GameLogic.update`は`state`を受け取って新しい`state`を返すだけです。魚の移動・キャッチ判定・画面外の除去を1回のループでまとめて処理しています。

```kotlin:FishGameLogic.kt
override fun update(state: FishGameState, deltaTime: Float): FishGameState {
    if (state.phase != GamePhase.Playing) return state

    val newTimeRemaining = state.timeRemaining - deltaTime
    if (newTimeRemaining <= 0f) {
        return state.copy(phase = GamePhase.Finished, timeRemaining = 0f)
    }

    // 単一パス: 移動 + キャッチ判定 + 画面外除去を1回のループで処理
    val basketX = state.basket.x
    val basketWidth = state.basket.width
    val remainingFishes = ArrayList<Fish>(state.fishes.size)
    // ...
}
```

最初は「移動する」「当たり判定する」「消す」を別々の関数に分けてリストを3回まわしていたのですが、1パスにまとめました。

# Rendererはobjectにしてキャッシュする

描画は`object`の`DrawScope`拡張関数にしています。毎フレーム呼ばれるので、`Path`や`Brush`や`Color`をその都度作らないようにキャッシュしています。

```kotlin:FishGameRenderer.kt
object FishGameRenderer {

    // キャッシュ済みPathオブジェクト（reset()で再利用）
    private val fishTailPath = Path()
    private val basketPath = Path()

    // 背景グラデーションキャッシュ
    private var cachedBackgroundBrush: Brush? = null
    private var cachedBackgroundWidth = 0f
    private var cachedBackgroundHeight = 0f

    // FishType色キャッシュ（毎フレームColor()生成を回避）
    private val fishTypeColors = HashMap<FishType, Color>(FishType.entries.size)
}
```

背景のグラデーションはCanvasのサイズが変わったときだけ作り直します。

# フレーム落ちを測る

体感で「なんか重い」と言っていても仕方ないので、フレーム間隔を測ってログに出すようにしました。

```kotlin:FishGameEngine.kt
if (frameGapMs > 20f) {
    slowFrameCount++
    Napier.w(tag = "FishPerf") {
        "JANK: gap=${frameGapMs.toInt()}ms fish=${s.fishes.size} fx=${s.catchEffects.size}"
    }
}

// 1秒ごとにサマリーログ
if (sinceLastLog >= 1000) {
    Napier.d(tag = "FishPerf") {
        "FPS=$frameCount slow=$slowFrameCount maxGap=${maxFrameGapMs.toInt()}ms fish=${s.fishes.size} fx=${s.catchEffects.size}"
    }
}
```

`deltaTime`はキャップしているので、実際のフレーム間隔は別に測る必要がありました。魚とエフェクトの数も一緒に出しておくと、重いときに何が増えているかがわかります。

<!-- TODO: 実機で実際にどれくらいのFPSが出ていたか、どの端末で重かったかを本人が加筆 -->

# おわりに

ゲーム状態をSnapshotの外に置いて、redrawだけをトリガーする、というのが分かってからは素直に書けるようになりました。2つ目以降のミニゲームは`GameEngine`と`GameLogic`を実装するだけなので、だいぶ楽でしたし、ロジックが純粋関数なのでテストも書きやすかったです。

ただ、この書き方はComposeの流儀からは外れてる気もするので、もっといいやり方があれば知りたいなと思います。

# 参考

- https://developer.android.com/develop/ui/compose/graphics/draw/overview
- https://developer.android.com/develop/ui/compose/performance
