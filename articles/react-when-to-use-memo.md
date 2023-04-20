---
title: "【React】useMemo の使い時をまとめる"
emoji: "🗒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
publication_name: chot
---

chot Inc. で Web エンジニアをしているすてぃんです。今回は社内で `useMemo` の使い時がわからんという話題が挙がったので、ケースによる使い時と解説をまとめました。コードレビュー時などの参考になれば幸いです。

## 結論

- 値の計算量が大きい場合: **使う**
- 値の計算量が小さい場合
  - 値が primitive の場合: **使わない**
  - 値がオブジェクトや配列の場合
    - 値をスコープ外に持ち出す場合: **使う**
    - 値をスコープ外に持ち出さない場合: **使わなくてもいい**
- 値が関数の場合: **`useCallback` を使う**
- 色々条件あってよくわからんという場合: **使わなくていいです**

## 前提知識

### `useMemo` とは

`useMemo` は次のような型で定義される React Hooks の 1 つです。

https://github.com/DefinitelyTyped/DefinitelyTyped/blob/176187350228c605498041d1b98722b1457eb5e6/types/react/index.d.ts#L1095-L1102

第 1 引数で渡す関数 `factory` の戻り値と `useMemo` 自体の戻り値の型が一致しています。それもそのはずで、 `useMemo` は内部で `factory` を実行して得た結果を返すだけだからです。

使い方は次の通り。

```tsx
const MyComponent = () => {
  const [name, setName] = useState("");
  const [age, setAge] = useState(0);

  const inputData = useMemo(() => ({ name, age }), [name, age]);

  return <>...</>;
};
```

初めて実行される時や、第 2 引数の `deps` の要素 1 つ 1 つを比較して変化している時は `factory` が実行されて値を生成し、その値をメモ化しつつ返します。`deps` が変化していなければ `factory` の実行をスキップし、以前の値を使い回す仕組みです。

### React のレンダリングとは

込み入った解説はしませんが、レンダリングが何かについて触れておかないとこの先が納得できないと思うので軽く説明します。

React のレンダリングとはコンポーネントの本体たる関数を評価(実行)することです。レンダリングという名前が付いていますが、あくまで関数評価です。

再レンダリングするきっかけはいくつかあって、

- State が変化する
- 親が再レンダリングする(`React.memo` 未使用の場合)
- `useContext` で購読している Context が変化する

などが挙げられます。

State の持ち方などによっては高頻度に再レンダリングが行われるコンポーネントも有り得ますし、それ自体はあまり問題ではありません。

問題となるのは、関数の評価中に高負荷な計算が行われたり、再レンダリングの影響範囲が大きすぎる場合です。

## ケース毎の説明

### 値の計算量が大きい場合

関数コンポーネントの処理中に高負荷な計算を行う場合、 `useMemo` を使用してその計算をスキップすべきです。ただし、その名の通りメモであり初回の計算は当然避けられませんので、初回レンダリング時の高負荷な処理は許容する必要があります。

ではどの程度の計算量なら高負荷と言えるでしょうか。実は僕もあまり基準を調べずに使っていたのですが、公式ドキュメントが一新されて処理時間の目安が記載されていました。

https://react.dev/reference/react/useMemo#how-to-tell-if-a-calculation-is-expensive

この tip によると、1ms 以上かかる処理があれば `useMemo` を使用して処理をスキップしたほうがいいとのこと。計測方法も記載されていて親切ですね。

```tsx
const App: React.FC = () => {
  // レンダリングのたびに重い処理が実行される
  const computedValue1 = heavyComputing(deps1, deps2);

  // deps が変化したときだけ重い処理が実行される
  const computedValue2 = useMemo(
    () => heavyComputing(deps1, deps2),
    [deps1, deps2]
  );
};
```

### 値の計算量が小さい場合

値の計算量が小さい場合には、`useMemo` を使ってスキップされる処理時間よりも `useMemo` を使うこと自体のオーバーヘッドが大きくなります。

なぜなら `useMemo` を使うことには、

- インライン関数の宣言と実行
- 依存配列(`deps`)の各要素の等値判定
- 値のキャッシュ領域の確保

が含まれるからです。

では、欲しい値の計算量が小さい時は常に `useMemo` を使うべきではないかというと、実はそうではありません。

計算量が小さくても `useMemo` を使うメリットは、値の参照が無駄に変化しないことです。React は「`setState` で受け付けた値が現在 State と異なるか」「Context の値が変化しているか」「依存配列が前回レンダリング時と異なるか」など、あらゆる等値判定で `Object.is` を使用します。オブジェクトや配列などの非 primitive を `Object.is` で比較する時、参照が異なるかどうかで判定されます。比較した結果別の値と判定されれば、再レンダリングするとか effect を実行するとか追加の処理が行われるきっかけになるため、参照の変化を最小限に抑えることは重要です。

### 値が primitive の場合

導出の計算量が小さい値が primitive の場合は `useMemo` を使用する意味はありません。primitive 値の等値判定はその値の内容で決まります。 `useMemo` でキャッシュしなくても、 primitive 値なら値が変化したかどうかを適切に判定できます。

```tsx
const App: React.FC = () => {
  // これで十分
  const fullName1: string = firstName + lastName;

  // 意味がないどころか useMemo のオーバーヘッドが無駄
  const fullName2: string = useMemo(
    () => firstName + lastName,
    [firstName, lastName]
  );
};
```

### 値がオブジェクトや配列の場合

導出の計算量が小さい値がオブジェクトや配列の場合は `useMemo` で値をキャッシュすることで、新たな値の参照を生成しないようにすることができます。しかし、オブジェクトや配列ならば常に参照の変化を避けたいわけではなく、参照が変わっても構わないケースがあります。

具体的には、値がそのスコープから持ち出されるか否か、という条件で区別します。

`useMemo` が使用されるのは関数コンポーネントかカスタムフックの中です。「値を持ち出す」というのは、次のケースを想定しています。

- 関数コンポーネントで使用し、別のコンポーネントに props 経由で渡す
- 関数コンポーネントで使用し、別のカスタムフックの引数に渡す
- カスタムフックで使用し、そこから `return` する
- カスタムフックで使用し、別のカスタムフックの引数に渡す

注意点として、 `<div />` や `<p />` といった HTML そのままの要素(intrinsic elements)の props(attributes) に渡すことは該当しません。

#### 値をスコープ外に持ち出す場合

この節は少し僕の設計思想を含みます。

先述の通り React はあらゆる等値判定で `Object.is` を使用します。例えば `React.memo` で定義した関数コンポーネントは「親が再レンダリングしても、そこから受け取る props が変化していなければ自身は再レンダリングしない」という性質を持ちますが、各 props が変化しているかどうかの判定に `Object.is` が使用されます。 子として使うコンポーネントが `React.memo` で定義されていれば、渡す値に `useMemo` を使ったほうが余分な再レンダリングを防げてお得です。しかし、 子コンポーネントが `React.memo` されていないのであれば、 `useMemo` を使っていても子コンポーネントの再レンダリングは防げないので使うだけ無駄です。

カスタムフックの引数に渡すケースでも似たことが考えられます。別のカスタムフックが引数で受け取った値を `useEffect` や `useMemo` の `deps` として使っているかもしれません。`deps` の比較にも `Object.is` が使用されているので、引数に渡す値が `deps` として使われているのであれば `useMemo` を使っておいてあげたほうが effect を余分に発火することもないし、別の `useMemo` のキャッシュを無駄に破棄させることもないでしょう。しかし、 `deps` に使われていないのであれば `useMemo` するだけ無駄になるかもしれません。

ですが、コンポーネントやカスタムフックを実装していて別のコンポーネントやカスタムフックの内部実装が気になってしまうのは責務の逸脱と言えます。関数コンポーネントなら最終的に `return` する JSX を組み立てることだけを考えるべきだし、カスタムフックならそのフック自身の処理結果だけを考えるべきです。よその実装のためにこちらの内部実装の振る舞いを変更しないといけないのであれば、責務が正しく分離されておらず設計が間違っている可能性があります。よって値をスコープ外に持ち出す場合は、常に `useMemo` を使って参照の変化を最小限に抑えることで、別の実装のことを考慮する必要がなくなります。

```tsx
const useMyHook = () => {
  const [name, setName] = useState("");
  const [age, setAge] = useState(0);

  const userData = useMemo(() => ({ name, age }), [name, age]);

  return userData;
};
```

ちなみにこの考えは次の記事に大きく影響を受けています。

https://blog.uhy.ooo/entry/2021-02-23/usecallback-custom-hooks/

#### 値をスコープ外に持ち出さない場合

値をスコープ外に持ち出さない場合を考えます。同じ関数スコープに属する `useEffect` で消費するようなケースで、`useEffect` の実行回数を最適化するために使用したくなるかもしれません。

```tsx
const useSomeHook = () => {
  const userData = useMemo(() => ({ name, age }), [name, age]);

  useEffect(() => {
    doSomething(userData);
  }, [userData]);
};
```

しかしこういったケースでは、単に `useEffect` の中で変数宣言をすればいいだけの場合が多いです。 `userData` の宣言を `useEffect` に移動してやることで `useMemo` をなくすことができます。

```tsx
const useSomeHook = () => {
  useEffect(() => {
    const userData = { name, age };

    doSomething(userData);
  }, [name, age]);
};
```

複数の `useEffect` で利用するから事前に変数宣言しないといけないんだと言う場合は `useMemo` が有効です。

```tsx
const useSomeHook = () => {
  const userData = useMemo(() => ({ name, age }), [name, age]);

  useEffect(() => {
    doSomething1(userData);
  }, [userData]);

  useEffect(() => {
    doSomething2(userData);
  }, [userData]);
};
```

それでも、 `useMemo` を使うモチベーションにパフォーマンス改善以外を見出してはいけません。例えば `useEffect` が余分に実行されることでデータ不整合が発生するのであれば、それは冪等でない処理に問題があります。つまり、 `useMemo` を使わない次のコードは、使っている直前のコードと比べて実行回数以外の結果(描画結果、サーバーの状態など)が同じである必要があります。

```tsx
const useSomeHook = () => {
  const userData = { name, age };

  useEffect(() => {
    doSomething1(userData);
  }, [userData]);

  useEffect(() => {
    doSomething2(userData);
  }, [userData]);
};
```

以上より、値をそのスコープから持ち出さない場合は使わなくてもいいと考えます。

### 欲しい値が関数の場合

JavaScript では関数もオブジェクトの一種なので、`Object.is` は関数の参照によって等値判定を行います。それゆえに次のように `useMemo` を使って値をキャッシュすることに意味はあります。

```tsx
const handleClick = useMemo(() => () => doSomething(), []);
```

しかし関数の場合は `useCallback` がビルトインで用意されていますのでそちらを使うべきです。`useCallback` を使った方がネストが浅くなりますし、callback というワードから関数を宣言していることが一目瞭然です。

```tsx
const handleClick = useCallback(() => doSomething(), []);
```

### 色々条件あってよくわからんという場合

適切な使い時が判別できないのであれば、一切使わなくて良いと思います。ゴリゴリステート更新が走るような Web アプリケーションを実装しているならともかく、ちょっとした開閉状態を持っている程度の Web サイトにはまったく不要と言えます。`useMemo` などを使わずにコンポーネントが余分にレンダリングしても、画面に反映すべきことがなければそれで処理が終了するので十分高速です。

ドキュメントにも記載されていますが、`useMemo` はパフォーマンス改善のためだけに使用するフックです。`useMemo` を使用しなくとも正しく画面が表示されることをまず目指してください。`useMemo` を使わないとデータがおかしくなるとか画面表示がおかしくなるなどが起きている場合は、明確にバグを含んでいますので `useMemo` を使わなくても正しい挙動をするように修正箇所を探るべきです。そのうえでパフォーマンス改善すべきと判断されるのであれば初めて `useMemo` を使ってください。

参照の不変性に意味を見出しているケースも良くないです。例えば `useMemo` でクラスインスタンスを保持したりする場合です。React は将来的に `useMemo` で記憶していた値を破棄する機能を入れる可能性があると言っています([参考](https://react.dev/reference/react/useMemo#caveats))。破棄されて困る値は `useState` や `useRef` を使いましょう。例えば Supabase の Next.js チュートリアルでは、setter を使わない `useState` に Supabase Client を突っ込むようなサンプルコードになっています。

https://supabase.com/docs/guides/getting-started/tutorials/with-nextjs#set-up-a-login-component

## まとめ

React の `useMemo` の使い時について説明しました。

- `useMemo` はパフォーマンス改善のためのフックです
- 計算量の大きい箇所は積極的に使いたい
- 計算量が小さいなら使わなくていいケースが多い
- よくわからんのであればわかってから使おう

ぜひ必要な箇所で `useMemo` を使用していただきたいです。ぼくは React なんもわからんので使ってません(？)

それでは良い React ライフを！
