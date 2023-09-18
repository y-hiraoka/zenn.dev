---
title: "リポジトリで必要な環境変数のドキュメントを t3-env に任せる"
emoji: "🏭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["t3env"]
published: false
publication_name: chot
---

こんにちは、chot Inc. の Web エンジニアです。

chot Inc. は案件単位の開発を行っており、1 つの案件で数名のチームメンバーが開発しています。フェーズの切り替わりでメンバーが替わることもあるので、環境変数の共有が必要です。どこから取得するとか、何に使う値なのか、取りうる値はどんなものか、など。

そのコミュニケーションコストを下げるためか、`README.md` にリストアップしてみたり、notion にリストアップしてみたり、いろいろ工夫されていました。

しかし、ソースコードから独立したそれらのドキュメントでは当然メンテナンス漏れが発生します。これは t3-env で解決できるなと思い提案してみたところ、すぐに社のテンプレートリポジトリに導入されました。

t3-env の使い方はすでにいくつか記事があるのと、そもそも何も難しいものではないので、この記事では「信頼出来る唯一のドキュメント」として使えるという視点でお話します。

https://env.t3.gg/

## 変数に JSDoc で説明を与えられる

t3-env で宣言する環境変数オブジェクトはあくまで TypeScript コードです。`src/env.ts` ファイルに必要な環境変数とその制約を記述していまきます(ファイルパスは任意ですがこの記事では `src/env.ts` とします)。

そのプロパティである環境変数に JSDoc で説明を付与できます。

```tsx
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    /**
     * 〇〇サービスのAPIキー
     */
    SOME_API_KEY: z.string(),
  },
  client: {
    /**
     * リポジトリ一覧を取得するためのGitHubユーザー名
     * @default "vercel"
     */
    NEXT_PUBLIC_GITHUB_USER: z.string().default("vercel"),
  },
  runtimeEnv: {
    SOME_API_KEY: process.env.SOME_API_KEY,
    NEXT_PUBLIC_GITHUB_USER: process.env.NEXT_PUBLIC_GITHUB_USER,
  },
});
```

README や notion で変数の説明をする必要はなく、ただこの `src/env.ts` ファイルをみれば全て書いてあることを伝えればよいです。僕のプロジェクトでは、README に開発環境構築手順の 1 つとして `src/env.ts` へのリンクを設置して環境変数ファイルの用意を促しています。

## JSDoc が伝播する

`src/env.ts` に書いた説明は VSCode が拾ってくれて、使う時にも表示されるので、変数名で説明しきれない情報を日本語で参照できます。

![SOME_API_KEY というプロパティをサジェストしているインテリセンスに、JSDocで入力した説明文が表示されている例](/images/t3-env-as-documents/intelisense.png)

インテリセンスから選択する時に悩むことが減って嬉しいですね。

## バリデーションから取りうる値を明示できる

t3-env では zod を記述することで不足していたり意図していない値を指定された環境変数を検出します。そのバリデーションコードこそが取りうる値を明示するドキュメントになります。`.default()` があれば、入れなくても良いんだと判断することもできますね。

```tsx
export const env = createEnv({
  client: {
    NEXT_PUBLIC_VERCEL_ENV: z
      .enum(["development", "preview", "production"])
      .default("development"),
  },
  runtimeEnv: {
    NEXT_PUBLIC_VERCEL_ENV: process.env.NEXT_PUBLIC_VERCEL_ENV,
  },
});
```

README や notion では変数名と説明だけで、変数の取りうる値の列挙までしないかもしれません。zod バリデーションを書くとなれば、より厳密に変数に求められる値の制約を書く気になるでしょう。なってください(？)

## 後から参画するメンバーに正しい情報を提供できる

t3-env を導入しているプロジェクトでは、`process.env` を直接参照せず、バリデーションと型定義を済ませた `src/env.ts` ファイルから環境変数を使用します。

プロジェクトで必要な環境変数が増減した場合は必ず `src/env.ts` が変更されるということです。README や notion に列挙している場合は、使用箇所が増えてもメンテナンスされないかもしれません。t3-env を使えばコードとドキュメントが一致することになります。

ちなみに、t3-env を導入しているのに `process.env` を直接使用されてしまったら元も子もありません。その場合は `process.env` 自体を禁止にする eslint を設定すると良いです。

[https://github.com/eslint-community/eslint-plugin-n/blob/master/docs/rules/no-process-env.md](https://github.com/eslint-community/eslint-plugin-n/blob/master/docs/rules/no-process-env.md)

## デメリット

t3-env は zod でバリデーションを行う前提のライブラリです。zod も別の用途で使うつもりなら t3-env をインストールするのも抵抗が少ないでしょうが、そうではないならば t3-env のためだけに zod を追加でインストールする必要があります。最近よく問題視されてはいますが、zod はそこそこでかい上にツリーシェイクが効きません。環境変数チェックのためにバンドルサイズを膨らませることになります。

任意のバリデーションを挿せるようなインターフェイスに進化することを期待して待つしかありません。これについて、議論はされていたようです。止まってますけど…。

https://github.com/t3-oss/t3-env/issues/6

## まとめ

信頼出来るドキュメントになるという視点で t3-env をご紹介しました。zod を強制されるデメリットを理解した上で、みなさんが導入するきっかけになればと思います。

それでは良い TypeScript ライフを！
