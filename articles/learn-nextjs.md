---
title: "Learn Next.js"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

https://nextjs.org/learn/dashboard-app を読む。

Next.js App Routerを学べるとのこと。


## Global CSS

/src/app/ui/globals.cssを作成し

```css:globals.css
@tailwind base;
@tailwind components;
@tailwind utilities;

input[type='number'] {
  -moz-appearance: textfield;
  appearance: textfield;
}

input[type='number']::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input[type='number']::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
```

layout.tsxに次のように記述できる

```ts:layout.tsx
import '@/app/ui/globals.css';
```

@の記法は↓によるとmodule aliasというのかな。
https://nextjs.org/docs/app/building-your-application/configuring/absolute-imports-and-module-aliases

## Tailwind

TailwindはCSSフレームワークで、TSXマークアップに直接ユーティリティ・クラスを素早く記述できるようにすることで、開発プロセスをスピードアップします。

