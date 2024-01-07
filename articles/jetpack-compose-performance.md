---
title: "Jetpack Composeのリスト実装でパフォーマンスに関して気をつける３つのこと"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JetpackCompose","Android"]
published: true
---

# はじめに
Jetpack Composeのリスト実装はほんとに簡単で直感的に書けちゃうなと思いますし、それがいいところでもあると思うのですが、パフォーマンス観点で気をつけたほうがいいなと思うことがあったので、具体的な実装を示しながら、どういう問題があり、どう解決するとよさそうかを紹介していこうと思います。
以下に出てくるコードは下のリポジトリにおいています。

https://github.com/kwmt/JetpackComposePlayGround/tree/main/feature/samples/src/main/java/net/kwmt27/jetpackcomposeplayground/feature/samples/list/instagram

# 具体的な実装例

どういう問題があるかの説明の前にどういう実装をしたのか、具体的な例をまずは説明したいと思います。

まずはこちらの動画を見てください。

![](https://storage.googleapis.com/zenn-user-upload/4d5705ce3b70-20231212.gif)

こちらは、Instagramの検索タブのUIを真似て実装したものでして、
- グレーの正方形っぽいのが画像を表示する領域で
- 縦長の部分が動画

だと思ってください。緑色は動画再生中を表していて、赤は動画が止まっていることを表しています。始めは1番目の動画が再生中の状態で、スクロールすると2番目の動画再生され、再生中だった一番目の動画は停止します。このように、スクロールすると再生する動画が次々と変わっていくように実装しました。（切り替わるタイミングはもう少し考慮の余地があると思いますが、今回の説明ではこだわっても仕方ないので適当です）

想像してみてください。画像が見えてるところだけでも12枚表示し、動画は3つ表示されていて、動画はそのうちの1つしか再生されないとはいえ、これらが一気に表示する処理は重いのでは？と想像できるのではないでしょうか。

## 構造の説明
次に、説明のためにどのような構造になっているかを説明します。
リスト全体の構造は下図のようになっており、各アイテムには `GridRowItem` という名前でアイテムのComposable関数を作っています。

| リスト全体の構造 |
| ---- |
| ![](https://storage.googleapis.com/zenn-user-upload/2dacad06c964-20231213.png) |


実装は、`LazyColumn`の`itemsIndexed`に`GridRowItem`を置くイメージです。
```kotlin
LazyColumn(
  // 省略,
) {
    itemsIndexed(gridListData.list) { index, gridRowData ->
	// 省略
        GridRowItem(
            gridRowData = /*省略*/,
            width = /*省略*/,
            isPlay = /*省略*/,
        )
    }
}
```

`GridRowItem`の構造は下図のようになっていて、
| GridRowItemの構造 |
| ---- | 
![](https://storage.googleapis.com/zenn-user-upload/233f50d5be77-20231213.png) |



`GridRowItem`の実装は`GridItemImages` と `ItemMovie` を横に(Row)並べているというイメージです。（`ItemMovie`が右にあったり左にあったりするのは、今回の説明にあまり意味ありません。ただインスタっぽくしたかっただけです。ちなみに実装は`itemsIndexed`の`index`を使って`ItemMovie`を表示する・しないを切り替えてるだけです。）

```kotlin
@Composable
private fun GridRowItem(
    gridRowData: GridRowData,
    width: Dp,
    isPlay: Boolean,
) {
    val itemImageDataList: List<List<ItemData>> = gridRowData.list.take(4).chunked(2)
    val itemMovieData = gridRowData.list.last()

    Row {
        // 省略
        GridItemImages(itemImageDataList, width)
        ItemMovie(itemMovieData, width, isPlay)
    }
}
```

## スクロールすることで再生したい動画を切り替えるには？

ユーザーがリストをスクロールしたら、1番目の動画が再生中だったのを1番目の動画の再生中の状態を停止状態にし、2番目の動画を再生中の状態にするにはどうしたらいいでしょうか。

スクロールの状態を取得するには、[`rememberLazyListState`](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/package-summary#rememberLazyListState(kotlin.Int,kotlin.Int))を使って[`LazyListState`](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/LazyListState)を取得し、`firstVisibleItemScrollOffset` や[`firstVisibleItemIndex`](https://developer.android.com/reference/kotlin/androidx/compose/foundation/lazy/LazyListState#firstVisibleItemIndex())を使ってスクロールの状態を取得します。上記で`GridRowItem`の実装を示しましたが、`isPlay`の部分に`playMovieIndex==index`を渡して、画面上に見えてる1番目のitemだったら動画を再生するようにしてみます。


```kotlin
@Composable
private fun InstagramSearchListLayout(
    gridListData: GridListData,
) {
    val listState = rememberLazyListState()
    val playMovieIndex =
        if (listState.firstVisibleItemScrollOffset == 0) {
            listState.firstVisibleItemIndex
        } else {
            listState.firstVisibleItemIndex + 1
        }
    BoxWithConstraints {
        LazyColumn(
            modifier = Modifier
                .fillMaxWidth(),
            verticalArrangement = Arrangement.spacedBy(1.dp),
            state = listState,
        ) {
            itemsIndexed(gridListData.list) { index, gridRowData ->
                GridRowItem(
                    gridRowData = gridRowData,
                    width = maxWidth / 3,
                    moviePosition = moviePosition,
                    isPlay = index == playMovieIndex
                )
            }
        }
    }
}
```

`GridRowItem`内の`ItemMovie`の実装は以下のような感じです。`isPlay`フラグを見て色とテキストを切り替えています。

```kotlin
@Composable
private fun ItemMovie(item: ItemData, width: Dp, isPlay: Boolean) {

    val color = if (isPlay) Color.Green else Color.Red
    Column(
        modifier = Modifier
            .size(width = width, height = width * 2 + 1.dp)
            .background(color = color),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "${item.id} 動画",
            style = TextStyle(fontSize = 20.sp, color = Color.White)
        )
        val text = if (isPlay) "Playing" else "Stop"
        Text(
            text = text,
            style = TextStyle(fontSize = 20.sp, color = Color.White)
        )
    }
}
```

# どういう問題があるのか？その対策は？
具体的な実装例を見てきましたが、これのどこに問題があり対策はそれぞれどうするのかを見ていきたいと思います。

## 少しスクロールするだけでRecomposeされてしまう

### 問題
問題があるのはこの部分です。

```kotlin
val playMovieIndex =
    if (listState.firstVisibleItemScrollOffset == 0) {
        listState.firstVisibleItemIndex
    } else {
        listState.firstVisibleItemIndex + 1
    }
```


この実装で動かしてRecompositionカウントを確認してみると次のようになります。

![](https://storage.googleapis.com/zenn-user-upload/6bffb3ae3d86-20231230.gif)

`LazyColumn` とitemとして作っている `GridRowItem`のRecompositionカウントが、少しスクロールしただけで数十回Recomposeされていることがわかると思います。



`InstagramSearchListLayout`を再掲します。

```kotlin
@Composable
private fun InstagramSearchListLayout(
    gridListData: GridListData,
) {
    val listState = rememberLazyListState()
    val playMovieIndex =
        if (listState.firstVisibleItemScrollOffset == 0) {
            listState.firstVisibleItemIndex
        } else {
            listState.firstVisibleItemIndex + 1
        }
    BoxWithConstraints {
        LazyColumn(
            modifier = Modifier
                .fillMaxWidth(),
            verticalArrangement = Arrangement.spacedBy(1.dp),
            state = listState,
        ) {
            itemsIndexed(gridListData.list) { index, gridRowData ->
                GridRowItem(
                    gridRowData = gridRowData,
                    width = maxWidth / 3,
                    moviePosition = moviePosition,
                    isPlay = index == playMovieIndex
                )
            }
        }
    }
}
```
`playMovieIndex`はLazyColumnのitemsIndexedに渡しているのですが、`playMovieIndex`はスクロール状態を読み取るので、InstagramSearchListLayout Composable全体がスクロールするたびにRecomposeされるため、`LazyColumn`と`GridRowItem` がRecomposeされていました。


### 対策 [`derivedStateOf`](https://developer.android.com/jetpack/compose/side-effects?hl=ja#derivedstateof)を使う

`playMovieIndex`の部分を`derivedStateOf`を使って次のように囲うだけです。

```kotlin
val playMovieIndex by remember {
    derivedStateOf {
        if (listState.firstVisibleItemScrollOffset == 0) {
            listState.firstVisibleItemIndex
        } else {
            listState.firstVisibleItemIndex + 1
        }
    }
}
```

このようにして動かした動画が次の動画です。

![](https://storage.googleapis.com/zenn-user-upload/32349e5408b5-20231230.gif)

前とは違って少しスクロールしただけでは数十回もRecomposeされていません。
更新してほしいタイミングで`GridRowItem`だけRecomposeされていることがわかるかと思います。
ちなみに、`derivedStateOf`を使わなかったら[lintが警告](https://googlesamples.github.io/android-custom-lint-rules/checks/FrequentlyChangedStateReadInComposition.md.html)してくれるので、その警告を無視しないようにすると良さそうです。
![](https://storage.googleapis.com/zenn-user-upload/72fb16ed8942-20231230.png)

少しスクロールするだけで数十回もRecomposeされてしまう問題の対策はこれだけなんですが、動画をよく見るとたまに青いグラデーションがハイライトされてるのが確認できると思うのですが、ハイライトされるのはRecomposeされてることを意味します。（これは[LayoutInspectorの機能](https://developer.android.com/jetpack/compose/tooling/layout-inspector?hl=ja#recomposition-counts)です。ちなみにハイライトされるカラーも選択できます。）


![](https://storage.googleapis.com/zenn-user-upload/cec8f6aa80be-20231230.png)


先程の動画をみたときに動画がRecomposeされるのはいいのですが、よくみるとなにも変更がないはずの画像までRecomposeされていることがわかるかと思います。何も変更がない画像はRecomposeしたくないと思うので、次はそちらについて見ていきます。

## 変更がないComposable関数がRecomposeされてしまう
### 問題

先程と繰り返しになりますが、変更がないはずの画像までRecomposeされてしまう問題について考えてみます。

`GridRowItem`の実装を再掲します。

```kotlin
@Composable
private fun GridRowItem(
    gridRowData: GridRowData,
    width: Dp,
    isPlay: Boolean,
) {
    val itemImageDataList: List<List<ItemData>> = gridRowData.list.take(4).chunked(2)
    val itemMovieData = gridRowData.list.last()

    Row {
        // 省略
        GridItemImages(itemImageDataList, width)
        ItemMovie(itemMovieData, width, isPlay)
    }
}
```

これまで見てきてように、スクロールに依存して`isPlay`が変わる可能性があります。引数の値が更新されると、そのComposable関数（ここでは`GridRowItem`）はRecomposeされます。
そのため、`GridRowItem`内の動画（`ItemMovie`）はRecomposeされても良いかもしれませんが、画像（`GridItemImages`）までRecomposeされてしまっていました。

### 対策 読み取りを遅延する

対策としては、`isPlay`の読み取りを遅延させればよいです。
読み取りを遅延させるというのは、Booleanを返す関数ラムダを渡すようにするということです。
具体的なコードは以下のようになります。

```kotlin
@Composable
private fun GridRowItem(
    gridRowData: GridRowData,
    width: Dp,
    isPlay: () -> Boolean,
) {
    val itemImageDataList: List<List<ItemData>> = gridRowData.list.take(4).chunked(2)
    val itemMovieData = gridRowData.list.last()

    Row {
        // 省略
        GridItemImages(itemImageDataList, width)
        ItemMovie(itemMovieData, width, isPlay)
    }
}
```

これによって、`isPlay`はラムダになったので、`isPlay`の値が変わることはなく、GridRowItem関数はRecomposeされないということになり、画像（`GridItemImages`）もRecomposeされなくなります。

下の動画は、この対策を実行した動画です。動画（`ItemMovie`）だけハイライトされていて、画像（`GridItemImages`）はハイライト表示されていない事がわかるかと思います。

![](https://storage.googleapis.com/zenn-user-upload/2215b0e2ec42-20231230.gif)

ただ、よく見ると変更されない動画（`ItemMovie`）もRecomposeされてるので、次はそれについて見ていきます。


## 変更がないComposable関数がRecomposeされてしまう(パート２)
### 問題
動画だけRecomposeされるになったのはいいのですが、変更のない動画（`ItemMovie`）もRecomposeされてしまっています。
そこまで気にしなくてもいいかもしれませんが、変更のない動画をRecomposeしないようにもできるので紹介だけしておきます。

### 対策 LazyListStateを渡す

対策としては、スクロールするたびに変更してほしいのは動画だけなので、`ItemMovie` Composable関数内でlistのstateを読み取るという方法です。

```kotlin
@Composable
private fun ItemMovie(item: ItemData, width: Dp, listState: LazyListState, index: Int) {
    val isPlay by remember {
        derivedStateOf {
            val firstVisibleItemIndex = if (listState.firstVisibleItemScrollOffset == 0) {
                listState.firstVisibleItemIndex
            } else {
                listState.firstVisibleItemIndex + 1
            }
            firstVisibleItemIndex == index
        }
    }

    val color = if (isPlay) Color.Green else Color.Red
    Column(
        modifier = Modifier
            .size(width = width, height = width * 2 + 1.dp)
            .background(color = color),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(
            text = "${item.id} 動画",
            style = TextStyle(fontSize = 20.sp, color = Color.White)
        )
        val text = if (isPlay) "Playing" else "Stop"
        Text(
            text = text,
            style = TextStyle(fontSize = 20.sp, color = Color.White)
        )
    }
}
```

`ItemMovie`が`LazyListState`やリストの`index`を知ってしまうので、もし`ItemMovie`がリストとは関係ないところでも使いたいものだったとしたらもう少し工夫は必要かもですが、これで余計なRecomposeがされなくなります。
下の動画は、この実装を動かした時の動画です。
![](https://storage.googleapis.com/zenn-user-upload/515f57ff7637-20231230.gif)

isPlayがtrueからfalseになったものと、falseからtrueになった２つの動画のみハイライト（Recompose）され、それ以外はハイライトされていないことがわかるかと思います。

# まとめ
- UIの状態を更新したいタイミング以上にRecomposeされている場合は、`derivedStateOf`を使うと良いです。今回はスクロール状態の読み取りでしたが、[このYoutube](https://youtu.be/ahXLwg2JYpc?t=982)では、テキストフィールドに文字を入力してある条件を満たさないとボタンが有効にならない仕様があったときの例を紹介されていました。
- Composableの引数にラムダを渡すとラムダは変更されないのでRecomposeされないということを紹介しました。Composeには３つのフェーズ（Composition、Layout、Draw）があって、Compositionフェーズをスキップしているからなのですが、その説明は[このあたりのYoutube](https://youtu.be/ahXLwg2JYpc?t=248)がわかりやすいかなと思います。
- LazyListStateインスタンスを、スクロールの状態によって更新されるComposableまで渡していくというのを紹介しました。とくにどこかを参考にしたわけではなくやってみたら、変更がないComposableがRecomposeしなくなっただけで正しい方法じゃないかもしれませんが、LazyListStateを渡すのは意外とアリなんだというのを忘れないために書いた感じです。

# おわりに
パフォーマンスに関しての動画を見ていると、必ずと言っていいほど免責事項の説明があります。リリースモードで確認しろ、R8有効にしろ、対策があらゆる環境で有効ではないかもしれない、計測しろなどなど。
リリースビルドは時間かかるから開発中はあんまりしたくないなぁと思いますし、デバッグビルドでも明らかにガタついて遅いのはすぐにわかるので、もし遅いときはこういった対策をしてみると良いのかもしれません。
パフォーマンス観点で引数の[安定性](https://youtu.be/ahXLwg2JYpc?t=511)についても書きたかったのですが、それはまた別の記事で。

# 参考

- [More performance tips for Jetpack Compose](https://youtu.be/ahXLwg2JYpc?si=raDO8Swgo9DvWlz2)
    - Composeには３つのフェーズ（Composition、Layout、Draw）があるという基本的なことから学べます。
    - Composable関数の引数の型がパフォーマンスに影響あるかもしれないのですが、その説明も参考になるかと思います。
- [Debugging Jetpack Compose](https://www.youtube.com/watch?v=Kp-aiSU8qCU)
    - Android Studio HeadghogからデバッガーでComposableの引数の状態を確認できるようになったなどのデバッグに関する情報が盛りだくさんで、デバッグ時の参考になるかと思います。
