---
title: '巷で話題の新しい状態管理ライブラリ "unreduxed" を試す！'
emoji: "🆕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "redux"]
published: true
---

# unreduxed

リリースされたばかりの `unreduxed` が話題ですね！ `react-redux` と `unstated-next` に影響を受けた API になっており、非常にシンプルで扱いやすいライブラリになっているようです。

今回はそんな `unreduxed` の使い方を記していきます！

[リポジトリ](https://github.com/y-hiraoka/unreduxed)

## 特徴

- **シンプルな API**

非常にシンプルな API が特徴になっているそうです。react hooks と react context の理解さえあればすぐにライブラリを使用することができます。

API は `unstated-next` の `createContainer` をベースに、`react-redux` の `useSelector` の考えを取り込んでいます。

- **余分な再レンダリングを抑制する**

ベースになっている `unstated-next` は `unreduxed` 同様に hooks と context の理解だけでステート共有が可能なります。
しかし問題点として、context の機能をそのまま利用しているため、購読側コンポーネントの関心がないステートの変化まで検知して再レンダリングされてしまうことが挙げられます。
小さなアプリやステートの変化が少ないアプリでは `unstated-next` でグローバルステートを管理してもそれほど問題にはなりませんが、アプリの規模が大きくなっていくにつれてパフォーマンスが悪くなっていきます。
`redux` はアプリ全体でただひとつのステートを持ちますが、`react-redux` の `useSelector` がうまく値をキャッシュして余分な再レンダリングを抑制します。ただし学習コストが高いと言われています。

`unreduxed` は `unstated-next` の非常にシンプルな考えに `react-redux` の `useSelector` を持ち込みました。

## 使い方

何はともあれ、使ってみましょう！

### インストール

```
npm install unreduxed
```

### コンテナフックを作成する

`unreduxed` でいうコンテナフックとは複数コンポーネント間で共有したいステートを宣言するための、値を返却するカスタムフックのことのようです。

よくある数字をカウントするだけのカスタムフックで試していきます。

```tsx
import React from "react";
import unreduxed from "unreduxed";

const useCounter = (init?: number) => {
  const [count, setCount] = React.useState(init ?? 0);

  const increment = React.useCallback(() => setCount((prev) => prev + 1), []);
  const decrement = React.useCallback(() => setCount((prev) => prev - 1), []);

  return { count, increment, decrement };
};

export const [ContainerProvider, useContainer] = unreduxed(useCounter);

```

ライブラリから `default export` される関数にコンテナフックを突っ込むだけですね。
コンテナフックには `Provider` 経由で初期値を渡すことも可能です(ただし初期値を渡すことは任意なので `undefined` を考慮する必要があります)。ここでもコンテナフックの引数から初期値 `init` を受け取るように作ります。

### `ContainerProvider` を配置する

上で取得された `ContainerProvider` (変数名はなんでもいい) を共有したいコンポーネントツリーのトップに配置します。
`ContainerProvider` に囲われたコンポーネント (ここでは `<Count />` と `<Buttons />`) は `useContainer` が使用可能になります。
初期値は `initialState` という `props` で渡します。 `Provider` のネストももちろん可能です。

```tsx
export default function App() {
  return (
    <ContainerProvider>
      <Count />
      <Buttons />
      <ContainerProvider initialState={100}>
        <Count />
        <Buttons />
      </ContainerProvider>
    </ContainerProvider>
  );
}
```

これは `unstated-next` と同じ、というより `React.createContext` をベースにしているいろんなライブラリと同じ書き方なのでわかりやすいですね。`useContainer` が `React.useContext` のラッパーであることも想像がつきます。

### `useContainer` でコンテナから値を取り出す

`ContainerProvider` の内側のコンポーネントで `useContainer` を使用します。
`useContainer` には `selector` 関数が渡せて、値の絞り込みが可能になっています。これが `react-redux` の `useSelector` を持ち込んだと述べている点になりますね。

```tsx
const getRandomNum = () => Math.floor(Math.random() * 255);
const getColor = () => `rgb(${getRandomNum()},${getRandomNum()},${getRandomNum()})`;

const Count = () => {
  const count = useContainer((container) => container.count);

  const style = { color: getColor() };

  return <p style={style}>{count}</p>;
};

const Buttons = () => {
  const increment = useContainer((container) => container.increment);
  const decrement = useContainer((container) => container.decrement);

  const style = { color: getColor() };

  return (
    <div>
      <button onClick={increment} style={style}>
        increment
      </button>
      <button onClick={decrement} style={style}>
        decrement
      </button>
    </div>
  );
};
```

余分な再レンダリングを抑制するのが特徴とのことなので、再レンダリングを視覚的に確認できるようにランダムに文字色を変える動作を仕込んでいます。これについては [こちらのブログ記事](https://blog.ojisan.io/detect-rerender) を参考にしました。

### 動かしてみる

ここまでのソースコードを含んだデモアプリがこちらになります。

@[codesandbox](https://codesandbox.io/embed/unreduxed-demo-app-lgh16?fontsize=14&hidenavigation=1&theme=dark)

`Buttons` コンポーネント内で `useContainer` を使っているのに、各ボタンをクリックしてステートが変化しても `Buttons` 自体の色は変わりません！確かに余分な再レンダリングが抑制されているようです。

### モックコンテナを渡すこともできる

`ContainerProvider` は `mock` というコンテナと同じ型の `props` を受け取れるようになっています。 `mock` を渡すとコンテナフックは実行されなくなり、代わりに `mock` が配信されることになります。

```tsx
const MockProvider: React.FC = props => {
  const mock = {
    count: 500,
    increment: () => console.log("increment!"),
    decrement: () => console.log("decrement!"),
  }

  return (
    <ContainerProvider mock={mock}>
      {props.children}
    </ContainerProvider>
  )
}
```

これは Storybook など見た目を確認するツールで利用されることを想定されています。ロジックが停止されて固定された値が常に取得できれば見た目の確認のみに集中することができますね。

## まとめ

`unreduxed` を非常に簡単にグローバルなステートを宣言できるようになりますね。しかも各コンポーネントで使用したい値だけを取り出せば、余分な再レンダリングは抑制してくれるためパフォーマンスの心配もなさそうです。

よければみなさんもぜひ使ってみてください！

## 実は

記事の冒頭に「話題に」と言われていますがきっと読者の方は初めて耳にするライブラリだという人ばかりでしょう。
それもそのはずで、これは私が先日作成したステート管理ライブラリだからです。自分のライブラリの紹介記事でした。

そこでライブラリの中身について少し書き記していきます。

### 余分な再レンダリングを抑制する仕組み

context に愚直にステートを突っ込んで `useContext` で拾い上げる `unstated-next` パターンは、ステートが更新されたときに `useContext` しているコンポーネントすべてが再レンダリングされてしまいます。
`useContext` で再レンダリングされない条件は context の中身が同じ参照を持ち続けていることです。ただ、 `useContext` で取得するオブジェクトが不変だと購読側コンポーネントは変わってほしいときにも再レンダリングされなくなってしまいます。そこで思いついたのが、購読側コンポーネントが Provider から値を受け取るのではなく、購読側コンポーネントを再レンダリングさせる関数を Provider に集約することでした。

![unreduxedの内部イメージ](https://storage.googleapis.com/zenn-user-upload/uz7ywteatdwdo6ahtg689cn8eb4g)
_unreduxed の内部イメージ_

上の内部イメージにおいて、`notifier` はクラスインスタンスとして宣言されており、それを `Provider` 内部で `React.useRef` で保持しています。
`context` で配信するのは `notifier` インスタンスなので、参照が変更されることはありません。そのため、`useContext` による再レンダリングは発生しません。
`notifier` インスタンスは `useContainer` によって購読側コンポーネントまで下っていき、そこで購読側コンポーネントを再レンダリングするための listener 関数を集めます。
listener 関数ひとつひとつはコンテナを受け取り `selector` に渡して次の値を評価します。`useContainer` は `useRef` で前回値を保持しているので、次の値と比較して変更されていれば `ref` を更新して新しい値を返します。
`ContainerProvider` 内では、コンテナの変化を検知する `useEffect` によってすべての `listener` を発火します。

React hooks 時代にクラスインスタンスかよと思われるかもしれませんが、私もそう思うので API はまったくそれを感じさせないために `unstated-next` を踏襲した関数型のインターフェイスにしました(作ってから別にクラスインスタンスじゃなくてもできそうって思った)。ライブラリを使用する側は何も意識することなくただカスタムフックを組み合わせることでステートの共有が可能となります。学習コストもほぼゼロでしょう。

つらつらと日本語でしゃべるよりもプログラム言語で読んだほうが理解が早いと思うので興味のある方はこちらでご確認ください。

https://github.com/y-hiraoka/unreduxed/blob/master/src/createUseContainer.ts#L15-L35

### 今後について

React v18 で Concurrent Mode という機能が搭載されると言われています。React コンポーネントで非同期処理を扱いやすくするための機能です。Promise を throw すると React が吸収してなんかいい感じにしてくれるアレです(よくわかっていない)。
`unreduxed` はまったく Concurrent Mode を考慮していません。おとなしく、、、 [Recoil](https://recoiljs.org/) を、、、使いましょう、、、。

### 終わりに

`useContext` でも余計な再レンダリングしたくない！という一心でライブラリを作ってみました。
良ければ使っていただき、いいところ悪いところのフィードバックをいただけると幸いです。
特に内部実装の問題点があれば指摘していただけると助かります。
