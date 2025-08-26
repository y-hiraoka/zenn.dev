---
title: "CommonJS アプリケーションを ESM に移行する。ついでに Vitest にも移行する"
emoji: "🔀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "nodejs", "vitest"]
published: true
publication_name: chot
---

Node.js アプリケーションを CommonJS (CJS) から ES Modules (ESM) に移行したのでやったことを書き記します。

今回移行したアプリケーションは、バンドラーを伴わない純粋な Node.js アプリケーションです。TypeScript で書かれており、ビルド時には単に tsc で型を落としているだけです。ビルド成果物には `require` / `module.exports` が残っていて各ファイル同士が参照し合う、古典的なCJSアプリケーションとなります。

Node.js バージョンが 20 とやや古いので、残念ながら TypeScript のまま実行させること(`--experimental-strip-types`)はできません。よってこの記事では引き続き TypeScript を JavaScript に変換してから実行する場合の ESM 対応です。

## tsconfig.jsonのパスエイリアスを廃止する

ESM 化とは直接関係ないですが、import 文を絶対パスで書くために tsconfig.json に `paths` 指定がありました。

```json: tsconfig.json
{
   "compilerOptions": {
      "paths": {
        "#*": "./src/*.ts"
     }
  }
}
```

```ts
import { foo } from "#some/module/path";
```

この設定で得られるメリットは恐らく import 文から大量のドットがなくなって嬉しいとか、ファイル移動しても書き換えなくていいことだと思います。前者はどうせ import 文なんて見ないし、IDEが入力補完してくれるので我慢すれば良いでしょう。後者は型検査で要修正箇所が炙り出せます。

この設定のデメリットは、tsconfig.json と package.json と jest で同じ設定を何度も書く必要があることです。Next.js のようなバンドラーが tsconfig.json の `paths` を Single Source of Truth として面倒を見てくれるならいいのですが、バンドラーなしで自分で全て管理するのはかなり面倒です。

実際にこの設定を原因とした問題に直面しました。react-email によるメール送信のために `.tsx` ファイルを使い始めたとき、型エラーは出ないのにランタイムでモジュール解決できない現象に遭遇して時間を溶かしたことがあります。原因は、 tsconfig.json には `.ts` のパスしか書かれていなかったからです。VS Code は tsconfig.json を考慮して謎の import 文を自動挿入し、型エラーは出ませんでした。しかしその謎の import 文は Node.js にとっては存在しないパスだったのです。

### 対応

`paths` 設定によって絶対パス指定された import 文を一括で相対パスに置換する eslint plugin を作って紹介されている記事を思い出しました。

https://zenn.dev/mkizka/articles/a6148876fa46bd

これを使用することで、ほとんど一瞬で絶対パスをなくすことができました。僕の当該アプリケーションは monorepo の一部でしたが、ルートディレクトリを工夫することで正しく置換できました。

## ESM にする

ここから本題です。

ESM で実行されるということはビルド後の JavaScript ファイルにも import, export 文が残り、Node.js がそれを認識してモジュール解決を行うことになります。また、Node.js の ESM では import 文にファイル拡張子が必須になり、`index.js` という名前のファイルがディレクトリ代表のような特別扱いを受けなくなります。

ESM 化によって得られるメリットは、ESM オンリー npm パッケージを素直に読み込めることと、TLA(Top Level Await) が使えることです。

### package.json の `"type": "module"` 指定

Node.js のモジュール解決が CJS か ESM のどちらになるかは次のような条件で決まります。

- JavaScript ファイルの拡張子が `.cjs` である: CJS
- JavaScript ファイルの拡張子が `.mjs` である: ESM
- JavaScript ファイルの拡張子が `.js` であり、かつ
  - package.json の `type` フィールドが 未指定または`"commonjs"` である: CJS
  - package.json の `type` フィールドが `"module"` である: ESM

今後 `.js` ファイルを CJS で実行したい機運もないので、package.json に `"type": "module"` を指定して運用することにしました。もし特別な理由で一部のファイルを CJS 扱いしたい場合は、拡張子を `.cjs` (TypeScript の場合は `.cts`) にすれば良いです。

### tsconfig.json の指定方法と意味

`tsconfig.json` では compiler options として `module` と `moduleResolution` を選択できます。これらは名前の通りモジュール解決に関する設定を意味しています。

ESM 移行にあたり、これらの変更前と変更後は次のようになっています。

#### 変更前

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node"
  }
}
```

`"module": "commonjs"` は CJS を意味します。コンパイル時に `import` / `export` がそれぞれ `require` / `module.exports` に変換されるルールです。

`"moduleResolution": "node"` も CJS を意味します。`"module": "node"` のときは `"moduleResolution": "node"` しか選べないようになっています。

これらの設定によって、Node.js の `require` で到達するようなファイルの探し方を TypeScript もすることになります。例えば `./foo/index.ts` というファイルを `import "./foo"` (`require("./foo")`) で解決できて型チェックが可能なのは、これらの設定が寄与しています。

#### 変更後

```json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  }
}
```

`"module"`, `"moduleResolution"` ともに `"NodeNext"` を指定します。

これによって Node.js の ESM によるモジュール解決に TypeScript も追従することになります。

具体的には、import 文で `.js` 拡張子が必須となります。全て TypeScript で書かれたプロジェクトであっても、必ず `.js` を要求されます。

また、tsc によるトランスパイル後の JavaScript ファイルに `import`, `export` が残ります。ESM でモジュール解決する Node.js は `import`, `export` を解釈できるため、最適なトランスパイル結果となります。

### eslint pluginで `.js` を一括付与

さて、tsconfig.json を書き換えたことで import 文には `.js` 拡張子が要求されるようになりました。しかしソースコードは CJS の名残りで拡張子なしの import 文が大量に残っています。これをどうにか変更する必要があります。

この記事冒頭に紹介した tsconfig paths をやめる eslint plugin のように、import 文の拡張子を強制かつ自動 fix できる eslint plugin があるかもと考えました。実際にありました。

https://github.com/anza-xyz/eslint-plugin-require-extensions

ただ原因不明ですが、インストールして使うのには失敗しました(Flat Config との相性が悪いかも)。plugin のソースコードをコピーして少々手を入れた plugin を自作して `eslint --fix` を実行して一括拡張子付与に成功しました。

### `__dirname`, `__filename` を定義

`__dirname`, `__filename` は ESM で起動する Node.js ランタイムでは定義されていません。Node.js 20 からは代替の `import.meta.dirname`, `import.meta.filename` がありますが、v20 時点では Experimental API で stable になるのは v24 以降です。

ですので、使用箇所で毎回次のように変数宣言するようにしました。

```ts
import path from "path";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```

### libmime のエラー修正

libmime というパッケージをインストールして使っています。ESM に置き換えた後、コンパイルエラーは出ませんでしたがランタイムエラーとなりました。

原因は、`@types/libmime` の型定義が間違っていることでした。

libmime の実装では次のようにクラスとそのインスタンスが export されています(一部抜粋)。

```js
class Libmime { ... }
module.exports = new Libmime();
module.exports.Libmime = Libmime;
```

しかし `@types/libmime` では次のように定義されていました(一部抜粋)。

```ts
export function encodeWord(...): string;
export function encodeWords(...): string;
```

`encodeWord` などは実際は `Libmime` クラスのメソッドなのですが、型定義ではトップレベルにあるプレーンな関数のように定義されています。

`import { encodeWord } from "libmime";` のように import を書くと、CJS では動作しますが ESM では存在しない export でエラーになるのですね。

CJS で動作する理由は、tsc によるトランスパイル結果が次のようになり、たまたまメソッドアクセスになるからです。

```ts: トランスパイル前
import { encodeWord } from "libmime";

encodeWord("こんにちは");
```

```js: トランスパイル後
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
const libmime_1 = require("libmime");
(0, libmime_1.encodeWord)("こんにちは");
```

次のように default export を利用すれば ESM でも動作します。

```ts
import libmime from "libmime";

libmime.encodeWord("こんにちは");
```

### Jest を一生懸命直そうとするが断念

テストランナーとして Jest を使っていました。なんとか Jest のまま ESM に移行しようと思っていましたが、エラーが潰し切れずに諦めました。

具体的には、ファイルを指定したモック化を行っている箇所がどうしても直せませんでした。ESM 化することで Jest がモジュール解決できなくなっていたようです。

## Vitest 移行

Jest を諦めて ESM をサポートしている Vitest に同時に移行することを決めました。

Vitest は Jest よりも後発ライブラリで、Jest の互換性を売りにしています。そのおかげで、ほとんど機械的な修正のみで ESM コードのテストを実行できるようになりました。

CJS かつ Jest のプロジェクトを ESM 化する方は、同時に Vitest にしてしまうのをオススメします。テストコードの部分は機械的に置き換え可能で、少ない労力で対応可能です。

### Jest と異なる部分

僕のプロジェクトで Jest から Vitest に移行にあたり必要になった変更は以下の通りです。

#### テスト関数を import する

Vitest はデフォルトでは `test`, `describe`, `it` などの関数がグローバル変数として定義されていません。これらを Vitest パッケージから import する必要があります。

Jest を利用している場合は型定義パッケージとして `@types/jest` を利用しているはずです。これをアンインストールしてやれば、`test`, `describe`, `it` などが未定義変数の型エラーとして炙り出せます。

型エラーになったファイルで import 文を追加していきます。未使用の import 変数が自動で削除されるような eslint plugin を導入していれば、`vitest` から全てを import しているテキストをひたすら貼り付けて保存していくだけです。

```ts
import {
  beforeAll,
  afterAll,
  beforeEach,
  afterEach,
  test,
  describe,
  it,
  expect,
  vi,
} from "vitest";
```

#### global teardown の書き方

Jest にも Vitest にも、すべてのテストを実行する前に1度だけ実行される global setup と、すべてのテスト終了後に1度だけ実行される global teardown を定義できます。この２つを組み合わせて、テスト全体で共通のセットアップ処理とそれらをクリーンアップする処理を実行できます。僕のプロジェクトでは、[testcontainers](https://testcontainers.com/) を使ってテスト用のデータベースを立ち上げるのに利用しています。

Jest と Vitest では global teardown の書き方が少し異なります。

Jest では 別々のファイルを用意して config ファイルに指定します。

```js
const config = {
  globalSetup: "./tests/global-setup.ts",
  globalTeardown: "./tests/global-teardown.ts",
};

module.exports = config;
```

`global-setup.ts` と `global-teardown.ts` の例は次の通りです。ファイルを跨ぐため、グローバル変数を定義して `postgresContainer` の参照を引き渡す必要があることがわかります。

```ts: global-setup.ts
import {
  PostgreSqlContainer,
  StartedPostgreSqlContainer,
} from "@testcontainers/postgresql";

export default async () => {
  const postgresContainer = await new PostgreSqlContainer().start();
  process.env.DATABASE_URL = postgresContainer.getConnectionUri();
  globalThis.__POSTGRESQL_CONTAINER__ = postgresContainer;
};

declare global {
  // eslint-disable-next-line no-var
  var __POSTGRESQL_CONTAINER__: StartedPostgreSqlContainer;
}
```

```ts: global-teardown.ts
export default async () => {
  await globalThis.__POSTGRESQL_CONTAINER__.stop();
};
```

Vitestでは globalSetupファイルだけが定義できます。

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globalSetup: "./tests/global-setup.ts",
  },
});
```

global teardown に相当する処理は global setup のファイルに定義した関数の戻り値として定義します。ちょうど、React の `useEffect` がクリーンアップ関数を関数の戻り値で表現しているのと似ていますね。参照をグローバル変数で引き渡す必要がなく、Jest よりもスマートに感じます。

```ts: global-setup.ts
import { PostgreSqlContainer } from "@testcontainers/postgresql";

export default async () => {
  const postgresContainer = await new PostgreSqlContainer().start();
  process.env.DATABASE_URL = postgresContainer.getConnectionUri();

  return async () => {
   await postgresContainer.stop();
  };
};
```

#### viネームスペース

Jest では `jest.mock` のように `jest` のネームスペースからテストユーティリティ関数を呼び出せます。これは Vitest では `vi.mock` に相当します。単に `jest.` を `vi.` に置換するだけで済みます。

ただし `vi` 自体は `vitest` モジュールから import する必要があります。

```ts
import { vi } from "vitest";
```

## まとめ

<gpt>
Node.js アプリケーションを CommonJS（CJS）から ES Modules (ESM) へ移行し、テストランナーも Jest から Vitest へ変更した手順を解説しました。パスエイリアスや import 文の拡張子付与など、ESM 移行時に必要な設定変更やツールの活用方法を紹介しています。

ESM 化により、ESM 専用パッケージの利用や TLA（Top Level Await）が可能になりました。

`__dirname` や `__filename` の代替手法、型定義の不一致によるライブラリエラーの対処法も説明しています。

Jest から Vitest への移行では、テスト関数の import や global setup/teardown の書き方の違い、モック関数の呼び出し方法の変更など、主な修正点をまとめました。全体として、CJS から ESM、Jest から Vitest への移行は一部手作業を伴うものの、ツールやプラグインを活用することで効率的に進められることが分かりました。今後同様の移行を検討している方の参考になれば幸いです。
</gpt>

それでは良い Node.js ライフを！
