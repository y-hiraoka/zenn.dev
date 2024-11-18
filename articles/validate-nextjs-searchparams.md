---
title: "Next.jsのsearchParamsはas stringせずに必ずバリデーションしてくれ。またはvalibotのちょいテクニック"
emoji: "😈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "valibot"]
published: true
publication_name: chot
---

## Next.jsのsearchParamsの型問題

Next.jsの`searchParams`の型は少々厄介です。[`searchParams`のドキュメント](https://nextjs.org/docs/app/api-reference/file-conventions/page#searchparams-optional)では次のように型定義が記載されています。

```tsx
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}) {
  const filters = (await searchParams).filters;
}
```

各パラメーターの型が`string | string[] | undefined`となっていますね。これを使うときに型チェックが面倒になって`searchParams.filters as string`と書いてしまっているのをよく見ます。`string[]`になるのは、次のように同じパラメーターキーを複数指定したURLの場合です。

```
/search?filters=foo&filters=bar
```

別の悪い例としては、そもそも`searchParams`が実際に届き得る値よりも狭い型定義をしてしまうことです。例えば次のようなコンポーネント定義です。

```tsx
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{
    q: string;
    sort: "asc" | "desc";
  }>;
}) {
  // ...
}
```

`sort`はまるで必ず`"asc"`か`"desc"`が届くかのような型定義になっていますが、実際はサイト訪問者のURL手入力により`sort`は`string`どころか`string[]`にも`undefined`にもなり得ます。しかし上記のコードでは取りうる値が型に反映されていないため、コード上は特に考慮しなくても型チェックは通ってしまうでしょう。`q`も同様に`string`だけでなく`string[] | undefined`の考慮ができていません。

`string`がくる前提の処理に`string[]`や`undefined`が届けば、高確率でランタイムエラーになることでしょう。それは500エラーとなり、エラーログが汚染され、エラーは無視される習慣になるかもしれません…。

`searchParams`の型をごまかすだけで、外からエラーを発生させることが可能になってしまいます。それを避けるためにも`searchParams`はランタイムでバリデーションすべきです。

ところで、パスパラメーターの`params`はランタイムチェックしなくてもセーフです。なぜなら`params`はページコンポーネントのファイル名から型が決まるため、`string`想定のパラメーターに`string[]`が入ってくる可能性がありません。例えば次のようにファイル名とパラメーターの型が対応します。

| File Path                            | Type                                          |
| ------------------------------------ | --------------------------------------------- |
| `app/shop/[slug]/page.js`            | `Promise<{ slug: string }>`                   |
| `app/shop/[category]/[item]/page.js` | `Promise<{ category: string, item: string }>` |
| `app/shop/[...slug]/page.js`         | `Promise<{ slug: string[] }>`                 |

とは言え、`params`も`string`型であること以上にくわしいフォーマットを期待する場合があるでしょう。自然数に変換可能な文字列を期待したり、リテラル型を期待したりなどです。その場合はやはりランタイムでバリデーションを行うべきです。

余談ですが、僕がNext.jsで作るときは次のような`next.d.ts`ファイルを作ることで、ページコンポーネントの型定義を少し楽にしています。

```tsx:next.d.ts
import "next";

declare module "next" {
  export type NextSegmentPage<
    Props extends {
      params?: Record<string, string | string[]>;
    } = object,
  > = React.FC<{
    params: Promise<Props["params"]>;
    searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
  }>;
}
```

`params`と`searchParams`が定義済みの`React.FC`を`NextSegmentPage`として定義します。それをあたかも`next`モジュールからexportされているように型定義しています。

使うときは次のように`params`の型だけを指定します。(本当はファイル名から勝手に指定されてほしいが無理なので…)

```tsx
import { NextSegmentPage } from "next";

const Page: NextSegmentPage<{ params: { slug: string } }> = async ({
  params,
  searchParams,
}) => {
  const slug: string = (await params).slug;
  const queryString: string | string[] | undefined = (await searchParams).q;
};

export default Page;
```

これでわざわざページコンポーネントを書くときに

```tsx
const Page: React.FC<{
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>;
}>;
```

と長々と書かなくても済むようになります。Next.js 15になって`Promise`にもなったので、より型定義を楽にする効果があります。

## valibotでバリデーションする

`searchParams`の型をごまかすことが危険なことはわかりました。型をごまかすのではなくランタイムでバリデーションをしましょう。

valibotを使ってくださいと言ってしまえば終わりなのですが、僕がよくやっている書き方を紹介します。

valibotは宣言的にバリデーションを書くことができるライブラリです。どんな値であるべきかを宣言するように書けるので、読みやすいバリデーションをサクッと実装できます。

zodでも他のバリデーションライブラリでもいいのですが、どれも未使用なら圧倒的にvalibotをおすすめします。バンドルサイズが小さくできるからです。`searchParams`の検証はサーバーサイドなのでバンドルサイズは関係ないですが、どうせそのうちクライアントサイドでもバリデーションすることになるのでね。書き心地もzodと比べて劣りません。快適です。

```bash
npm install valibot
```

### 基本方針

僕の`searchParams`バリデーションの方針は次の通りです。

- `string`を期待するパラメーターが`string[]`になっていたら失敗とする
- 変換する必要があるパラメーターはvalibotの`v.transform()`で変換まで行う
- パースには`v.safeParse()`/`v.safeParseAsync()`ではなく`v.parse()`/`v.parseAsync()`を使う

`v.parse()`/`v.safeParse()`はバリデーションを実行するメソッドで、違いはエラーをthrowするかResult型を返すかです。

### `string`を期待するパラメーターが`string[]`になっていたら失敗とする

前述の通り、`searchParams`は同じキーを複数指定することで`string[]`として渡すことができます。ただ、多くの`searchParams`は複数指定されることは期待しません。例えばURLに`sort`を2つ含む場合、それはサイトの訪問者がURLバーに手入力した可能性が高く、無視しても問題ないと考えます。

```ts
const QueryStringSchema = v.object({
  q: v.optional(v.string()),
});

// qが配列なので、エラーをthrowする
v.parse(QueryStringSchema, { q: ["foo", "bar"] });
```

上記コードは`q`が`string`か`undefined`であることを期待し、`string[]`の場合は検証失敗とします。もし単一の値を期待するパラメーターに複数指定されたとき、検証失敗ではなく先頭の値を採用する方針としたい場合は次のように`v.union()`と`v.transform()`を使ってスキーマを組み立てることができます。

```ts
const QueryStringSchema = v.object({
  q: v.optional(
    v.pipe(
      v.union([v.string(), v.array(v.string())]),
      v.transform((value) => (Array.isArray(value) ? value[0] : value)),
    ),
  ),
});

// qが配列なので、先頭の値を採用する
const validated = v.parse(QueryStringSchema, { q: ["foo", "bar"] });
```

あまり多くはないかも知れませんが、逆に`string[]`を期待するパラメーターは`string`を`string[]`に変換するような`v.transform()`を噛ませておくと、プログラム上は常に`string[]`として扱えて便利です。

```ts
const QueryStringSchema = v.object({
  q: v.optional(
    v.pipe(
      v.union([v.string(), v.array(v.string())]),
      v.transform((value) => (Array.isArray(value) ? value : [value])),
    ),
  ),
});

// validated.q は string[] 型
const validated = v.parse(QueryStringSchema, { q: ["foo", "bar"] });
```

### 変換する必要があるパラメーターはvalibotの`transform()`で変換まで行う

検索結果ページなどを実装する際は`page`に自然数フォーマットの`string`を期待するでしょう。その時、`v.parse()`したらすでに`number`になっていると嬉しいです。`v.transform()`で型も合わせておくと、プログラム上で別途変換しなくて良いです。

```ts
const PageSchema = v.object({
  page: v.pipe(
    v.string(),
    v.transform((v) => Number(v)),
    v.integer(),
    v.minValue(1),
  ),
});

// validated.page は number 型
const validated = v.parse(PageSchema, { page: "1" });
```

別の例として、圧縮した文字列を`searchParams`で扱っていることもあるかもしれません。`searchParams`を圧縮する手段については以前記事を書きましたので参考になれば幸いです。

https://zenn.dev/chot/articles/lz-string-vs-compression-stream

この記事にある`compressToEncodedURIComponent()`で圧縮された文字列が、`searchParams`に渡されてきたことを考えます。その文字列を解凍するには`decompressFromEncodedURIComponent()`を使いますが、これもスキーマの中で変換処理を噛ませてしまいます。

```ts
const CompressedCodeSchema = v.objectAsync({
  code: v.optionalAsync(
    v.pipeAsync(
      v.string(),
      v.transformAsync(async (value) => {
        try {
          return await decompressFromEncodedURIComponent(value);
        } catch {
          return undefined;
        }
      }),
    ),
  ),
});

const codeCompressed =
  await compressToEncodedURIComponent('const foo = "bar";');

const validated = await v.parseAsync(CompressedCodeSchema, {
  code: codeCompressed,
});
```

`compressToEncodedURIComponent()`は非同期処理なので、valibotのメソッドも非同期版を使用していることに注意してください。非同期処理を含むスキーマも同期版のメソッド名に`Async`を付けるだけで非常にわかりやすいのがvalibotの嬉しいポイントです。

### パースには`v.safeParse()`/`v.safeParseAsync()`ではなく`v.parse()`/`v.parseAsync()`を使う

`v.safeParse()`はその結果がいわゆる`Result`型となります。それはそれで嬉しいケースもありますが、`searchParams`の検証でいちいち成功可否の分岐を書くのは手間です。バリデーションに失敗してもさっさとデフォルト値で埋めて、処理を続行したい。

対して`v.parse()`は一発で検証済みの値を取得できますが、バリデーション違反があるとエラーをthrowします。`searchParams`を検証するだけでエラーを投げられるのは面倒ですが、必ず成功するスキーマを組み立てれば`try-catch`する必要もなくなります。必ず成功するスキーマを組み立てるには、`v.fallback()`を使用します。

```tsx
import { NextSegmentPage } from "next";
import * as v from "valibot";

const SearchParamsSchema = v.object({
  q: v.fallback(v.string(), ""),
  page: v.fallback(
    v.pipe(
      v.string(),
      v.transform((v) => Number(v)),
      v.integer(),
      v.minValue(1),
    ),
    1,
  ),
  sort: v.fallback(v.picklist(["asc", "desc"]), "asc"),
});

const Page: NextSegmentPage<{
  params: {
    slug: string;
  };
}> = async ({ searchParams }) => {
  const validatedSearchParmas = v.parse(SearchParamsSchema, await searchParams);

  validatedSearchParmas.q; // string
  validatedSearchParmas.page; // number
  validatedSearchParmas.sort; // "asc" | "desc"

  // ...
};

export default Page;
```

上記コードは`searchParams`の各キーに`v.fallback()`を使用しています。(ここまでの記事のまとめコードにもなっています)

上記コードは`v.parse()`によってエラーがthrowされることはありません。すべてのキーが`v.fallback()`を使用しており、バリデーション違反があってもデフォルト値で埋められるためです。また、各キーそれぞれで`v.fallback()`を使用しているので、バリデーションに成功した値はそれが使われ、失敗した値だけがフォールバック値で埋められます。

例えば`sort`は`"asc"`か`"desc"`を期待しますが、それ以外の値が指定された場合は無視して`"asc"`にフォールバックします。

また、`page`は`number`に変換した後さらに自然数かどうかのチェックもしていますが、それに失敗した場合も`1`にフォールバックします。小数やゼロ以下の数に変換できてしまうのはサイト訪問者が手入力している可能性が高いため、無視しても問題ないという判断です。

これで、`searchParams`をランタイムにバリデーションできるようになりました。エラーをthrowすることもないし、失敗したらフォールバックさせることもできるし、型安全でもあります。valibotは本当に便利ですね。

## まとめ

Next.jsの`searchParams`の型は実際に`string | string[] | undefined`になり得ます。`as`アサーションなどで型をごまかすのは避けてランタイムでバリデーションをしてください。

valibotを使うことで、バリデーションのスキーマを宣言的に記述できます。検証中に`v.transform()`で変換処理も挟むことができます。また、`v.fallback()`を適宜使用することで、`v.parse()`を使ってもエラーはthrowせずに一発で検証済みの値を取得できて便利です。

valibotでNext.jsの安全性を高めましょう。もちろん`searchParams`以外も、外界からの値のバリデーションにも使えます。

それでは良いvalibot/Next.jsライフを！
