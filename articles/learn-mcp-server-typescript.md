---
title: "MCP Quickstart For Server Developersを読む(TypeScript)"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MCP", "TypeScript"]
published: true
---

# はじめに
MCPドキュメントのQuickstartの For Server Developersを読みながら動かしたことをメモしておこうと思います。
https://modelcontextprotocol.io/quickstart/client

今回はTypeScriptでやっていきます。

少しだけ[modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk/tree/main)の内部実装も見ていってるので、quickstartからさらに深掘りたいという方には興味持ってもらえるかもしれません。

# 読んでいく

## セットアップ

```shell
$ npm init -y
Wrote to ./package.json:

{
  "name": "mcp-server-typescript",
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
```

package.jsonに以下を追加

```json
{
  "type": "module",
  "bin": {
    "weather": "./build/index.js"
  },
  "scripts": {
    "build": "tsc && chmod 755 build/index.js"
  },
  "files": [
    "build"
  ],
}
```

```shell
# Initialize a new npm project
npm init -y

# Install dependencies
npm install @modelcontextprotocol/sdk zod
npm install -D @types/node typescript

# Create our files
mkdir src
touch src/index.ts
```



tsconfig.jsonを作成


```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}

```

ビルド確認。
```shell
─ npm run build
> mcp-client-typescript@1.0.0 build
> tsc && chmod 755 build/index.js
```


## Building your server

### Importing packages and setting up the instance
MCPサーバーインスタンスを生成


```ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const NWS_API_BASE = "https://api.weather.gov";
const USER_AGENT = "weather-app/1.0";

const server = new McpServer({
  name: "wweather",
  version: "1.0.0",
  capabilities: {
    resource: {},
    tools: {},
  },
});
```


### Helper functions

次に、National Weather Service API(NWS API)からデータを問い合わせ、フォーマットするためのヘルパー関数を追加します。

https://modelcontextprotocol.io/quickstart/server#helper-functions-2

このへんはNWS APIの仕様なので、たとえば `/alerts` APIなら
https://www.weather.gov/documentation/services-web-api#/default/alerts_query
このドキュメントをみてレスポンスをモデル化するだけです。


### Implementing tool execution

Toolを登録します。

:::message
Toolに関してはこちらを参照
https://modelcontextprotocol.io/docs/concepts/tools
:::

[ドキュメント](https://modelcontextprotocol.io/quickstart/server#implementing-tool-execution-2)の方には2つのtool実装がありますが、どちらもほぼ同じなので、わかりやすいget-forecast(天気予報)の部分だけのせます。



```ts
server.tool(
  "get-forecast",
  "Get weathre foreacast for a location",
  {
    latitude: z.number().min(-90).max(90).describe("Latitude of the location"),
    longitude: z
      .number()
      .min(-180)
      .max(180)
      .describe("Longitude of the location"),
  },
  async ({ latitude, longitude }) => {
    const pointsUrl = `${NWS_API_BASE}/points/${latitude.toFixed(4)},${longitude.toFixed(4)}`;
    const pointsData = await makeNWSRequest<PointsResponse>(pointsUrl);
    if (!pointsData) {
      return {
        content: [
          {
            type: "text",
            text: `Failed to retrieve grid point data for coordinates: ${latitude}, ${longitude}. This location may not be supported by the NWS API (only US locations are supported).`,
          },
        ],
      };
    }

    const forecastUrl = pointsData.properties.forecast;
    if (!forecastUrl) {
      return {
        content: [
          {
            type: "text",
            text: "Failed to get forecast URL from grid point data",
          },
        ],
      };
    }

    const forecastData = await makeNWSRequest<ForecastResponse>(forecastUrl);
    if (!forecastData) {
      return {
        content: [
          {
            type: "text",
            text: "Failed to retrieve forecast data",
          },
        ],
      };
    }

    const periods = forecastData.properties?.periods || [];
    if (periods.length === 0) {
      return {
        content: [
          {
            type: "text",
            text: "No forecast periods available",
          },
        ],
      };
    }

    const formattedForecast = periods.map((period) => {
      return [
        `${period.name || "Unknown"}:`,
        `Temperature: ${period.temperature || "Unknown"}°${period.temperatureUnit || "F"}`,
        `Wind: ${period.windSpeed || "Unknown"} ${period.windDirection || ""}`,
        `${period.shortForecast || "No forecast available"}`,
        "---",
      ].join("\n");
    });

    const forecastText = `Forecast for ${latitude}, ${longitude}:\n\n${formattedForecast.join("\n")}`;

    return {
      content: [
        {
          type: "text",
          text: forecastText,
        },
      ],
    };
  },
);
```

こちらのtoolの関数シグネチャと

https://github.com/modelcontextprotocol/typescript-sdk/blob/7e18c7016943af389b16bd670671c3527e3da21d/src/server/mcp.ts#L630-L635

Tool構造の仕様

https://modelcontextprotocol.io/docs/concepts/tools#tool-definition-structure

をみながら上のコードを見ていきます。


- `get-forecast` は `name` でuniqueである必要がありそうです。
- `"Get weathre foreacast for a location"`は、 `description` で、Human-readable descriptionとあるので、なんでも良さそうです。
- `{ latitude: z.number(), longitude: z.number() }` (省略していますが)の部分は、関数の方だと `paramsSchemaOrAnnotations` で、定義だと厳密には `inputScheme`ではないが、`inputScheme` に必要なパラメータと思って間違いなさそう。そして、[ここ](https://github.com/modelcontextprotocol/typescript-sdk/blob/7e18c7016943af389b16bd670671c3527e3da21d/src/server/mcp.ts#L683)で[ZodType](https://github.com/colinhacks/zod/blob/f210613edf99c322b015ff535f6b1388d17b59bd/src/types.ts#L169)かどうかチェックしているのでZodTypeである必要があります。
- `async ({ latitude, longitude }) => { return [{type: "text", text: forecastText}] }`（いろいろ省略）この部分は関数の `cb: ToolCallback<Args>`の部分で、`ToolCallback`の実装をみると

https://github.com/modelcontextprotocol/typescript-sdk/blob/7e18c7016943af389b16bd670671c3527e3da21d/src/server/mcp.ts#L907-L913


となっていて、（厳密ではないが）引数 `args` を受取り、 `CallToolResult` を返すという感じになっていました。get-forecastでの実装もそのようになっていて、引数`{latitude, longitude}`を受取り ` [{type: "text", text: forecastText}]`(を返しています。`CallToolResult`のコードを追っていくと、[TextContentScheme](https://github.com/modelcontextprotocol/typescript-sdk/blob/7e18c7016943af389b16bd670671c3527e3da21d/src/types.ts#L667-L675)にたどり着き、そのようになっていることがわかります。また、

https://github.com/modelcontextprotocol/typescript-sdk/blob/7e18c7016943af389b16bd670671c3527e3da21d/src/types.ts#L857-L862

こちらを見る限り、TextだけじゃなくImageやAudioなども返せそうでした。


### Running the server
最後にmain関数を実装します。
```ts
async function main() {
    const transport = new StdioServerTransport();
    await server.connect(transport);
    console.error("Weathre MCP Server running on stdio");
}

main().catch((error) => {
    console.error("Fatal error in main():", error);
    process.exit(1);
});
```

これで `npm run build`してMCPクライアントから動かせばOK

## 動かす

### MCP Inspectorで動作確認してみる
以下のコマンド実行して、
```shell
╰─ npx @modelcontextprotocol/inspector node build/index.js                                                                                                                                                                                                                                                                                                                                                                     ─╯
Starting MCP inspector...
⚙️ Proxy server listening on port 6277
🔍 MCP Inspector is up and running at http://127.0.0.1:6274 🚀
```

http://127.0.0.1:6274 にアクセスして確認してみてください。


![](https://storage.googleapis.com/zenn-user-upload/5f1c76a692ed-20250429.png)


### (自作の)MCP Clinetで動かしてみる

https://zenn.dev/yasi/articles/learn-mcp-client-typescript

こちらに書きましたが、自作のMCP Clinetで動かしてみるというのもありかもです。

自作のMCP Clientのディレクトリで以下のように実行するとよさそう。

```shell
node build/index.js ~/mcp-server-typescript/build/index.js
```

すると、いかのような感じで動いてることが確認できました。

```shell
(node:93486) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
Connected to server wthith tools: [ 'get-alerts', 'get-forecast' ]

MCP Client Started!
Type your queries or 'quit' to exit.

Query: NY
Response: {
  id: 'msg_01UpKNgxPgoST4wKjX9xd5is',
  type: 'message',
  role: 'assistant',
  model: 'claude-3-7-sonnet-20250219',
  content: [
    {
      type: 'text',
      text: "You're asking about New York, but I'd like to help you more specifically. Would you like to know about:\n" +
        '\n' +
        '1. Weather alerts for New York state\n' +
        '2. Weather forecast for a specific location in New York\n' +
        '\n' +
        "For weather alerts, I can provide current alerts for NY state. For a forecast, I would need specific latitude and longitude coordinates of a location in New York. Please let me know what information you're looking for."
    }
  ],
  stop_reason: 'end_turn',
  stop_sequence: null,
  usage: {
    input_tokens: 579,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
    output_tokens: 92
  }
}

You're asking about New York, but I'd like to help you more specifically. Would you like to know about:

1. Weather alerts for New York state
2. Weather forecast for a specific location in New York

For weather alerts, I can provide current alerts for NY state. For a forecast, I would need specific latitude and longitude coordinates of a location in New York. Please let me know what information you're looking for.

Query:
```


# おわりに

何番煎じかわかりませんが、自分で実際に読むのは他の記事よむだけよりものすごく理解しました。MCP Server実装の雰囲気がわかったので満足です。
今度はImageやAudioも返せることがわかったので、そのあたりを触ってみたいですね！
