---
title: "Android Studioでライブラリモジュールの作り方"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


Android StudioのProjectツリーでproject rootを選択し、⌘+Nを押します。

![](https://storage.googleapis.com/zenn-user-upload/7f2b527fd12b-20240106.png)


`Module`を選択すると、`Create New Module`ダイアログが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/b5ccdf559c6e-20240106.png)

Templates[^1]で`Android Library`を選択し、右側で必要事項を入力します。

- Module name: 作りたい適切なモジュール名
- Package name: パッケージ名(自分は applicationId + module name とすることが多いです)
- その他はデフォルトで良いかと思います。

入力を終えてたら、Finishをクリックします。すると新しく作ったモジュールが作られます。


![](https://storage.googleapis.com/zenn-user-upload/4d5e5ec18b89-20240106.png)


[^1]: Templatesで私がよく使うテンプレートを紹介
    - `Android Library`
      - appモジュール(applicationモジュール)から使いたいモジュールを作りたいとき使用
    - `Phone & Tablet`
      - 同じプロジェクト内に別のアプリを作りたい場合に利用
      - たとえば、nowinandroidだと通常アプリとは別に[カタログアプリ](https://github.com/android/nowinandroid/tree/main/app-nia-catalog)で使われています。
    - `Benchmark`
      - [ベンチマーク](https://developer.android.com/topic/performance/benchmarking/benchmarking-overview?hl=ja)を行うときに利用


