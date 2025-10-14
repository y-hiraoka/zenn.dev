---
title: "monorepoにESLintのFlat Configを導入した"
emoji: "✅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eslint"]
published: true
publication_name: chot
---

先日、業務で開発しているmonorepoのESLintをv9にアップデートして設定ファイルをFlat Configに変更しました。

自分自身はmonorepoにFlat Configを導入するのが初めてでした。色々調べた結果、学びがあったりきれいにまとまった設定ファイルが作れたと感じたので共有します。

Flat Config自体は目新しいものではなくなってきているので、細かい使い方は他の記事や公式ドキュメントを御覧ください。

## 設定ファイル完成形

`my-monorepo`は各自のプロジェクト名に、`some-workspace`は個別のワークスペース名に置き換えてください。

:::message

追記

本記事で紹介している `tseslint.config` は非推奨になりました。ESLint 本体が提供する `defineConfig` を使用してください。

https://typescript-eslint.io/packages/typescript-eslint/#config-deprecated
:::

```js :eslint.config.mjs
import globals from "globals";
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import eslintConfigPrettier from "eslint-config-prettier";
import eslintPluginReact from "eslint-plugin-react";
import eslintPluginReactHooks from "eslint-plugin-react-hooks";
import * as eslintPluginImport from "eslint-plugin-import";
import eslintPluginUnusedImports from "eslint-plugin-unused-imports";
import eslintPluginOnlyWarn from "eslint-plugin-only-warn";
import eslintPluginNext from "@next/eslint-plugin-next";
import eslintPluginTailwindCSS from "eslint-plugin-tailwindcss";

export default tseslint.config(
  {
    name: "my-monorepo/ignore-globally",
    ignores: [
      "**/node_modules",
      "**/.turbo",
      "**/dist",
      "**/.next",
      "**/temp",
      "server/src/__tests__/__generated__",
      "server/src/openapi",
      "some-workspace/src/lib/openapi/genereated-client",
    ],
  },
  {
    name: "my-monorepo/load-plugins",
    languageOptions: {
      ecmaVersion: 2022,
      sourceType: "module",
      globals: {
        ...globals.browser,
        ...globals.node,
      },
    },
    plugins: {
      import: eslintPluginImport,
      "unused-imports": eslintPluginUnusedImports,
      react: eslintPluginReact,
      "react-hooks": eslintPluginReactHooks,
      "@next/next": eslintPluginNext,
      "only-warn": eslintPluginOnlyWarn,
      tailwindcss: eslintPluginTailwindCSS,
    },
    settings: {
      react: {
        version: "detect",
      },
      next: {
        rootDir: "./*",
      },
    },
  },

  // 以下、ルールのon/off設定
  {
    name: "my-monorepo/global-tuning",
    extends: [eslint.configs.recommended],
    rules: {
      "import/order": "error",
      "unused-imports/no-unused-imports": "error",
      "unused-imports/no-unused-vars": [
        "error",
        {
          vars: "all",
          varsIgnorePattern: "^_",
          args: "after-used",
          argsIgnorePattern: "^_",
        },
      ],
    },
  },
  {
    name: "my-monorepo/for-typescript",
    files: ["**/*.ts", "**/*.tsx"],
    extends: [
      tseslint.configs.strict,
      eslintPluginReact.configs.flat.recommended,
      eslintPluginReact.configs.flat["jsx-runtime"],
    ],
    rules: {
      "@typescript-eslint/no-unused-vars": "off",
      "@typescript-eslint/no-namespace": "off",
      "react/prop-types": "off",
      ...eslintPluginReactHooks.configs.recommended.rules,
    },
  },
  {
    name: "my-monorepo/for-nextjs",
    files: [
      "some-workspace/**/*.{ts,tsx,js,jsx}",
      "some-workspace-2/**/*.{ts,tsx,js,jsx}",
      "some-workspace-3/**/*.{ts,tsx,js,jsx}",
    ],
    rules: {
      ...eslintPluginNext.configs.recommended.rules,
      ...eslintPluginNext.configs["core-web-vitals"].rules,
    },
  },
  {
    name: "my-monorepo/for-some-workspace",
    files: ["some-workspace/**/*.{ts,tsx,js,jsx}"],
    extends: [...eslintPluginTailwindCSS.configs["flat/recommended"]],
    settings: {
      tailwindcss: {
        callees: ["cn"],
        config: "some-workspace/tailwind.config.ts",
      },
    },
    rules: {
      "no-restricted-imports": [
        "error",
        {
          paths: [
            {
              name: "next/link",
              importNames: ["default"],
              message:
                "Please use NextLink from '@/components/next-link' instead.",
            },
          ],
        },
      ],
      "@next/next/no-img-element": "off",
    },
  },
  {
    name: "eslint-config-prettier",
    ...eslintConfigPrettier,
  },
);
```

## 説明

`tseslint.config`の引数に渡されるオブジェクトひとつひとつを指してConfiguration Objectと呼びます。

### 全体的な方針

monorepoにおけるESLint設定ファイルの配置方法には2通りあるかと思います。

- 各ワークスペース毎に`eslint.config.js`を配置する
- monorepoのルートに1つだけ`eslint.config.js`を配置する

このmonorepoではTurborepoを利用しているため、各ワークスペース毎に配置して`npm run lint`をワークスペース単位で実行することでTurborepoのキャッシュを活用する選択肢もありました。しかし今回はルートに1つだけ配置することにしました。理由としては以下が挙げられます。

- ESLint自体にもキャッシュ機能がある
- Flat Configでは明示的にディレクトリ(ワークスペース)を指定することが可能
- ルートに配置することでESLint設定を一元管理できる

Configuration Objectの記述順序については以下のような方針を取りました。

- ignoreの指定やpluginの読み込みは上にまとめる
- rulesの設定は下にまとめる
  - より広範囲に適用されるrulesは上の方に書く
  - ワークスペース固有のrulesは下の方に書く

外部パッケージのConfiguration Objectを読み込むことで、それに必要なプラグインが自動で読み込まれることもあります。しかし個人的にはどのプラグインを利用しているかを明示したいため、`eslint.config.js`の上の方でプラグインをまとめて読み込むようにしました。

適用するルールについては基本的に各種プラグインのrecommended

Configuration Objectには`name`を指定します。ESLint config inspectorが名前を表示してくれるため、独自のConfiguration Objectなのか外部パッケージ由来のものなのかが判別できます。

config inspectorはFlat Configでどのファイルにどのルールが適用されているかなどが確認できるツールです。以下のコマンドを実行するとローカルサーバーが起動してブラウザが立ち上がります。

```sh
npx eslint --inspect-config
```

### `tseslint.config`について

`tseslint.config`は`typescript-eslint`が提供してくれているFlat Config向けのヘルパー関数です。

https://typescript-eslint.io/packages/typescript-eslint#config

これを通すことで`// @ts-check`コメントによる型チェックが有効になったり、VSCodeの入力補完が効くようになります。

ほとんど可変長引数に渡した要素そのままのFlat Configを生成してくれるのですが、`extends`プロパティだけは`typescript-eslint`の独自プロパティです。これを使うことで、他のルールセットを利用しつつ一部のルールを追加・削除できます(旧来の設定ファイルにおける`extends`とは違い、モジュール名の文字列ではなくConfiguration Objectの変数を受け取ります)。

```js
{
  name: "my-monorepo/global-tuning",
  extends: [eslint.configs.recommended], // ESLintのrecommendedを継承
  rules: {
    "no-unused-vars": "off", // 特定のルールを削除
    "unused-imports/no-unused-imports": "error", // 特定のルールを追加
  },
}
```

### `eslint-plugin-next`について

Next.jsプロジェクトでESLintを使用する場合、`eslint-config-next`を使うことが一般的です。`eslint-config-next`を読み込めば`eslint-plugin-next`が勝手に読み込まれるようになっているため、`eslint-plugin-next`を直接インストールすることは多くありません。しかし、`eslint-config-next`はFlat Configに対応した形式になっていないため、`eslint-plugin-next`を直接使用することにしました。

`eslint-config-next`は`eslint-plugin-next`が提供するルールの適用だけでなく、Reactプロジェクトとして必要な多くのルールを適用してくれます。`eslint-config-next`を使わないため、`eslint-plugin-react`や`eslint-plugin-react-hooks`もまた自前で読み込む必要があることに注意が必要です。

### `eslint-config-prettier`について

`eslint-config-prettier`はPrettierと競合する可能性があるESLintのルールをオフにしてくれるパッケージです。Flat Configになっても最後に読み込ませることに変わりはありません。

`eslint-config-prettier`をスプレッド構文で展開しているのには理由があります。

```js
{
  name: "eslint-config-prettier",
  ...eslintConfigPrettier,
}
```

`eslint-config-prettier`は内部で`name`プロパティが指定されていませんでした。config inspectorで一覧する際、すべてのConfiguration Objectが`name`プロパティを持っていてほしいと感じたため、自分で`name`を明示的に指定しました。

`eslint-config-prettier`に`name`を追加するプルリクエストはすでに提出されているので、いずれこのような書き方は不要になると思います。

https://github.com/prettier/eslint-config-prettier/pull/294

### `eslint-plugin-tailwindcss`について

`eslint-plugin-tailwindcss`はTailwindCSSのクラス名をチェックするプラグインです。

どんな`tailwind.config.js`を使うかやどんなクラス名連結関数(`clsx`, `classNames`, `cn`,...)を使っているかはワークスペース毎に異なります。そのため、ワークスペース毎のConfiguration Objectで`settings`プロパティとともに読み込むことにしています。

### その他好みのプラグイン

#### `eslint-plugin-import`

import文周りのルールを提供してくれるプラグインです。僕は`import/order`だけほしいので導入しています。

#### `eslint-plugin-unused-imports`

未使用のimport文を自動で削除してくれます。ESLint本体の`no-unused-vars`はautofixを提供していないので、こちらを追加で導入しています。

#### `eslint-plugin-only-warn`

ESLintが検出したすべてのerrorをwarnレベルに変更してくれるプラグインです。これを導入することで「どのルールをerrorにしてどのルールをwarnにするか」という議論を避けることができます。

これを導入しているのはエラーを許容したいからではありません。ESLintはwarnであってもコードにひとつも残すべきではなく、warnが出るようなコードは修正するか`eslint-ignore`を明示的に残すべきと考えます。

このプラグインを利用しつつ`eslint --max-warnings=0`で実行することで、ひとつのwarnすらも許さない開発環境を構築できます。

## 感想

Flat ConfigによってESLintの設定ファイルの読みやすさが格段に向上したと感じています。

monorepoでもただ一つの`eslint.config.js`に全体のルールを集約することで、Flat Configのカスケードの仕組みを活かしつつもワークスペース固有のルールを設定しやすくなりました。

## まとめ

業務で開発しているmonorepoにESLintのFlat Configを導入しました。

- monorepoのルートにひとつだけ`eslint.config.js`を配置
- Configuration Objectの記述順序に方針を持つ
- `tseslint.config`を利用してTypeScriptの型チェックを有効にする
- 適用するルールは各種プラグインのrecommendedをベースに好みで調整

Flat ConfigによってESLintの設定ファイルの読みやすさが向上し、ワークスペース毎のルールを設定しやすくなりました。

Flat Configの登場によって各種サードパーティパッケージがFlat Configに対応していることが多いため、設定ファイルの記述量が減りました。

それでは良いESLintライフを！
