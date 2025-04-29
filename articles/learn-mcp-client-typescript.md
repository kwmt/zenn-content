---
title: "MCP Quickstart For Client Developersを読む(TypeScript)"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MCP", "TypeScript"]
published: true
---

# はじめに
MCPドキュメントのQuickstartの For Client Developersを読みながら動かしたことをメモしておこうと思います。
https://modelcontextprotocol.io/quickstart/client

今回はTypeScriptでやっていきます。

# 読んでいく

## セットアップ
専用ディレクトリ作って、typescriptとmodelcontextprotocol/sdkが使えるようにセットアップしていきます。

```shell
$ mkdir mcp-client-typescript
$ cd mcp-client-typescript
$ npm init -y
package.json:

{
  "name": "mcp-client-typescript",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
$ npm install @anthropic-ai/sdk @modelcontextprotocol/sdk dotenv
npm WARN deprecated node-domexception@1.0.0: Use your platform's native DOMException instead

added 106 packages, and audited 107 packages in 3s

19 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

$ npm install -D @types/node typescript

added 3 packages, changed 2 packages, and audited 110 packages in 793ms

19 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

$ touch index.ts
```

あとコミットしていくので、gitignoreとかも設定しておきます。
gitignoreはこちらを拝借
https://github.com/microsoft/TypeScript/blob/main/.gitignore


package.jsonを更新

```shell
--- a/package.json
+++ b/package.json
@@ -3,7 +3,9 @@
   "version": "1.0.0",
   "description": "",
   "main": "index.js",
+  "type": "module",
   "scripts": {
+    "build": "tsc && chmod 755 build/index.js",
     "test": "echo \"Error: no test specified\" && exit 1"
   },
   "keywords": [],
```
Typescript Compiler(tsc)でコンパイルするために、tsconfig.jsonを作成します。

```json
{
    "compilerOptions": {
      "target": "ES2022",
      "module": "Node16",
      "moduleResolution": "Node16",
      "outDir": "./build",
      "rootDir": "./",
      "strict": true,
      "esModuleInterop": true,
      "skipLibCheck": true,
      "forceConsistentCasingInFileNames": true
    },
    "include": ["index.ts"],
    "exclude": ["node_modules"]
  }
```

ここでいったんビルド成功するか確認しておきます。

```shell
╰─ npm run build                                                                                                                                                                                                            ─╯

> mcp-client-typescript@1.0.0 build
> tsc && chmod 755 build/index.js
```
とくにエラーもなくビルドできたようです。

## Anthropic API Keyの設定

[Anthropic Console](https://console.anthropic.com/settings/keys)からAPI Keyを作って、`.env`ファイルに


```
ANTHROPIC_API_KEY=<your key here>
```
のように記述します。

## MCPクライアント実装

### Basic Client Structure

まず、modelcontextprotocol/sdkからClientをimportしてセットアップするコードをindex.tsに書きます。

```ts
import Anthropic from "@anthropic-ai/sdk";
import type { Tool } from "@anthropic-ai/sdk/src/resources/messages/messages.js";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import dotenv from "dotenv";

dotenv.config();

const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
if (!ANTHROPIC_API_KEY) {
  throw new Error("ANTHROPIC_API_KEY is not set in the environment variables.");
}

class MCPClient {
  private mcp: Client;
  private anthropic: Anthropic;
  private transport: StdioClientTransport;
  private tools: Tool[];

  constructor() {
    this.anthropic = new Anthropic({
      apiKey: ANTHROPIC_API_KEY,
    });
    this.mcp = new Client({
      name: "mcp-client-cli",
      version: "1.0.0",
    });
  }
}
```
Anthropic APIと連携するためのAPIクライアントである`Anthropic`をクラスと、
MCPクライアント `Client` を初期化します。

### Server Connection Management
1つのMCPサーバーと接続する部分を実装します。

```ts
  async connectToServer(serverScriptPath: string) {
    try {
      const isJs = serverScriptPath.endsWith(".js");
      const isPy = serverScriptPath.endsWith(".py");
      if (!isJs && !isPy) {
        throw new Error("Server script must be a .js or .py file.");
      }
      const command = isPy
        ? process.platform === "win32"
          ? "python"
          : "python3"
        : process.execPath;

      this.transport = new StdioClientTransport({
        command,
        args: [serverScriptPath],
      });
      // connect()が呼ばれると、クライアントは自動的にサーバーとの初期化フローを開始します。
      this.mcp.connect(this.transport);

      const toolResult = await this.mcp.listTools();
      this.tools = toolResult.tools.map((tool) => {
        return {
          name: tool.name,
          description: tool.description,
          input_schema: tool.inputSchema,
        };
      });
      console.log(
        "Connected to server wthith tools:",
        this.tools.map(({ name }) => name),
      );
    } catch (e) {
      console.error("Failed to connect to MCP Server:", e);
    }
  }
```

ここでのポイントは以下かなと思います。
- `transport` の初期化
  - `StdioClientTransport` の初期化に各言語ごとに必要な実行コマンドと、実行対象のスクリプトパスを渡しています。
- `mcp.connect`
  - mcp.connectでMCPクライントがサーバーと接続し初期化されます。
- `mcp.listTools`
  - mcp.listToolsでToolのリストを返してます。帰ってきた結果は後ほど使います。

### Query Processing Logic
クエリを処理する部分の実装

```ts
  async processQuery(query: string) {
    const messages: MessageParam[] = [
      {
        role: "user",
        content: query,
      },
    ];
    const response = await this.anthropic.messages.create({
      model: "claude-3-7-sonnet-20250219",
      max_tokens: 1000,
      messages,
      tools: this.tools,
    });

    const finalText = [];
    const toolResults = [];
    for (const content of response.content) {
      if (content.type === "text") {
        finalText.push(content.text);
      } else if (content.type === "tool_use") {
        const toolName = content.name;
        const toolArgs = content.input as { [x: string]: unknown } | undefined;

        const result = await this.mcp.callTool({
          name: toolName,
          arguments: toolArgs,
        });
        toolResults.push(result);
        finalText.push(
          `[Calling tool ${toolName} with args ${JSON.stringify(toolArgs)}]`,
        );
        messages.push({
          role: "user",
          content: result.content as string,
        });
        const response = await this.anthropic.messages.create({
          model: "claude-3-7-sonnet-20250219",
          max_tokens: 1000,
          messages,
        });
        finalText.push(
          response.content[0].type === "text" ? response.content[0].text : "",
        );
      }
    }
    return finalText.join("\n");
  }
```

ここでの気になりポイントは以下です。

- `this.anthropic.messages.create` 
  - [`Messages` API](https://docs.anthropic.com/en/api/messages)を利用
    - テキストや画像を送ることができる。
  - [`tools`引数](https://docs.anthropic.com/en/api/messages#body-tools)
    - mcpクライアントが利用可能なtoolを取得していた（tools)のでそれを渡す。
    - レスポンスに`tool_user`を返すこともある
- `this.mcp.callTool`
  - MCPサーバーのツールを実行します。



### Interactive Chat Interface
次に、チャットの入力待ち受けとcleanup処理を実装します。

```ts
  async chatLoop() {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    });
    try {
      console.log("\nMCP Client Started!");
      console.log("Type your queries or 'quit' to exit.");
      while (true) {
        const message = await rl.question("\nQuery: ");
        if (message.toLowerCase() === "quit") {
          break;
        }
        const response = await this.processQuery(message);
        console.log(`\n${response}`);
      }
    } finally {
      rl.close();
    }
  }
  async cleanup() {
    await this.mcp.close();
  }
```

ここはMCPクライアントという観点からは気になりポイントはないのでスルー。

### Main Entry Point
最後に、メインの実行ロジックを追加します。これはもちろん`MCPClient` クラス外に書く。
```ts
async function main() {
  if (process.argv.length < 3) {
    console.error("Usage: node index.js <path_to_server_script>");
    process.exit(1);
  }
  const mcpClinet = new MCPClient();
  try {
    await mcpClinet.connectToServer(process.argv[2]);
    await mcpClinet.chatLoop();
  } finally {
    await mcpClinet.cleanup();
    process.exit(0);
  }
}
main();
```

これもエントリポイントを書いただけなのでスルー。

## MCPクライントの実行

ビルドして
```shell
npm run build
```

クライアントを実行
```shell
node build/index.js path/to/build/index.js # node server
```


今回は、mcp serverのquick startにある[weathre-server-typescript](https://github.com/modelcontextprotocol/quickstart-resources/tree/main/weather-server-typescript)をクローンしてて、そのpathを指定します。

クローンしたら、
```shell
npm i && npm run build
```
でbuild/index.jsができるので、そのパスを指定してMCPクライアントを動かしてみます。


```shell
$ node build/index.js ~/quickstart-resources/weather-server-typescript/build/index.js                                                                             ─╯
(node:45490) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
Weather MCP Server running on stdio
Connected to server wthith tools: [ 'get-alerts', 'get-forecast' ]

MCP Client Started!
Type your queries or 'quit' to exit.

Query:

```
ひとまずMCPクライントの起動に成功したようです！


## MCPクライアントを使ってみる
MCPサーバーのことは天気予報を教えてくれるっぽいぐらいしかわかってないですが、とりあえず品川の天気でも聞いてみました。

```
Query: 品川の今日の天気を教えて

I notice you're asking about the weather in Shinagawa (品川) for today in Japanese. To provide you with the weather forecast, I need to use the weather tools available, but I'll need the latitude and longitude for Shinagawa, which is a district in Tokyo, Japan.

Let me ask you for that information: Could you please provide the latitude and longitude coordinates for Shinagawa, Tokyo? Once I have those coordinates, I can check the weather forecast for you.

Query: lat: 35.613069 lng: 139.73470
Error making NWS request: Error: HTTP error! status: 404

I see you've provided coordinates:
- Latitude: 35.613069
- Longitude: 139.73470

I can use these coordinates to get the weather forecast for this location. Let me do that for you right away.
[Calling tool get-forecast with args {"latitude":35.613069,"longitude":139.7347}]
I notice that the coordinates you've provided (35.613069, 139.73470) are located in Tokyo, Japan. The error message indicates that the National Weather Service (NWS) API only supports locations within the United States.

Unfortunately, I cannot provide weather data for this specific location in Tokyo using the NWS API. If you need weather information for this location in Japan, you would need to use a different weather service that provides international coverage.

Would you like me to suggest alternative ways to get weather information for this location, or would you prefer weather information for a US-based location instead?
```

NWS APIは東京の天気なんて知らないといわれた。あと日本語のほうが楽なので、日本語で回答できるか聞いてみたら日本語もOKでした。

```
Query: 日本語で回答できますか？

はい、日本語で回答することができます。天気予報や気象警報などについてのご質問がありましたら、お気軽にお尋ねください。

利用可能なツールとしては:
1. 州ごとの気象警報を取得する機能 (米国の州コードが必要です)
2. 特定の場所の天気予報を取得する機能 (緯度と経度の座標が必要です)

何かお手伝いできることがありましたら、お知らせください。

Query:
```

アメリカの天気じゃないとダメそうなので、ワシントンD.C.の天気予報を教えてもらいました。
```
Query: ワシントンD.C.の天気を教えて

ワシントンD.C.の天気予報をお調べします。天気予報を取得するためには、ワシントンD.C.の緯度と経度の情報が必要です。それでは、ワシントンD.C.の緯度と経度を使って天気予報を取得します。
[Calling tool get-forecast with args {"latitude":38.9072,"longitude":-77.0369}]
ワシントンD.C.の天気予報です：

今夜：
気温：8°C（47°F）
風：北西 21〜29 km/h（13〜18 mph）
ほぼ晴れ

日曜日：
気温：21°C（70°F）
風：北西 22〜32 km/h（14〜20 mph）
晴れ

日曜の夜：
気温：9°C（49°F）
風：北西 8〜24 km/h（5〜15 mph）
ほぼ晴れ

月曜日：
気温：24°C（75°F）
風：西 1〜8 km/h（1〜5 mph）
晴れ

月曜の夜：
気温：12°C（54°F）
風：南 約10 km/h（6 mph）
ほぼ晴れ

火曜日：
気温：29°C（84°F）
風：南 10〜16 km/h（6〜10 mph）
ほぼ晴れ

その後の週は気温が20°C台前半から29°C程度で推移し、後半は雷雨の可能性があります。
```
ちゃんと天気予報を返してくれました。


試しに天気とは関係ないことを聞いてみましたが、だめでした。
```
Query: MCPサーバーについて教えて

MCPサーバーについてのご質問ですね。申し訳ありませんが、私が利用できるツールの中には「MCPサーバー」に関する情報を直接提供できるものがありません。

現在利用可能なツールは、天気予報と気象警報に関するものだけです：
1. 特定の州の気象警報を取得する機能
2. 特定の緯度・経度に基づく天気予報を取得する機能

MCPサーバーについての詳細情報をお求めでしたら、もう少し具体的にお聞きしたいことをお教えいただけますか？また、もしMCPサーバーが特定の場所に関連するものであれば、その場所の天気情報をお調べすることは可能です。
```

## 補足

### tool_use まわり
`tool_use`で分岐しているところがいまいちわからなかったので「ワシントンD.C.の天気を教えて」と聞いた場合を例にしてもう少し詳細に調べてみました。

```ts
    const response = await this.anthropic.messages.create({
      model: "claude-3-7-sonnet-20250219",
      max_tokens: 1000,
      messages,
      tools: this.tools,
    });
```
最初のmessages.createのレスポンスが次のように返ってきていました。


```json
{
  id: 'msg_01J7u1yhfVmhkKgzSPLJ9jTX',
  type: 'message',
  role: 'assistant',
  model: 'claude-3-7-sonnet-20250219',
  content: [
    {
      type: 'text',
      text: 'ワシントンD.C.の天気予報をお調べします。天気予報を取得するためには、ワシントンD.C.の緯度と経度の情報が必要です。それでは、ワシントンD.C.の緯度と経度を使って天気予報を取得します。'
    },
    {
      type: 'tool_use',
      id: 'toolu_01K7yJLDHhC6g3sVfsMXUwaH',
      name: 'get-forecast',
      input: {"latitude":38.9072,"longitude":-77.0369}
    }
  ],
  stop_reason: 'tool_use',
  stop_sequence: null,
  usage: {
    input_tokens: 587,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
    output_tokens: 165
  }
}
```

`content`が配列になっていて、`type`が`text`の場合はまぁよくて、`type`が`tool_use`の場合は、使うToolとそのToolに渡すためのinput情報が返ってきました。
今回のだと`get-forecast`というツールはどうも緯度経度の情報が必要なので、inputとしてワシントンD.C.の緯度軽度が返ってきてました。

そして、MCPサーバーの 'get-forecast`ツールを実行し、

```ts
this.mcp.callTool('get-forecast', {"latitude":38.9072,"longitude":-77.0369})
```

以下のような感じでワシントンD.C．の天気予報が返ってくるので、

```json
{
  content: [
    {
      type: 'text',
      text: 'Forecast for 38.9072, -77.0369:\n' +
        '\n' +
        'Overnight:\n' +
        'Temperature: 47°F\n' +
        'Wind: 13 to 16 mph NW\n' +
        'Mostly Clear\n' +
        '---\n' +
        'Sunday:\n' +
        'Temperature: 70°F\n' +
        'Wind: 14 to 20 mph NW\n' +
        'Sunny\n' +
        '---\n' +
        'Sunday Night:\n' +
        'Temperature: 49°F\n' +
        'Wind: 5 to 15 mph NW\n' +
        'Mostly Clear\n' +
        '---\n' +
        'Monday:\n' +
        'Temperature: 75°F\n' +
        'Wind: 1 to 5 mph W\n' +
        'Sunny\n' +
        '---\n' +
        'Monday Night:\n' +
        'Temperature: 54°F\n' +
        'Wind: 6 mph S\n' +
        'Mostly Clear\n' +
        '---\n' +
        'Tuesday:\n' +
        'Temperature: 84°F\n' +
        'Wind: 6 to 10 mph S\n' +
        'Mostly Sunny\n' +
        '---\n' +
        'Tuesday Night:\n' +
        'Temperature: 66°F\n' +
        'Wind: 6 to 10 mph SW\n' +
        'Slight Chance Showers And Thunderstorms\n' +
        '---\n' +
        'Wednesday:\n' +
        'Temperature: 82°F\n' +
        'Wind: 8 mph W\n' +
        'Partly Sunny then Chance Rain Showers\n' +
        '---\n' +
        'Wednesday Night:\n' +
        'Temperature: 61°F\n' +
        'Wind: 2 to 6 mph NE\n' +
        'Chance Rain Showers\n' +
        '---\n' +
        'Thursday:\n' +
        'Temperature: 81°F\n' +
        'Wind: 2 to 8 mph E\n' +
        'Partly Sunny then Chance Showers And Thunderstorms\n' +
        '---\n' +
        'Thursday Night:\n' +
        'Temperature: 65°F\n' +
        'Wind: 8 mph SE\n' +
        'Chance Showers And Thunderstorms\n' +
        '---\n' +
        'Friday:\n' +
        'Temperature: 84°F\n' +
        'Wind: 6 to 12 mph SW\n' +
        'Showers And Thunderstorms Likely\n' +
        '---\n' +
        'Friday Night:\n' +
        'Temperature: 58°F\n' +
        'Wind: 9 mph W\n' +
        'Showers And Thunderstorms Likely\n' +
        '---\n' +
        'Saturday:\n' +
        'Temperature: 71°F\n' +
        'Wind: 7 to 10 mph NW\n' +
        'Slight Chance Rain Showers\n' +
        '---'
    }
  ]
}
```
これを使ってもう一度、Anthoropic API(messages.create)に投げて、自然言語でわかりやすく返してくれるという流れでした。

最終的には、「MCPクライアントを使ってみる」で示した通りの結果が出力されていました。

# おわりに

ざっくりMCPクライントの作り方がわかった気になれました。
今回の試したコードは以下のリポジトリに置いてますが、[公式](https://github.com/modelcontextprotocol/quickstart-resources/tree/main/mcp-client-typescript)のとほぼ同じですし、自分用のメモなので公式見たほうがいいです。

https://github.com/kwmt/mcp-client-typescript
