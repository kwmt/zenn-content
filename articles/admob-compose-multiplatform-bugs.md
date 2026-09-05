---
title: "Compose MultiplatformでAdMobを表示するときに踏んだ4つの不具合"
emoji: "📺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AdMob", "ComposeMultiplatform", "KotlinMultiplatform", "iOS", "Android"]
published: false
---

# はじめに

個人で作っている猫育成ゲーム（KMP + Compose Multiplatform）に、AdMobのバナー広告とネイティブ広告を入れた。広告の表示自体は`AndroidView` / `UIKitView`でネイティブのViewを埋め込むだけなのだが、そのあと不具合を4つ踏んだのでめもっとく。

1. iOSで広告が出る瞬間に黒く光る
2. 画面遷移から戻るたびにバナーがリロードされる
3. 本番だけネイティブ広告が表示されない
4. MediaViewを使っていなくてポリシー違反になりかけた

# 1. iOSで広告が出る瞬間に黒く光る

## 症状

iOSでバナー広告とネイティブ広告が表示される瞬間、1フレームだけ黒く見える。

## 原因

Compose Multiplatformの`UIKitView`は、Swift側のUIViewがattachし終わるまでの1フレームの間、interop layer（CALayer）のデフォルト背景が黒として描画されることがあるらしい。

コンテナのUIViewに`backgroundColor`を設定していなかったので、`layoutSubviews`が終わるまでの間そのlayerが透けて黒く見えていた。

## 対策

`UIKitView`の`background`パラメータに背景色（このアプリはクリーム色）を渡す。これがCMPの公式APIとしての解決策になっている。あわせてSwift側のcontainerViewにも同じ色を設定した。

もう1つ、`UIKitView`はintrinsic sizeを持たないので、高さを`heightIn(max)`にしていると領域が確保されない。Inline Adaptiveバナーの最大高さで固定の`height`にして、先に領域を予約するようにした。

Android側も`AndroidView`で同じ構造だが、inflateが同期的なので再現しなかった。報告のスクショもステータスバーの形からiOSだったので、iOSだけ直している。

# 2. 画面遷移から戻るたびにバナーがリロードされる

## 症状

ホーム → コインショップなどの詳細画面に遷移して戻ると、画面下のバナー広告が毎回読み直される。表示が一瞬崩れるだけでなく、新しいimpressionが発生するので課金にも影響する。

## 原因

`Scaffold`（と`bottomBar`）をタブ画面の`MainShell`の中に置いていた。`MainShell`は`composable<MainShellRoute>`の配下にあるので、別のrouteにnavigateすると`MainShell`ごとcompositionから外れる。

戻ってくると`MainShell`が再composeされて、中の`remember { mutableStateOf(banner) }`や`remember { AdView(...) }`がリセットされる。結果、新しい`AdView`が作られて新しいimpressionがロードされる、という流れだった。

## 対策

`Scaffold`をNavHostの上に持ち上げた。`Scaffold`が`NavHost`を包む形にして、`bottomBar`の表示は`currentBackStackEntryAsState()`のdestinationがタブ画面かどうかで出し分ける。

あわせて、選択中のタブ（`rememberSaveable`）やバナーの設定などのstateもrootにhoistした。これで詳細画面に遷移して`MainShell`が破棄されても、`bottomBar`の`AdView`はそのまま生き残る。

「広告Viewの寿命 = Navigationのどこにぶら下がっているか」なので、広告を置く場所はNavHostの外、というのを覚えておきたい。

# 3. 本番だけネイティブ広告が表示されない

## 症状

stagingでは画面下の広告が表示されるのに、本番だと何も出ない。

## 原因

このアプリはバナーの設定（種類・Ad Unit ID・表示場所）を管理画面から出し分けられるようにしてある。画面下のバナーを描画するところで`bannerType`を見ずに、常にバナー広告のView（`BannerAdView`）で描画していた。

本番の設定はネイティブ広告（`admob_native`）でネイティブ広告用のAd Unit IDが入っているので、バナーとして読み込むとフォーマット不一致でロードに失敗して、何も表示されない。

stagingで気づけなかったのは、stagingは`ad_unit_id`が空でテスト用のバナーIDにフォールバックしていたから。つまりstagingでは常にバナーとして正しく表示されていて、本番の設定でしか壊れない状態だった。

## 対策

ホーム画面のほうは`bannerType`で分岐していたので、同じように分岐させて`ADMOB_NATIVE`のときは`NativeAdView`を使うようにした。

「テスト環境ではテストIDにフォールバックする」という親切な作りが、本番との差分を隠していた、というのが学びだった。

# 4. MediaViewを使っていなくてポリシー違反になりかけた

## 症状

AdMobのnative ad validatorが `MediaView not used for main image or video asset` を検出していた。

## 原因

ネイティブ広告のレイアウトで、左側の画像を普通の`ImageView` / `UIImageView`（アイコン扱い）で描画していて、必須の`MediaView`がなかった。AdMobのポリシー違反にあたるので、放っておくと配信制限の対象になりうる。

## 対策

- Android: 左の`ImageView`を`com.google.android.gms.ads.nativead.MediaView`に置き換えて、`setNativeAd`の**前**に`nativeAdView.mediaView`へ登録する
- iOS: 左の`UIImageView`を`MediaView`に置き換えて、`adView.mediaView`に登録し、`mediaView.mediaContent = nativeAd.mediaContent`を設定する

そのあと、MediaViewは120×120px以上が必要というのも引っかかったので、サイズも広げた。

自分で見て「広告が出ているからOK」と思っていても、ポリシー的にNGということがあるので、validatorの出力は見ておいたほうがいい。

<!-- TODO: validator の警告にどこで気づいたか（AdMob の管理画面？ログ？）を本人が加筆 -->

# おわりに

広告を出すこと自体はViewを埋め込むだけなのに、その周りで踏むものが多かった。特に2番の「画面遷移でリロードされる」は、動いているように見えるので気づきにくいし、impressionが増えるぶん数字にも影響する。広告を入れたら、まず遷移を往復してリロードされていないか見たほうがよさそう。

<!-- TODO: 実際に収益や表示回数にどう影響したか、言える範囲で本人が加筆 -->

# 参考

- https://developers.google.com/admob/android/native/advanced
- https://developers.google.com/admob/ios/native/advanced
- https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-swiftui-integration.html
