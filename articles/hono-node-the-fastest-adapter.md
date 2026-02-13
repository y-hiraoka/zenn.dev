---
title: "Hono on Node.js 最速レスポンス選手権"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hono"]
published: true
publication_name: chot
---

## Intro

Hono は Web 標準の Request を受けて Response を返す Web フレームワークです。

Cloudflare Workers や Bun などの JavaScript ランタイムは、`(request: Request) => Response` 関数(以後 fetch ハンドラー)を渡してやれば、それがそのまま HTTP ハンドラーとして機能します。Hono はそれらのランタイムが要求するインターフェイスを満たしているため、簡単に Hono とランタイムを組み合わせることができます。

一方で Node.js は `node:http` モジュールの `createServer` 関数でサーバーを起動します。`createServer` 関数は `(req: IncomingMessage, res: ServerResponse) => void` 型の関数を要求し、`IncomingMessage` と `ServerResponse` は Web 標準の Request / Response とは異なる Node.js 独自のインターフェイスです。

これらのギャップを埋めるために、Hono には `@hono/node-server` が用意されています。

ところで、`@hono/node-server` の他にも Node.js 上で Web 標準の fetch ハンドラーを HTTP サーバーとして動作させるライブラリがいくつか存在します。Node.js でも Cloudflare Workers や Bun と同じようなインターフェイスで Web アプリケーションを開発したいというニーズが多いということですね。

それらのライブラリが Web 標準インターフェイスで HTTP サーバーを起動するということは、`@hono/node-server` 以外のライブラリでも Hono アプリケーションを Node.js 上で動作させることができるということです。

そこで今回は、Node.js 上で fetch ハンドラーを HTTP サーバーとして起動するライブラリを集めて、Hono アプリケーションを動作させたときのパフォーマンスをベンチマークしてみました。

## 出場者

### `@hono/node-server`

https://github.com/honojs/node-server

Hono 公式の Node.js アダプターです。 README によれば、Express.js の3.5倍高速とのことです。

### `@remix-run/node-fetch-server`

https://github.com/remix-run/remix/tree/main/packages/node-fetch-server#readme

Remix チームが提供している Node.js アダプターです。Remix も Web 標準で動作するフレームワークなので、Node.js 上で起動するにはアダプターが必要です。

ちなみに、フレームワークとしての React Router のほうは Remix コアメンバーの個人ライブラリである `@mjackson/node-fetch-server` のほうが使用されていました([利用箇所](https://github.com/remix-run/react-router/blob/18ae5d189e7ce529e00669ec797d7fa445274a33/packages/react-router-serve/cli.ts#L8C10-L8C31))。`@remix-run/node-fetch-server` は個人ライブラリを Remix リポジトリに取り込んだものと思われます。

### `srvx`

https://srvx.h3.dev/

[Nitro](https://nitro.build/) や [Nuxt](https://nuxt.com/) の内部で使われている [H3](https://h3.dev/) という Web フレームワークの、さらに内部で利用されるライブラリです。`export default { fetch }` というひとつのコードを、あらゆる JS ランタイムで HTTP サーバーとして起動できるようにするためのアダプターや CLI を提供しています。

### `@whatwg-node/server`

https://github.com/ardatan/whatwg-node#readme

`Request` / `Response` でサーバーアプリを記述し、Node.js を含むあらゆるランタイムやフレームワークで動作させることを目的とするライブラリです。Cloudflare Workers や Deno だけでなく、Express や Fastify などとの統合もサポートします。

## 最速レスポンス選手権(ベンチマーク)

ベンチマーク用に用意したリポジトリはこちらです。

https://github.com/stinbox/hono-node-the-fastest-adapter

### ベンチマーク方法

`GET /hello` と `POST /hello` エンドポイントを持つ Hono アプリケーションを用意します。リクエストボディの変換コストも計測できるように POST を含めています。

```ts
export const honoApp = new Hono()
  .get("/hello", (c) => c.text("Hello, Hono!"))
  .post("/hello", async (c) => {
    const name = (await c.req.json()).name;
    return c.text(`Hello, ${name}!`);
  });
```

それを各ライブラリで serve するためのエントリーファイルを用意します。

```ts: with-hono-node-server.ts
import { serve } from "@hono/node-server";
import { honoApp } from "./hono-app.ts";

serve({
  fetch: honoApp.fetch,
  port: 3000,
});
```

```ts: with-remix.ts
import * as http from "node:http";
import { createRequestListener } from "@remix-run/node-fetch-server";
import { honoApp } from "./hono-app.ts";

const adapter = createRequestListener(honoApp.fetch);
const server = http.createServer(adapter);

server.listen(3000);
```

```ts: with-srvx.ts
import { serve } from "srvx/node";
import { honoApp } from "./hono-app.ts";

serve({
  fetch: honoApp.fetch,
  port: 3000,
  silent: true,
});
```

```ts: with-whatwg-node.ts
import * as http from "node:http";
import { createServerAdapter } from "@whatwg-node/server";
import { honoApp } from "./hono-app.ts";

const adapter = createServerAdapter(honoApp.fetch);
const server = http.createServer(adapter);

server.listen(3000);
```

ベンチマーク測定には [bombardier](https://github.com/codesenberg/bombardier) を使用しました。各アダプターライブラリで同じ Hono アプリケーションを動作させ、10 秒間リクエストを送り続けたときのレスポンス数を計測しています。

```sh: scripts/bench-get.sh
#!/bin/bash

ADAPTERS=("with-hono-node-server" "with-remix" "with-srvx" "with-whatwg-node")

for adapter in "${ADAPTERS[@]}"; do
  echo "=== Benchmarking: $adapter ==="

  node "src/${adapter}.ts" &
  PID=$!

  # サーバー起動を待機
  sleep 1

  bombardier http://localhost:3000/hello

  kill $PID

  echo ""
done
```

```sh: scripts/bench-post.sh
#!/bin/bash

ADAPTERS=("with-hono-node-server" "with-remix" "with-srvx" "with-whatwg-node")

for adapter in "${ADAPTERS[@]}"; do
  echo "=== Benchmarking: $adapter ==="

  node "src/${adapter}.ts" &
  PID=$!

  # サーバー起動を待機
  sleep 1

  bombardier -m POST -H "Content-Type: application/json" -b '{"name":"Hono"}' http://localhost:3000/hello

  kill $PID

  echo ""
done
```

ベンチマークを実行するマシンスペックは以下の通りです。

```
Model Name: MacBook Pro
Model Identifier: Mac14,7
Chip: Apple M2
Total Number of Cores: 8 (4 performance and 4 efficiency)
Memory: 24 GB
```

Node.js のバージョンは v24.12.0 です。

### ベンチマーク結果

ベンチマークの結果を表にまとめると次のようになりました。単位は Req/Sec(1 秒あたりのリクエスト数) です。

#### GET リクエスト

| Rank | Adapter                      | Avg      | Stdev    | Max      |
| ---- | ---------------------------- | -------- | -------- | -------- |
| 🥇   | @hono/node-server            | 71596.23 | 13219.44 | 81973.02 |
| 🥈   | srvx                         | 56502.75 | 5402.22  | 62303.14 |
| 🥉   | @whatwg-node/server          | 52166.80 | 6027.07  | 62683.41 |
| 4️⃣   | @remix-run/node-fetch-server | 40126.88 | 7076.50  | 52596.60 |

#### POST リクエスト

| Rank | Adapter                      | Avg      | Stdev   | Max      |
| ---- | ---------------------------- | -------- | ------- | -------- |
| 🥇   | srvx                         | 46913.20 | 6676.71 | 57639.99 |
| 🥈   | @whatwg-node/server          | 44661.90 | 7089.05 | 52978.26 |
| 🥉   | @hono/node-server            | 32836.98 | 5926.09 | 39686.79 |
| 4️⃣   | @remix-run/node-fetch-server | 31284.20 | 6267.30 | 39326.90 |

:::details GET リクエストの完全なベンチマーク結果を見る

```
=== Benchmarking: with-hono-node-server ===
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     71596.23   13219.44   81973.02
  Latency        1.75ms     1.41ms   159.16ms
  HTTP codes:
    1xx - 0, 2xx - 715500, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:    16.58MB/s

=== Benchmarking: with-remix ===
scripts/bench-get.sh: line 5: 69434 Terminated: 15          node "src/${adapter}.ts"
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     40126.88    7076.50   52596.60
  Latency        3.11ms     3.77ms   302.68ms
  HTTP codes:
    1xx - 0, 2xx - 401392, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     9.87MB/s

=== Benchmarking: with-srvx ===
scripts/bench-get.sh: line 5: 69885 Terminated: 15          node "src/${adapter}.ts"
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     56502.75    5402.22   62303.14
  Latency        2.21ms     1.80ms   231.84ms
  HTTP codes:
    1xx - 0, 2xx - 565077, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:    13.90MB/s

=== Benchmarking: with-whatwg-node ===
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     52166.80    6027.07   62683.41
  Latency        2.40ms     2.07ms   250.42ms
  HTTP codes:
    1xx - 0, 2xx - 521711, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:    12.83MB/s
```

:::

:::details POST リクエストの完全なベンチマーク結果を見る

```
=== Benchmarking: with-hono-node-server ===
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     32836.98    5926.09   39686.79
  Latency        3.81ms     3.75ms   354.19ms
  HTTP codes:
    1xx - 0, 2xx - 327781, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     9.72MB/s

=== Benchmarking: with-remix ===
scripts/bench-post.sh: line 5: 74197 Terminated: 15          node "src/${adapter}.ts"
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     31284.20    6267.30   39326.90
  Latency        3.99ms     4.37ms   431.49ms
  HTTP codes:
    1xx - 0, 2xx - 312993, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     9.73MB/s

=== Benchmarking: with-srvx ===
scripts/bench-post.sh: line 5: 74643 Terminated: 15          node "src/${adapter}.ts"
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     46913.20    6676.71   57639.99
  Latency        2.67ms     2.32ms   263.00ms
  HTTP codes:
    1xx - 0, 2xx - 468445, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:    14.56MB/s

=== Benchmarking: with-whatwg-node ===
Bombarding http://localhost:3000/hello for 10s using 125 connection(s)
[==============================================================================================================================================================================================================================================================================================================================================================================================================================================] 10s
Done!
Statistics        Avg      Stdev        Max
  Reqs/sec     44661.90    7089.05   52978.26
  Latency        2.80ms     2.68ms   289.96ms
  HTTP codes:
    1xx - 0, 2xx - 446705, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:    13.88MB/s
```

:::

### 考察

GET リクエストについては、`@hono/node-server` が最も高いパフォーマンスでした。さすが高速を謳うだけありますね。`@hono/node-server` は高速化のための工夫が施されていることが開発者のブログにも紹介されていました。

https://zenn.dev/yusukebe/articles/7ac501716ae1f7

一方で POST リクエストについては、`srvx` が最も高いパフォーマンスを示しました。`@hono/node-server` は GET リクエストでは最速ですが、POST リクエストでは3位という結果になっています。これは、先程のブログで紹介されている工夫というのが、専ら GET リクエスト(リクエストボディに関心がないハンドラー)の速度改善を目的としているからでしょう。

## 選択肢が複数あること

Node.js で Web 標準インターフェイスを使った HTTP サーバーを起動するアダプターのパフォーマンス比較をしましたが、ベンチマークはベンチマークに過ぎません。CPU だけで返すレスポンス速度を比較したところで、Web アプリケーションのボトルネックはそこではないこと(外部 HTTP リクエスト、DB クエリなど)が多いですからね。

本記事で紹介したかったのは、`hono/node-server` 以外にも、Hono(や Web 標準インターフェイスによる HTTP サーバー)を Node.js 上で動作させる選択肢があるということです。これは Web 標準という共通インターフェイスを実装しているからこそ可能なことですね。

GET/POST ともに最下位だった `@remix-run/node-fetch-server` ですが、実はパフォーマンス以外の側面で優位と言える点があります。例えば以下のような `Request` を継承したクラスがあるとします。

```ts
class CustomRequest extends Request {
  constructor(request: Request) {
    super(request);
  }
}
```

標準の `Request` 同様に、別の `Request` をコピーしつつ新しいインスタンスを `new CustomRequest(request)` で生成できることが分かります。

Hono では `c.req.raw` プロパティに標準の `Request` オブジェクトが格納されています。これを使えば、`CustomRequest` を Hono のコンテキストから生成することができるはずです。

```ts
export const honoApp = new Hono().get("/custom-request", (c) => {
  new CustomRequest(c.req.raw);

  return c.text("Custom Request Works!");
});
```

しかし、上記の Hono アプリケーションは `@remix-run/node-fetch-server` 以外のアダプターでは `new CustomRequest(c.req.raw)` の行でエラーが throw されます。`@remix-run/node-fetch-server` だけが安定して `CustomRequest` でコピー可能な `Request` オブジェクトを生成しているということです。

このようなカスタム Request クラスをアプリケーションレイヤーで作ることは滅多にないですが、Web 標準インターフェイスによる接続を提供する別の Web フレームワークが内部で使っている可能性があります。下記は架空の認証フレームワーク `some-auth-framework` を Hono アプリケーションで使う例です。

```ts
import { Hono } from "hono";
import { Auth } from "some-auth-framework";

const auth = new Auth();

export const honoApp = new Hono()
  .get("/hello", (c) => c.text("Hello, Hono!"))
  .all("/auth/*", (c) => auth.handleRequest(c.req.raw));
```

`some-auth-framework` が `Request` を継承した独自クラスを使っている場合、`@remix-run/node-fetch-server` 以外のアダプターでは正しく動作しない可能性があります。これは架空の例ですが、こういった Web 標準経由の接続を提供するライブラリ/フレームワークは多いです。

また、`node:http` の `createServer` に渡すハンドラーを生成する以外の機能を提供しているライブラリもあります。

`@hono/node-server` はリクエスト元のIPアドレスなどを取得するユーティリティを `@hono/node-server/conninfo` で提供します。

`srvx` は逆のアダプターも提供しています。つまり、Node.js の `(req:IncomingMessage, res: ServerResponse) => void` 型のハンドラーを Web 標準の `(request: Request) => Response` 型の関数に変換するための `toFetchHandler` です。これによって、Express.js アプリケーションを Web 標準インターフェイスで動作させられるかもしれません(動作未確認)。

このように、パフォーマンス以外の側面でアダプターライブラリを選択することも重要です。

## まとめ

Node.js 上で Web 標準インターフェイスを使った HTTP サーバーを起動するアダプターを4つ紹介しました。また、それらのアダプターで Hono アプリケーションを動作させたときのパフォーマンスをベンチマークしました。

GET リクエストでは `@hono/node-server` が最速、POST リクエストでは `srvx` が最速という結果になりました。

Hono アプリケーションを Node.js 上で動作させる場合、`@hono/node-server` 以外にも選択肢があります。パフォーマンスだけでなく、アプリケーションに必要な機能を提供しているか、他のフレームワークとの組み合わせても安定するか、などを考慮して選定することが重要です。

それでは良い Hono ライフを！
