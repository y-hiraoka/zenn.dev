---
title: "valibotでenvをタイプセーフに扱うvalibot-envを作った"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["valibot", "nextjs"]
published: true
publication_name: chot
---

こんにちは、エンジニアです。

みなさんはTypeScriptライブラリのvalibotを使っていますか？タイプセーフなバリデーションライブラリで、ソースコードの外界からやってくるあらゆる値のチェックに使えます。

同じくタイプセーフなバリデーションライブラリであるzodと比べて圧倒的にTree Shakingが効きやすい作りになっていて、特にバンドルを必要とするフロントエンド環境には強い味方です。

また、僕は最近社内ライブラリをよく作っているのですが、内部でバリデーションロジックを作る際にvalibotを使っています。これもTree Shakingの効きやすさが理由で、配布するライブラリを無駄に大きくしないことを重視しています。

以前はzodに比べて書き心地の観点ではあまり良くはないと感じていましたが、バージョン0.31.0で大幅に改善されて、zodに引けを取らない使いやすさになったと感じています(主観)。

ところで、以前t3-envが良いぞという内容の記事を書きました。

https://zenn.dev/chot/articles/t3-env-as-documents

これはprocess.envのような環境変数をバリデーションしつつタイプセーフにしてくれる便利なやつなのですが、zodとの組み合わせでしか使えません。

執筆当時、僕は当たり前のようにzodを使っていたのであまり気にしていなかったのですが、最近はvalibotを使うようになってきたので、zod依存必須がネックになっていました。

t3-envのREADMEでは、好きなバリデーションライブラリと組み合わせられるアップデートを予定していると宣言されているものの、なかなか来ません。

ということでvalibot版の環境変数バリデーションライブラリvalibot-envを自作しました。

https://github.com/y-hiraoka/valibot-env

ちなみに、この記事は会社のpublication投稿ですが、ライブラリは個人開発です。

## 使い方

### インストール

t3-envはコアAPIとNext.js用、Nuxt用にパッケージが分かれていますが、valibot-envでは分けていません。valibot-envと、peerDepsであるvalibotをインストールしてください。

```bash
npm install valibot-env valibot
```

### Next.js以外のプロジェクト

Next.js以外のプロジェクトでは、コアAPIを使います。

```tsx
import { createEnv } from "valibot-env";
import * as v from "valibot";

export const env = createEnv({
  publicPrefix: "PUBLIC_",
  schema: {
    public: {
      PUBLIC_SITE_URL: v.pipe(v.string(), v.url()),
      PUBLIC_SITE_NAME: v.string(),
    },
    private: {
      API_URL: v.pipe(v.string(), v.url()),
      API_KEY: v.string(),
    },
    shared: {
      NODE_ENV: v.union([v.literal("development"), v.literal("production")]),
      VERCEL_ENV: v.union([
        v.literal("development"),
        v.literal("preview"),
        v.literal("production"),
      ]),
    },
  },
  values: process.env,
});
```

`createEnv`には、環境変数を検証するスキーマ(`schema`)と実際の値(`values`)を指定します。スキーマには環境変数名とvalibotのスキーマの組を指定します。値の方は、Next.jsのような使用した環境変数だけがバンドルされるフレームワークではなければ、`process.env`を直接指定できます。

PrivateとPublicのランタイムを想定しています。主にサーバーとブラウザを識別することを目的としており、それは`isPrivate`によって区別されます。デフォルトでは`window`オブジェクトが存在するかどうかです(t3-envではServer/Clientですが、ServerはClientの変数を触れるので、Private/Publicのほうが命名として妥当だと思い、命名を踏襲していません)。

`publicPrefix`と`privatePrefix`に文字列リテラルを指定すれば、その文字列から始まる環境変数だけが型レベルで指定可能になります。環境変数名がプレフィックスを満たさない場合はTypeScriptがエラーを示してくれるでしょう。`NEXT_PUBLIC_`とか`VITE_`などが該当します。

`shared`については、プレフィックスがついていないがパブリックな値を想定しています。`NODE_ENV`や`VERCEL_ENV`などが該当します。

### Next.jsプロジェクト

Next.jsの場合、`publicPrefix: "NEXT_PUBLIC_"`を事前に指定済みの関数を用意しています。`"valibot-env/nextjs"`からimportしてください。

```tsx
import { createEnv } from "valibot-env/nextjs";
import * as v from "valibot";

export const env = createEnv({
  schema: {
    public: {
      NEXT_PUBLIC_SITE_URL: v.pipe(v.string(), v.url()),
      NEXT_PUBLIC_SITE_NAME: v.string(),
    },
    private: {
      API_URL: v.pipe(v.string(), v.url()),
      API_KEY: v.string(),
    },
    shared: {
      NODE_ENV: v.union([v.literal("development"), v.literal("production")]),
      VERCEL_ENV: v.union([
        v.literal("development"),
        v.literal("preview"),
        v.literal("production"),
      ]),
    },
  },
  values: {
    NEXT_PUBLIC_SITE_NAME: process.env.NEXT_PUBLIC_SITE_NAME,
    NEXT_PUBLIC_SITE_URL: process.env.NEXT_PUBLIC_SITE_URL,
    API_URL: process.env.API_URL,
    API_KEY: process.env.API_KEY,
    NODE_ENV: process.env.NODE_ENV,
    VERCEL_ENV: process.env.VERCEL_ENV,
  },
});
```

`publicPrefix`が固定されているだけでなく、`values`の型もコアAPIと異なっています。コアAPIの`values`は`unknown`型で`process.env`を直接指定できますが、Next.js用APIはちゃんと環境変数名を1つずつ指定しなければ型エラーになります(1つ1つはやはり`unknown`型です)。これは、Next.jsがアプリのソースコードで使用された`process.env`だけをバンドルに含めることが理由です。`process.env`直接指定ではどの変数も未使用として判定されるので、スキーマを通りません。よって1個ずつ明示的に指定してもらうために、`values`の型を変えています。

まだ用意していませんが、他のフロントエンドフレームワークでも同じような処理をするものがあるので、環境変数の個別指定必須な`createEnv`になるでしょう。他のフレームワーク用の追加PRお待ちしています(？)

### eslint を併用する

`env`オブジェクトをexportして、プロジェクト全体で`process.env`の代わりに使ってもらうことで効果を発揮します。`process.env`を直接使ってしまうと元も子もないので、eslintで`process.env`の使用を禁止することをおすすめします。

僕はよく`eslint-plugin-n/no-process-env`を使用しています。参考にしてみてください。

[https://github.com/eslint-community/eslint-plugin-n/blob/e5e758ea0cd238220127ae7bcbd967f1d8920f28/docs/rules/no-process-env.md](https://github.com/eslint-community/eslint-plugin-n/blob/e5e758ea0cd238220127ae7bcbd967f1d8920f28/docs/rules/no-process-env.md)

## 内部実装について

### モジュールのエクスポート

使用者側には、コアAPIとNext.js用APIで次のようにimportを使い分けてもらいます。

```tsx
// コアAPI
import { createEnv } from "valibot-env";

// Next.js用
import { createEnv } from "valibot-env/next";
```

これはpackage.jsonの`exoprts`フィールドで実現しています。

よくあるのは`main`フィールドでパッケージのエントリーポイントを指す方法ですが、TypeScriptがpackage.jsonの`exports`フィールドを参照して型を読めるようになった今、積極的に`exports`でエントリーポイントを指定可能です。`exports`はモジュールのエントリーポイントを複数指定できるので、コアAPIとは別にフレームワーク毎のAPIをサブパスでexportしています。`next`のようなフレームワーク自体には依存しておらずpeerDepsにすら書く必要がないので、ライブラリを分けずに`exports`で出し分けるほうが良いと考えました(一応フォールバックとして`main`も指定していますが)。

```json
{
  "name": "valibot-env",
  "main": "dist/index.js",
  "exports": {
    ".": "./dist/index.js",
    "./nextjs": "./dist/nextjs.js"
  }
}
```

TypeScriptは`exports`フィールドの中に`types`フィールドがない場合、`.js`を`.d.ts`に読み替えて型定義ファイルを探してくれるそうです。なので、ビルド結果の`dist`ディレクトリに`.js`と`.d.ts`が並ぶようにし、`exports`フィールドでは型定義ファイルの位置を明示していません。

### ビルドツール

ビルドにはtsupを使っています。ライブラリ向けにセットアップされたesbuildというイメージですね。

https://tsup.egoist.dev/

中身esbuildなのでバンドラーとしての役割も提供されていますが、僕はバンドルもミニファイもオフにしています。それらの最適化処理は、使う側にいるであろうフレームワークがやればいいからです。バンドルもミニファイもしないことで、使用者側が`node_modules`を直接覗き、デバッグに活かせることも期待します(ライブラリを疑って`node_modules`覗きに行くのってやりますよね？)。

### バージョン管理

CHANGELOG.mdの自動生成のためにchangesetsを導入してみました。

数百行の小さいライブラリなので宝の持ち腐れ感はありますが、練習がてら使ってみたく導入しました。非常に簡単に導入できたので今後も利用していきたいと思います。

一つだけ躓いた点としては、GitHub ActionsからNPMにリリースするワークフローを[changesets/action](https://github.com/changesets/action)のREADMEを見ながら作っていたのですが、`changeset publish`というコマンドがあることに気づかず、`npm publish`を実行してうまくいかないなぁとなっていました。ルートディレクトリのpackage.jsonをNPMにリリースしようとしてしまっていました…(`private:true`に救われた)。

https://github.com/changesets/action?tab=readme-ov-file#with-publishing

publishワークフローでは、ソースコードのビルドなどを実行後、最終的に`changeset publish`のコマンドが実行される必要があります。誰かが同じ轍を踏まないように、この情報が参考になれば幸いです。

## t3-envとの比較

t3-envとvalibot-env（というかzodとvalibot）のバンドルサイズ比較をしてみましょう。

Next.js Pages Routerで次のような2つのページを用意します。valibot-envを使った環境変数を表示するだけのページと、t3-envを使った環境変数を表示するだけのページです。どちらも同じ検証ロジックのスキーマを用意しています。

```tsx:pages/with-valibot.tsx
import { FC } from "react";
import { createEnv } from "valibot-env/nextjs";
import * as v from "valibot";

const env = createEnv({
  schema: {
    public: {
      NEXT_PUBLIC_SITE_URL: v.pipe(v.string(), v.url()),
      NEXT_PUBLIC_SITE_NAME: v.string(),
    },
    private: {
      DUMMY_API_URL: v.pipe(v.string(), v.url()),
      DUMMY_API_KEY: v.string(),
    },
    shared: {
      NODE_ENV: v.union([v.literal("development"), v.literal("production")]),
      VERCEL_ENV: v.union([
        v.literal("development"),
        v.literal("preview"),
        v.literal("production"),
      ]),
    },
  },
  values: {
    NEXT_PUBLIC_SITE_NAME: process.env.NEXT_PUBLIC_SITE_NAME,
    NEXT_PUBLIC_SITE_URL: process.env.NEXT_PUBLIC_SITE_URL,
    DUMMY_API_URL: process.env.DUMMY_API_URL,
    DUMMY_API_KEY: process.env.DUMMY_API_KEY,
    NODE_ENV: process.env.NODE_ENV,
    VERCEL_ENV: process.env.VERCEL_ENV,
  },
});

const Page: FC = () => {
  return <pre>{JSON.stringify(env, null, 2)}</pre>;
};

export default Page;
```

```tsx:pages/with-zod.tsx
import { FC } from "react";
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

const env = createEnv({
  client: {
    NEXT_PUBLIC_SITE_URL: z.string().url(),
    NEXT_PUBLIC_SITE_NAME: z.string(),
  },
  server: {
    DUMMY_API_URL: z.string().url(),
    DUMMY_API_KEY: z.string(),
  },
  shared: {
    NODE_ENV: z.union([z.literal("development"), z.literal("production")]),
    VERCEL_ENV: z.union([
      z.literal("development"),
      z.literal("preview"),
      z.literal("production"),
    ]),
  },
  runtimeEnv: {
    NEXT_PUBLIC_SITE_NAME: process.env.NEXT_PUBLIC_SITE_NAME,
    NEXT_PUBLIC_SITE_URL: process.env.NEXT_PUBLIC_SITE_URL,
    DUMMY_API_URL: process.env.DUMMY_API_URL,
    DUMMY_API_KEY: process.env.DUMMY_API_KEY,
    NODE_ENV: process.env.NODE_ENV,
    VERCEL_ENV: process.env.VERCEL_ENV,
  },
});

const Page: FC = () => {
  return <pre>{JSON.stringify(env, null, 2)}</pre>;
};

export default Page;
```

next-bundle-analyazerでzodとvalibotを含むJSチャンクのサイズを見てみましょう(next-bundle-analyzerのセットアップは省略)。

![valibot/distのバンドルチャンクサイズを表示している。Parsed size:3.52KB, Gzipped size:1.41KB](/images/abount-valibot-env/with-valibot-chunk-size.png)

![zod/libのバンドルチャンクサイズを表示している。Parsed size:55.78KB, Gzipped size:13.28KB](/images/abount-valibot-env/with-zod-chunk-size.png)

zodチャンクがgzip後で13.28KBに対し、valibotチャンクがgzip後で1.41KBです。zodに比べてvalibotが10分の1程度になっていることがわかります。valibotに対してTree Shakingが有効に機能して、小さくなっていることがわかります。

ところでビルドログ。

```bash
Route (pages)                             Size     First Load JS
┌ ○ /404                                  181 B          78.4 kB
├ ○ /with-valibot                         3.01 kB        81.2 kB
└ ○ /with-zod                             15.5 kB        93.7 kB
+ First Load JS shared by all             78.2 kB
  ├ chunks/framework-fbacec006b17dfec.js  45.2 kB
  ├ chunks/main-8db24bc05f412dc8.js       32 kB
  └ other shared chunks (total)           982 B
```

First Load JSを見ると、確かにzodとvalibotの差分くらいあります。

こういった小さい依存の積み重ねが大きなバンドルを作るので、zodとvalibotの差はやはり大きいです。フロントエンドにおけるTree Shakingの効きやすさは、重要なライブラリ選定基準ですね。

## まとめ

valibotでenvをいい感じにバリデーションするためのvalibot-envを作ったのでご紹介しました。

また、それを実装するために導入したツールや意図も紹介しました。

よかったらvalibot-envを使ってみてください。また、Next.js以外のフレームワークについては完全に門外漢なので、プルリクエストしていただけると嬉しいです。

それでは良いvalibotライフを！
