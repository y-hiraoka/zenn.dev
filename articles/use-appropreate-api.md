---
title: "そろそろ技術ブログで setCount(count + 1) と書くのはやめませんか"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

# 用意されている適切な API を使用しましょう

## 結論

こうではなく

```ts
const [count, setCount] = useState(0);

const increment = () => setCount(count + 1);
const decrement = () => setCount(count - 1);
```

これが正しい書き方です。

```ts
const [count, setCount] = useState(0);

const increment = () => setCount((prevCount) => prevCount + 1);
const decrement = () => setCount((prevCount) => prevCount - 1);
```

## setState の引数は 2 種類ある

- 次のステートを直接引数で受け取るインターフェイス

```ts
setState(newState);
```

- 直前のステートから新しいステートを計算する関数を引数で受け取るインターフェイス

```ts
setState((prev) => createNewStateFromPrevState(prev));
```

`useState` の紹介記事の題材でよくサンプルとして提示されるカウンターアプリの `increment` は「直前のカウントに 1 を足す関数」を意味しているはずです。
ですので、2 つ目の関数インターフェイスに当てはめて `setCount((prev) => prev + 1)` と書くのが適切でしょう。 `decrement` も同様です。

## 問題が発生するシチュエーション

「でも 1 つ目の書き方でも正しく動くじゃん」

それはそのサンプルがたまたま正しく動いているだけで、気づきにくいバグの可能性を含んでいます。

例えば、アプリのどこからでも使っていいユーティリティ関数として `useCounter` をモジュール化したとします:[^1]

[^1]: useCallback を使用すべき箇所ですが説明のために省略しています

```ts :useCounter.ts
export const useCounter = (init: number = 0) => {
  const [count, setCount] = useState(init);

  const increment = () => setCount(count + 1);
  const decrement = () => setCount(count - 1);

  return { count, increment, decrement };
};
```

`increment` や `decrement` が引数を受け取らないため差分 1 ずつしか操作できないことに気づいた `useCounter` の使用者が、応用して 2 ずつ変化させて使おうと考えます:

```tsx :CounterDouble.tsx
import { useCounter } from "./useCounter";

export default function App() {
  const { count, increment, decrement } = useCounter(10);

  const incrementDouble = () => {
    increment();
    increment();
  };

  const decrementDouble = () => {
    decrement();
    decrement();
  };

  return (
    <div className="App">
      <h1>Count: {count}</h1>
      <button onClick={incrementDouble}>2 増やす</button>
      <button onClick={decrementDouble}>2 減らす</button>
    </div>
  );
}
```

CodeSandbox で動作確認してみましょう。

@[codesandbox](https://codesandbox.io/embed/for-zenn-usecounter-bad-example-wytrp?fontsize=14&hidenavigation=1&theme=dark)

差分 2 ずつ変化させるはずが 1 ずつしか変化しません。
ボタンが 1 度クリックされたら `increment` (or `decrement`) が 2 回ずつ呼ばれているので 2 変化するはずです。なぜでしょうか？

経験豊富な人や勘のいい人は既にお気づきでしょう。 `setCount` に同じ値を繰り返し渡しているだけだからです。

## `count` はあくまで定数であって変化する値ではない

下記の `App` コンポーネントを例示します。

```tsx
export default function App() {
  const [count, setCount] = useState(0);
  const increment = () => setCount(count + 1);
  const decrement = () => setCount(count - 1);

  return (
    <div className="App">
      <h1>Count: {count}</h1>
      <button onClick={increment}>増やす</button>
      <button onClick={decrement}>減らす</button>
    </div>
  );
}
```

`count === 20` の場合、コンポーネントの中身を書き下してみると下記のようになっています。

```tsx
export default function App() {
const [20, setCount] = useState(0);
const increment = () => setCount(20 + 1);
const decrement = () => setCount(20 - 1);

  return (
    <div className="App">
      <h1>Count: {20}</h1>
      <button onClick={increment}>増やす</button>
      <button onClick={decrement}>減らす</button>
    </div>
  );
}
```

`increment` (1 増加) と言いつつ、その実 `setCount` に `21` を渡しているだけなのがわかります。
これを踏まえると、先ほど出てきた `increment` を 2 回繰り返した `incrementDouble` の中身は `count === 20` のとき下記のようになっていることがわかります。

```ts
const incrementDouble = () => {
  setCount(20 + 1);
  setCount(20 + 1);
};
```

`incrementDouble` という関数名であるにもかかわらず、ステートを `21` に更新する処理を 2 回繰り返しているだけです。これでは想定している動作が実現できるわけがありませんね。

## setState に渡される関数はステート変換の 指示 である

それでは `useCounter` を正しく修正しましょう。

```ts :useCounter.ts
export const useCounter = (init: number = 0) => {
  const [count, setCount] = useState(init);

  const increment = () => setCount((prevValue) => prevValue + 1);
  const decrement = () => setCount((prevValue) => prevValue - 1);

  return { count, increment, decrement };
};
```

`useCounter` を使用している `App` コンポーネント側は全く修正していないのでコードは省略します。

CodeSandbox で動作を確認してみます。

@[codesandbox](https://codesandbox.io/embed/for-zenn-usecounter-good-example-6fpi5?fontsize=14&hidenavigation=1&theme=dark)

ボタンをクリックすると、正しく差分 2 ずつカウントが変化しているのがわかります。

関数 `increment` はもはや `count` に依存していません。その代わり、 `setCount` に対して「現在のステートを渡してくれたらそれに 1 加算して返却するからそれを新しいステートとせよ」という**指示**を出しています。ここで言う「現在のステート」というのは `count` のことではなく、 React のバックグラウンドでステートを計算している途中の値のことです。

`count === 20` のときに ボタンをクリックして `incrementDouble` を実行した場合の内部動作を見てみましょう。

まずは直前のステートの値を 受け取ります:

```tsx
let newState = 20;
```

1 つ目の `increment` が実行されます。直前のステートの値である `20` に対して**指示**を実行します:

```tsx
newState = 20 + 1; // 21
```

ここで計算された `newState` はまだ画面には反映されません。同時に渡された**指示**をすべて処理し終えるまでループします。

2 つ目の `increment` が実行されます。直前のステートの値である `21` に対して**指示**を実行します:

```tsx
newState = 21 + 1; // 22
```

すべての**指示**を処理したので、ステートが確定して画面に反映されます。
これなら本当に意図していること(1 を加算する処理を 2 回行う)が実行されているのがわかりますね。

ちなみにここで説明している動作の本物のソースコード該当箇所は下記リンクです。ぜひ読んでみてください(`useState` は `useReducer` の特殊ケースとして実装されているので `useReducer` のソースコードとなります)。
https://github.com/facebook/react/blob/master/packages/react-dom/src/server/ReactPartialRendererHooks.js#L288-L305

ここまでカウンターアプリを題材にして説明してきましたが、ステートがどんな型であっても共通して言えることです。
**更新後のステートが更新前のステートに依存しているなら、 `setState` には値ではなく関数を渡してあげましょう。**

`setState` に渡す値を `state` から作るべきケースはないと思っています(あったらコメントください)。

## まとめ

「`setCount(count + 1)` と書くのはやめませんか」という話をしてきました。
代わりに `setCount((prevValue) => prevValue + 1)` と書くようにしましょう。

初学者向けの技術ブログ記事の場合、簡略化のためわざと書いている可能性もあります。しかし、僕はそれがいいこととはあまり思えません。初学者が今回の記事のように自力で応用してみようとしたときに思い通りに動作せず困惑させることになるからです。
それよりかは多少難易度が上がったり説明文が多くなったとしても、適切な API の紹介をすべきだと思います。

以上、最後までご覧いただきありがとうございました 🎉
