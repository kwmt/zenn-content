---
title: "React Foundationsを呼んでわからなかったことのメモ"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React","Nextjs"]
published: false
---

## React hooksについて

https://nextjs.org/learn/react-foundations/updating-state
参考にメモする

Reactにはフックと呼ばれる関数群がある。フックを使えば、ステートのような追加ロジックをコンポーネントに追加できる。ステートとは、時間の経過とともに変化するUIの情報のことで、通常はユーザーとのインタラクションがトリガーになります。

stateを管理する関数として `useState` 関数がある。
以下のように、配列を返します。


```js
function HomePage() {
  // ...
  const [likes, setLikes] = React.useState();
 
  // ...
}
```
1番目の変数は、stateの値を表します。名前は何でも良いです。
2番目の変数は、stateを更新する関数を表します。prefixの`set`以外の名前は何でもよいです。

stateの値wの初期値を設定するには次のようにします。
```js
function HomePage() {
  // ...
  const [likes, setLikes] = React.useState(0);
}
```

これらを使う例としては以下のようになります。

```js
function HomePage() {
  // ...
  const [likes, setLikes] = React.useState(0);
 
  function handleClick() {
    setLikes(likes + 1);
  }
 
  return (
    <div>
      {/* ... */}
      <button onClick={handleClick}>Likes ({likes})</button>
    </div>
  );
}
```

![](https://storage.googleapis.com/zenn-user-upload/c9e5c595d4b6-20240519.gif)

## next.jsのどこに書けばいいのか？

たとえば上のコードをNext.jsのどこにかけばいいのか最初だとわからないのでメモっておく。

結論から言うと、`app/pages.tsx` に書けばいい。

```ts
import { useState } from 'react';
 
function Header({title} : {title:string}) {
  return <h1>{title ? title : 'Default title'}</h1>;
}
 
export default function HomePage() {
  const names = ['Ada Lovelace', 'Grace Hopper', 'Margaret Hamilton'];
 
  const [likes, setLikes] = useState(0);
 
  function handleClick() {
    setLikes(likes + 1);
  }
 
  return (
    <div>
      <Header title="Develop. Preview. Ship." />
      <ul>
        {names.map((name) => (
          <li key={name}>{name}</li>
        ))}
      </ul>
 
      <button onClick={handleClick}>Like ({likes})</button>
    </div>
  );
}
```

ただこれだと以下のエラーがでる。

```
Error:
  × You're importing a component that needs useState. It only works in a Client Component but none of its parents are marked with "use client", so they're Server Components by default.
  │ Learn more: https://nextjs.org/docs/getting-started/react-essentials
```

Next.jsはデフォルトでServer Componentsを使用します。これはアプリケーションのパフォーマンスを向上させるためで、Server Componentsを採用するために追加の手順を踏む必要はありません。

ブラウザのエラーを見ると、Next.jsは、サーバーコンポーネント内でuseStateを使おうとしていることを警告しています。インタラクティブな「Like」ボタンをクライアント コンポーネントに移動することで、これを修正できます。

LikeButtonコンポーネントをエクスポートする、like-button.jsという新しいファイルをappフォルダ内に作成します。

```ts:app/like-button.tsx
'use client';

import { useState } from 'react';

export default function LikeButton() {
    const [likes, setLikes] = useState(0);

    function handleClick() {
        setLikes(likes + 1);
    }
    return <button onClick={handleClick}>Like ({likes})</button>
}
```

```ts:app/page.tsx
import LikeButton from './like-button';
 
function Header({title} : {title:string}) {
  return <h1>{title ? title : 'Default title'}</h1>;
}
 
export default function HomePage() {
  const names = ['Ada Lovelace', 'Grace Hopper', 'Margaret Hamilton'];
 
  return (
    <div>
      <Header title="Develop. Preview. Ship." />
      <ul>
        {names.map((name) => (
          <li key={name}>{name}</li>
        ))}
      </ul>
 
    <LikeButton />
    </div>
  );
}
```

これでエラーなく表示できるようになった。