---
title: "Learn Next.js"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

https://nextjs.org/learn/dashboard-app を読む。

Next.js App Routerを学べるとのこと。


まずはsampleプロジェクトをダウンロードする

```shell
npx create-next-app@latest nextjs-dashboard --use-npm --example "https://github.com/vercel/next-learn/tree/main/dashboard/starter-example"
```


## プロジェクトの説明

### フォルダ構造


- `/app`：アプリケーションのすべてのルート、コンポーネント、ロジックを含みます。
  - `/app/lib`：再利用可能なユーティリティ関数やデータフェッチ関数など、アプリケーションで使われる関数が含まれます。
  - `/app/ui`：カード、テーブル、フォームなど、アプリケーションのすべてのUIコンポーネントが含まれています。時間を節約するために、これらのコンポーネントはあらかじめスタイル設定されています。
- `/public`：画像など、アプリケーションのすべての静的アセットが含まれます。
- `/scripts`：このスクリプトは後の章でデータベースにデータを投入するために使われます。
設定ファイル`next.config.js`のような設定ファイルもアプリケーションのルートにあります。これらのファイルのほとんどは、create-next-appを使用して新しいプロジェクトを開始するときに作成され、あらかじめ設定されています。このコースで修正する必要はありません。

### データについて

`/app/lib/placeholder-data.js` にモックデータが用意されている。
APIやdatabaseがまだなら便利だよという話。

### サーバー実行

初回やlibraryを追加した場合などに以下を実行する
```
npm i 
```

ローカルサーバーの実行は以下
```
npm run dev
```




## スタイリング
### Global CSS

`/app/ui/globals.css`

layout.tsxに次のように記述できる

```ts:layout.tsx
import '@/app/ui/globals.css';
```

@の記法は↓によるとmodule aliasというのかな。
https://nextjs.org/docs/app/building-your-application/configuring/absolute-imports-and-module-aliases

### Tailwind

TailwindはCSSフレームワークで、TSXマークアップに直接ユーティリティ・クラスを素早く記述できるようにすることで、開発プロセスをスピードアップします。


### CSS Module

CSSモジュールでは、一意のクラス名を自動的に作成することで、CSSをコンポーネントにスコープすることができます。

このコースでは引き続きTailwindを使用しますが、CSSモジュールを使用して上記のクイズと同じ結果を得る方法を少し見てみましょう。


### clsxライブラリを使ってクラス名を切り替える
状態やその他の条件に基づいて、要素に条件付きでスタイルを設定する必要がある場合があります。

clsxは、クラス名を簡単に切り替えることができるライブラリです。詳しくはドキュメントをご覧になることをお勧めしますが、基本的な使い方は以下の通りです：

ステータスを受け取るInvoiceStatusコンポーネントを作成したいとします。ステータスは 'pending' または 'paid' です。
「支払い済み」の場合、色は緑にします。'pending'の場合は灰色になります。
clsxを使って、このように条件付きでクラスを適用することができます：

```ts
import clsx from 'clsx';

    <span
      className={clsx(
        'inline-flex items-center rounded-full px-2 py-1 text-xs',
        {
          'bg-gray-100 text-gray-500': status === 'pending',
          'bg-green-500 text-white': status === 'paid',
        },
      )}
    >
      {status === 'pending' ? (
        <>
          Pending
          <ClockIcon className="ml-1 w-4 text-gray-500" />
        </>
      ) : null}
      {status === 'paid' ? (
        <>
          Paid
          <CheckIcon className="ml-1 w-4 text-white" />
        </>
      ) : null}
    </span>
  ```


### 画像について


`<Image>`コンポーネントは、HTMLの`<img>`タグを拡張したもので、以下のような自動画像最適化機能を備えている：

- 画像の読み込み時に自動的にレイアウトがずれるのを防ぎます。
- 画像をリサイズして、ビューポートの小さいデバイスに大きな画像を送らないようにする。
- デフォルトでの画像の遅延読み込み（画像はビューポートに入ると読み込まれます）。
- ブラウザがサポートしている場合、WebPやAVIFのような最新のフォーマットで画像を提供する。

```ts
import Image from 'next/image';
        <div className="flex items-center justify-center p-6 md:w-3/5 md:px-28 md:py-12">
          {/* Add Hero Images Here */}
          <Image
            src="/hero-desktop.png"
            width={1000}
            height={760}
            className="hidden md:block"
            alt="Screenshots of the dashboard project showing desktop version"
          />

          <Image
            src="/hero-mobile.png"
            width={500}
            height={620}
            className="md:hidden block"
            alt="Screenshots of the dashboard project showing desktop version"
          />
        </div>
```

またこれはレスポンシブ対応している。Tailwindの使い方だが、 `md` というのは `max-width: 768px;` という意味で、  `md:hidden` と書くと、画面幅が768px以上のときに要素を隠すという意味になるみたいです。

参考: 

https://tailwindcss.com/docs/container


つまり、2つのImageが次のようになっているということは、
```ts
<Image
  src="/hero-desktop.png"
  className="hidden md:block"
  />
<Image
  src="/hero-mobile.png"
  className="md:hidden block"
  />
```
- 画面幅が768px以上のとき、desktop.pngは表示される(block)がmobile.pngは非表示(hidden)になっているということ。
- 画面幅768px未満になったとき、destop.pngはhiddenでmobile.pngはblockとなり

レスポンシブ対応できるということのようだ。

## ルーティング
`page.tsx` は、Next.jsにとって特別なファイル。
`/app/page.tsx`は ルート/ に紐づきます。

たとえば、`/app/dashboard/page.tsx` ファイルを作ると、`/dashboard` にルーティングされます。


Next.jsで異なるページを作成する方法です。フォルダを使用して新しいルートセグメントを作成し、その中にページファイルを追加します。

ページファイルに特別な名前をつけることで、Next.jsはUIコンポーネント、テストファイル、その他の関連コードをルートと一緒に配置できます。
ページファイル内のコンテンツだけが一般公開されます。たとえば、/uiフォルダと/libフォルダは、ルートとともに/appフォルダ内に配置されます。

### dashboard pageのレイアウト作成する。
Next.jsでは、特別なlayout.tsxファイルを使用して、複数のページで共有されるUIを作成できます。

`<Layout />`コンポーネントはchildrenプロップを受け取ります。この子はページまたは別のレイアウトになります。
Next.jsでレイアウトを使用する利点の1つは、ナビゲーション時にページ コンポーネントだけが更新され、レイアウトは再レンダリングされないことです。これは部分レンダリングと呼ばれます。

### ページ間のナビゲーション

`<a>` タグだと、ページ全体がリフレッシュされてしまっている。

Next.jsでは`<Link>`コンポーネントというのがあり、HTMLの`<a>`タグを拡張した組み込みコンポーネントで、ルート間のプリフェッチとクライアントサイドナビゲーションを提供します。

https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#link-component


一般的なUIパターンは、現在どのページにいるのかをユーザーに示すためにアクティブリンクを表示することです。
これを行うには、URLからユーザーの現在のパスを取得する必要があります。
Next.jsは`usePathname()`というフックを提供しており、それを使ってパスをチェックし、このパターンを実装することができます。
`usePathname()`はフックなので、nav-links.tsxをクライアントコンポーネントにする必要があります。Reactの `"use client"`ディレクティブをファイルの先頭に追加し、next/navigationから`usePathname()`をインポートします。


## Supabase



### invoicesとcustomersテーブルをJOINするには

```ts
SELECT invoices.amount, customers.name, customers.image_url, customers.email, invoices.id
FROM invoices
JOIN customers ON invoices.customer_id = customers.id
```

こんなクエリを実現するには、以下のような感じ。

```ts
    import { QueryData } from '@supabase/supabase-js'
    const invoicesWithCustomersQuery = supabase.from('invoices').select(`
    id,
    amount,
    customers (
      id,
      name,
      image_url,
      email
    )
  `)
  type InvoicesWithCustomers = QueryData<typeof invoicesWithCustomersQuery>
  
  const { data, error } = await invoicesWithCustomersQuery
```

### orderするには

```ts
query.order('date', { ascending: false })
```

これで `order by date desc` と同じ。

https://supabase.com/docs/reference/javascript/order


### limitするには

```ts
query.limit(5)
```
https://supabase.com/docs/reference/javascript/limit

### Countするには

```
supabase().
    select('*', {count: 'exact', head: true})
    .from('hoge')
    .then((result) => {
        console.log(result.count)
    })
```

https://qiita.com/18sk/items/c7fd10a3cc12f652963f

### whereするには

```ts
const { data, error } = await supabase
  .from('countries')
  .select()
  .eq('name', 'Albania')
```
https://supabase.com/docs/reference/javascript/eq