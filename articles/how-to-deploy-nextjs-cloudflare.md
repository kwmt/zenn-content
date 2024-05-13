---
title: "Next.jsアプリケーションを新規作成して、Cloudflare Pagesにデプロイするまで"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "cloudflare pages"]
published: true
---

[C3 (create-cloudflare CLI)](https://developers.cloudflare.com/pages/get-started/c3/) というCLIを使って、next.jsプロジェクトを作成します。

以下のコマンドを実行する
```
npm create cloudflare@latest my-next-app -- --framework=next
```

`create-cloudflare@2.21.1` をインストールするかと聞かれるので、yを入力してEnter
```
Need to install the following packages:
create-cloudflare@2.21.1
Ok to proceed? (y) y
```

`create-next-app@14.1.0` をインストールするか聞かれるので、yを入力してEnter
```
> npx
> create-cloudflare my-next-app --framework=next


using create-cloudflare version 2.21.1

╭ Create an application with Cloudflare Step 1 of 3
│
├ In which directory do you want to create your application?
│ dir ./my-next-app
│
├ What type of application do you want to create?
│ type Website or web app
│
├ Which development framework do you want to use?
│ framework Next
│
├ Continue with Next via `npx create-next-app@14.1.0 my-next-app`
│

Need to install the following packages:
create-next-app@14.1.0
Ok to proceed? (y)
```

Typescriptを使いたいか聞かれるので、Yesを選択してEnter
```
? Would you like to use TypeScript? › No / Yes
```

ESLintを使いたいか聞かれるので、Yesを選択してEnter
```
? Would you like to use ESLint? › No / Yes
```

Tailwindを使いたいか聞かれるので、Yesを選択してEnter
```
? Would you like to use Tailwind CSS? › No / Yes
```

srcを作りたい聞かれるので、Yesを選択してEnter
```
? Would you like to use `src/` directory? › No / Yes
```

App Routerを使いたいか聞かれるので、Yesを選択してEnter
```
? Would you like to use App Router? (recommended) › No / Yes
```

インポートエイリアスを使いたいか聞かれるので、Yesを選択してEnter
```
? Would you like to customize the default import alias (@/*)? › No / Yes
```

import aliasの設定は何がいいか聞かれるのでデフォルトでよさそうなのでEnter
```
? What import alias would you like configured? › @/*
```


npm create を実行したパスに my-next-appプロジェクトが作成されました。
```
Creating a new Next.js app in ~/my-next-app.

Using npm.

Initializing project with template: app-tw


Installing dependencies:
- react
- react-dom
- next

Installing devDependencies:
- typescript
- @types/node
- @types/react
- @types/react-dom
- autoprefixer
- postcss
- tailwindcss
- eslint
- eslint-config-next


added 365 packages, and audited 366 packages in 32s

137 packages are looking for funding
  run `npm fund` for details

1 high severity vulnerability

To address all issues, run:
  npm audit fix --force

Run `npm audit` for details.
Initialized a git repository.

Success! Created my-next-app at ~/my-next-app

A new version of `create-next-app` is available!
You can update by running: npm i -g create-next-app

├ Created wrangler.toml file
│
├ Copying template files
│ files copied to project directory
│
╰ Application created
```


次にCloudflareの設定に進むみたいです。
```
╭ Configuring your application for Cloudflare Step 2 of 3
│
├ Installing wrangler A command line tool for building Cloudflare Workers
│ installed via `npm install wrangler --save-dev`
│
├ Installing @cloudflare/workers-types
│ installed via npm
│
├ Adding latest types to `tsconfig.json`
│ added @cloudflare/workers-types/2023-07-01
│
├ Retrieving current workerd compatibility date
│ compatibility date 2024-05-12
│
├ Created an env.d.ts file
```

next-on-pages eslint-pluginを使いたいか聞かれるのでYes
```
╰ Do you want to use the next-on-pages eslint-plugin?
  Yes / No
```

いろいろ更新やインストールされました。
```
├ Updating `next.config.mjs`
│ updated `next.config.mjs`
│
├ Updated the README file
│
├ Adding the Cloudflare Pages adapter
│ installed @cloudflare/next-on-pages@1, @cloudflare/workers-types, vercel, eslint-plugin-next-on-pages
│
├ Adding Wrangler files to the .gitignore file
│ updated .gitignore file
│
├ Updating `package.json` scripts
│ updated `package.json`
│
├ Committing new files
│ git commit
│
╰ Application configured
```

最後のステップっぽい。
Deployしたいか聞かれるが、あとでデプロイするのでNo
```
╭ Deploy with Cloudflare Step 3 of 3
│
╰ Do you want to deploy your application?
  Yes / No  
```


Git Integration

直接アップロードによるデプロイに加えて、Git 統合によるプロジェクトのデプロイも可能です。Git 統合では、GitHub または GitLab リポジトリを Pages アプリケーションに接続し、新しいコミットがプッシュされるたびに、Pages アプリケーションが自動的にビルドされ、デプロイされます。

Git Repositoryを作成して、プッシュします。

Cloudflareプロジェクトを作るときにしかGitHubとの連携設定ができないっぽい。

Workers & Pages > Create application > Pages > Connect to Git
でやったらうまくいった。


ローカルでPreviewするには

```
npm run preview
```
を実行する

ためしたリポジトリはこちら

https://github.com/kwmt/my-next-app