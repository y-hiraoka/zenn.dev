---
title: "react-router をわずかでも型安全にしたい。 typed-path-builder のご紹介"
emoji: "🧷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react", "reactrouter"]
published: true
---

# typed-path-builder

## react-router について

`react-router` の型安全性は非常に弱いです。

例えば `<Link />` の `to` の指定。

```tsx
<Link to="/route/does/not/exist"> つらい😇 </Link>
```

文字列を直接指定するので存在しないリンク先を指定できてしまいます。

例えば `<Route />` の `path` 指定。

```tsx
<Route path="/path/does/not/exist"> かなしい🥺 </Route>
```

やっぱり文字列を直接指定するので設計にないパスを指定できてしまいます。

僕は TypeScript ユーザーなのでこれらに非常に悩まされてきました。

そこでパスを型安全に生成するライブラリを開発してみました。

## リポジトリ

https://github.com/y-hiraoka/typed-path-builder

## インストール

```
npm install typed-path-builder
```

## 使い方

### URL パスをオブジェクトで表現

- パスパラメータは `":"` で始める
- クエリパラメータは `"_queries"` プロパティの内部に

```ts
import createTypedPathBuilder from "typed-path-builder";

const routeConfig = {
  foo: {
    ":fooId": {},
  },
  bar: {
    ":barId": {
      hoge: {
        _queries: {
          param1: {},
          param2: {},
        },
      },
    },
  },
};

export const [path, route] = createTypedPathBuilder(routeConfig);
```

### path builder

`<Route />` の `path` props に渡すような文字列を生成します。

```ts
path.foo.fooId._build(); // => "/foo/:fooId"
path.bar.barId.hoge._build(); // => "/bar/:barId/hoge"
```

### route builder

`<Link />` の `to` props にわたすような文字列を生成します。

```ts
route.foo.fooId("001"); // => "/foo/001"
route.bar.barId("002").hoge._queries({ param1: "value1" }); // => "/bar/002/hoge?param1=value1
```

## 開発モチベーション

先日、TypeScript コミッターとして有名な [uhyoさん](https://twitter.com/uhyo_) がこんな記事を上げていました(`typed-path-builder` に関してはこの記事より前から構想を練っていました)。
https://zenn.dev/uhyo/articles/type-safe-routing-2021

僕自身も `react-router` の型安全性には日々頭を悩ませていました。ただ、散々アプリを作り込んだあとにルーティングライブラリを差し替えるのも不可能なので、せめてパス文字列だけは型安全に生成したいと思い開発に至りました。

## ライブラリの注目ポイント

1. path builder の `_build()` の戻り値もリテラル型で静的に決まっている

    ![path_build_type](https://storage.googleapis.com/zenn-user-upload/o151obd9agyuap47ikm5thwgso2u)

    型パズル頑張りました…。`_build()` の戻り値がリテラル型であるメリットとしては、 [typed-url-params](https://github.com/menduz/typed-url-params) と組み合わせることが可能であることです。(というか typed-url-params の存在を知ってからリテラル型を返すように修正した)

1. クエリパラメータも型で確定される

    ![route_queries_](https://storage.googleapis.com/zenn-user-upload/o4f5mqhq5yhr117wlzvbtf1wg7j7)

    型パズル頑張りました…(2回目)。存在しないプロパティを持ったオブジェクトを渡そうとするとコンパイルエラーになってくれます。

## ライブラリのだめなとこ

1. URLの終端を表すのが空オブジェクトがダサい

    `const config = { path1: {} }` ← 空オブジェクトで終わらないといけないのダサくないですか？(どうでもいい)
    
1. 1階層に2つ以上パスパラメータがある場合に使えない


    実は `typed-path-builder` が完成したあとに気づいてしまったのですが、 `react-router` が内部で使用している `path-to-regexp` は下記のようなパスパラメータの持ち方ができるみたいです。(全然知らなかった)

    ```tsx
    <Route path="/hoge/:fooId-:barId/fuga">
      <Page />
    </Route>
    ```

    `<Page />` で `useParams` を呼び出すと `{ fooId: string, barId: string }` のオブジェクトをちゃんと取得できます。

1. 再帰的な階層の表現ができない

    日本語では言い表しにくいのですが、要するに `react-router` のこの example のことです。
    
    https://reactrouter.com/web/example/recursive-paths

    固定のオブジェクトからルーティングを生成するため、再帰的なパスを生成できません。

1. その他 `react-router` で表現可能なルーティングを表現できない(かもしれない)

    全部は確認できませんがまだまだいっぱいありそう(つらい)。

## まとめ

以上、 typed-path-builder` というライブラリをご紹介しました。
react-router を意識してはいますが、あくまでパスを生成するためだけのライブラリなので他でも利用可能です。
[ライブラリのだめなとこ](#ライブラリのだめなとこ) でも列挙したとおり生成できるパスの自由度が低めではありますが、型安全に生成できる利点はありますので、ぜひ使用してフィードバック等いただけると幸いです。