---
title: "【React】フレームワークなしで Static Generation する"
emoji: "🏭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "webpack"]
published: true
---

# Next.js が何をしているかを理解したかった

ふと自己紹介ページを作りたくなったので作って公開しました！

https://stin.ink

このサイトを作るに当たって、 Static Generation を Next.js なしでできないかなと考えました。ページは 1 枚でルーティングはないし、サーバーも要らないし、HTML 吐き出すだけならできるだろうと。

そして何より、Next.js 依存からいつでも脱却できる知識は備えておきたいと常々感じていました。

今回は最低限 Static Generation ができるために調べたことを書き残していきます。使用する React のバージョンは 18 です。

## `react-dom/server` と `react-dom/client`

create-react-app でプロジェクトを作ると、 `src/index.tsx` には次のようなコードが書かれています。

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

const root = ReactDOM.createRoot(document.getElementById("root") as HTMLElement);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

`react-dom/client` はブラウザ側で使用されること想定されているモジュールで、 `ReactDOM.createRoot` が空の div 要素 `document.getElementById('root')` に対して React アプリを構築する API です。「空の div 要素」ということで、 `React.createRoot` はブラウザ側で 1 から DOM を構築するいわゆる Client Side Rendering を行うために使用します。

一方、本記事の目的でもある Static Generation は予めコンポーネントの初期状態を HTML ファイルに書き込んでおくことを指します。そのためにはまずコンポーネントを HTML の文字列に変換する必要がありますが、それをやってくれるのが `react-dom/server` モジュールです。

```tsx
import ReactDOMServer from "react-dom/server";

const MyComponent = () => <div className="my-class">text</div>;

const html: string = ReactDOMServer.renderToString(<MyComponent />);

console.log(html); // <div class="my-class">text</div>
```

たったこれだけでコンポーネントが生成する HTML を string 型で受け取れるようになります。じゃあもうこれをブラウザからのリクエストに返すだけでいいんだ！…とは当然なりません。

もう少し React らしい例を試してみましょう。みんな大好き(？)カウンターアプリです。

```tsx
const App: FC = () => {
  const [count, setCount] = useState(0);
  const increment = useCallback(() => setCount((c) => c + 1), []);
  return (
    <div>
      <button onClick={increment}>increment</button>
      <p>count: {count}</p>
    </div>
  );
};

const html = ReactDOMServer.renderToString(<App />);

console.log(html); // <div><button>increment</button><p>count: <!-- -->0</p></div>
```

`useState` を使っています。 `increment` 関数を用意して、 `button` の `onClick` イベントに仕込んでいます。ですが、 `html` には `useState` が使われていることも `button` にイベントリスナーがセットされていることも情報として含まれていません。これをブラウザに送って読み込んでも、ブラウザは動かない `button` を描画するだけで **React アプリとして認識しない**のです。

`ReactDOMServer.renderToString` で生成した HTML を受け取ったブラウザには、それが React アプリであると認識してもらう必要があります。それを担当するのが `react-dom/client` に含まれている `hydrateRoot` です。これを使用することで React アプリとして読み込まれ、イベントリスナーが DOM にアタッチされていきます。この操作は hydrate/hydration と呼ばれます。

```tsx
import ReactDOMClient from "react-dom/client";
import { MyRoot } from "./component";

const rootElement = document.getElementById("react-root");

if (rootElement === null) throw new Error("rootElement was not found.");

ReactDOMClient.hydrateRoot(rootElement, <App />);
```

ここで `App` は先程の `ReactDOMServer.renderToString(<App />)` で使った `App` と同じものを指しています。また、その DOM は予め `<div id="react-root">` の子要素として HTML 化されているとします。

`ReactDOMServer.renderToString(<App />)` の結果の HTML を受け取ったブラウザは、このスクリプトも実行する必要があるということですね。

### まとめると

Static Generation をするためには、

- `react-dom/server` の `renderToString` でコンポーネントを HTML 文字列にして静的ファイルに書き込んで置いておく
- ブラウザは HTML を読み込んだらコンポーネントを含む div 要素に対して `react-dom/client` の `hydrateRoot` を実行する

この 2 つを達成すれば良さそうです。

## 作ってみる

### 環境構築

それでは 1 ページだけを Static Generation するための React 環境を構築していきましょう。

最終的なビルド結果は、 HTML, JavaScript, 画像など必要なファイルが build ディレクトリにまとまっている状態を期待します。 build ディレクトリ 1 つにまとまっていればあとは好きなホスティングサービスに渡してやれば配信することができると考えました。僕の場合は GitHub Pages を使っています。

コンポーネントから HTML を生成するのはビルド時に Node.js 上で行います。ソースコードは TypeScript で書くので、 ts-node を使います。

ブラウザに読み込んでもらう JavaScript の生成も行います。これは `react-dom/client` やページコンポーネント、そしてそれが import している無数のモジュールを含むので、バンドルする必要があります。この作業は定番の webpack にやってもらいましょう。

まずはプロジェクトディレクトリの作成。

```
mkdir react-static-generation
cd react-static-generation
npm init -y
```

dependencies のインストール。 meta タグを付けるために react-helmet も入れます。

```
npm i react react-dom react-helmet
```

TypeScript のための型定義ライブラリ。

```
npm i --save-dev @types/node @types/react @types/react-dom @types/react-helmet
```

ビルド時に使うツールたち。 copy-webpack-plugin は public ディレクトリの画像類を build にコピーする際に使います。 CSS Modules などを使う場合は css-loader などが追加で必要になります(自分は Chakra UI を使ったので不要でした)。

```
npm i --save-dev typescript ts-node webpack webpack-cli ts-loader copy-webpack-plugin npm-run-all
```

tsconfig.json はこんな感じです。

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "es2016",
    "jsx": "react-jsx",
    "module": "esnext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  },
  "ts-node": {
    "compilerOptions": {
      "module": "commonjs"
    }
  }
}
```

babel は使わず TypeScript コンパイラだけでブラウザで動くコードにするために、 `jsx: "react-jsx"` とします。 `module` については、 tree shaking を効かせるため webpack には `import`, `export` のまま渡ってほしいので `"esnext"`、 ts-node で実行する時は `"commonjs"` とします。あとはよくあるセットで。

webpack.config.js はこの通り。

```js:webpack.config.js
const CopyPlugin = require("copy-webpack-plugin");
const path = require("path");

/** @type {import("webpack").Configuration} */
module.exports = {
  mode: "production",
  entry: "./src/client.tsx",
  output: {
    path: path.resolve(__dirname, "build"),
    filename: "client.js",
  },
  resolve: {
    extensions: [".ts", ".tsx", ".js"],
  },
  module: {
    rules: [{ test: /\.tsx?$/, loader: "ts-loader" }],
  },
  plugins: [
    new CopyPlugin({
      patterns: [{ from: "public" }],
    }),
  ],
};
```

`entry` はあとで作るブラウザで読み込んでもらう `hydrateRoot` のスクリプトを指定します。 `output` は `build/client.js` が生成されるように設定。 `plugins` で `public` ディレクトリにあるファイルをごそっと `build` ディレクトリにコピーしてもらうための `CopyPlugin` を渡しています。

### ソースコード

ページに対応するコンポーネントを用意します。これはブラウザ用のスクリプトにもバンドルされるしビルド時にも必要なので、独立したモジュールとして用意します。~~何度でも登場するカウンターアプリです。~~

```tsx: src/page.tsx
import { FC, useCallback, useState } from "react";
import {Helmet} from "react-helmet"

export const Page: FC = () => {
  const [count, setCount] = useState(0);
  const increment = useCallback(() => setCount((c) => c + 1), []);
  return (
    <>
    <Helmet>
      <title>React Counter</title>
      <meta name="description" content="Static Generation のテスト" />
    </Helmet>
    <div>
      <button onClick={increment}>increment</button>
      <p>count: {count}</p>
    </div>
    </>
  );
};
```

react-helmet も使って `<title>` と `<meta>` が挿入できるか確認します。

そして、このコンポーネントを HTML に変換するスクリプトを書きます。

```tsx: src/generate.tsx
import fs from "fs";
import path from "path";
import ReactDOMServer from "react-dom/server";
import { Helmet } from "react-helmet";
import { Page } from "./page";

const pageString = ReactDOMServer.renderToString(<Page />);
const helmet = Helmet.renderStatic();

const html = `<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta name="viewport" content="width=device-width">
    ${helmet.title.toString()}
    ${helmet.meta.toString()}
  </head>
  <body>
    <div id="react-root">${pageString}</div>
  </body>
  <script src="client.js"></script>
</html>
`;

async function writeFile(file: string, data: string): Promise<void> {
  await fs.promises.mkdir(path.dirname(file), { recursive: true });
  fs.promises.writeFile(file, data);
}

writeFile(path.resolve(__dirname, "../build/index.html"), html);
```

ページコンポーネントを `ReactDOMServer.renderToString` で文字列に変換しています。この結果自体は対応する DOM のみを含んでいて完全な HTML ではありません。それをベースに完全な HTML を構築しています。そのとき、 `<div id="react-root">` で囲うのを忘れないようにします。これはブラウザ側でどの DOM 要素が React アプリなのかを把握するために必要です。また、 `react-dom/client` の `hydrateRoot` を実行するためのスクリプトである `client.js` が読み込まれるように `<script>` も挿入しておきます。最後に作った完全な HTML を build ディレクトリに書き込みます(build ディレクトリが存在しない場合でも失敗せずにファイルを生成できる `writeFile` 関数を用意しています)。

react-helmet については、 `Helmet.renderStatic` から取得するオブジェクトに `.title` や `.meta` など、要素別に含まれています。これらを必要な分だけ `<head>` に挿入しておきます。

続いてブラウザ側で hydrate を実行するスクリプトを書いていきます。

```tsx:src/client.tsx
import ReactDOMClient from "react-dom/client";
import { Page } from "./page";

const rootElement = document.getElementById("react-root");
if (rootElement === null) throw new Error("rootElement not found.");

ReactDOMClient.hydrateRoot(rootElement, <Page />);
```

`id="react-root"` な div 要素があるはずなのでそれを取得します。その div とページコンポーネントをセットで `ReactDOMClient.hydrateRoot` に渡します。これで DOM に対してイベントリスナーをアタッチし React アプリとして動くようになります。

### ビルドする

public ディレクトリに favicon.ico でも入れておいてください。 copy-webpack-plugin で public ディレクトリを指定していますが、ディレクトリ自体がないとエラーになってしまうためです。

package.json の `scripts` を編集します。 client.js のバンドルは webpack で、 HTML の生成は src/generate.tsx を ts-node で実行することでビルドします。それらを npm-run-all の `run-p` コマンドで並列実行します。

```json:package.json
  "scripts": {
    "build": "run-p build:*",
    "build:client": "webpack",
    "build:generate": "ts-node src/generate.tsx"
  },
```

これで `npm run build` を実行してみると、、、

![build ディレクトリに client.js, client.js.LICENSE.txt, favicon.ico, index.html が含まれている](https://storage.googleapis.com/zenn-user-upload/0f5cbeef3f23-20220417.png)

build ディレクトリに index.html と client.js が生成されました！ favicon.ico は public ディレクトリからのコピーになります。

build ディレクトリを配信することで React アプリが立ち上がるかを確認するためには、 serve をつかいましょう。

```
npx serve build
```

`localhost:3000` を開いてみると、、、

![Reactアプリが起動して increment ボタンをクリックすることでカウントアップしているgifムービー](https://storage.googleapis.com/zenn-user-upload/0de530152732-20220417.gif)

正常に React アプリが起動しました！あとは好きなホスティングサービスに build ディレクトリをデプロイするだけですね！

## まとめ

フレームワークを使わずに Static Generation をするために調べたことと、サンプルコードの紹介をしました。

Static Generation を実現するためには以下の 2 点が必要です。

- `react-dom/server` の `renderToString` でコンポーネントを文字列化する
- `react-dom/client` の `hydrateRoot` で DOM にイベントリスナーをアタッチする

じゃあサイトにページが増えたら？ルーティングは？開発サーバーは？Hot Module Replacement は？リクエスト毎にレンダリングは？ビルド時にリソース取得して HTML に埋め込むには？

...それではよい **Next.js** ライフを！
