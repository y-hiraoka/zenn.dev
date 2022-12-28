---
title: "【React】matchMedia で理解する useSyncExternalStore"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

React の API で、よくわからないしわかる必要性もあんまりない(かもしれない) React Hooks に `useSyncExternalStore` があります。Redux のように React 外で管理されているステートオブジェクトを React にインテグレートするためのフックということくらいは耳にしたことがあるのではないでしょうか。

そのフックの機能や使い方から主にステート管理ライブラリ開発者向けに用意されていると考えられます。ライブラリ開発者向け API ならアプリレイヤーの開発者には関係ないのではと思われるかもしれません。でも使い方を知っていれば、何か応用する案を思いつくこともあるでしょう。実際、 `useSyncExternalStore` はブラウザ API との統合にも使うことができるとドキュメントで紹介されています。

この記事では `useSyncExternalStore` と `matchMedia` を組み合わせたカスタムフック `useMatchMedia` を作ることで、使い方を理解していきたいと思います。

## 結論

```ts
import { useCallback, useMemo, useSyncExternalStore } from "react";

export function useMatchMedia(
  mediaQuery: string,
  initialState = false
): boolean {
  const matchMediaList = useMemo(
    () =>
      typeof window === "undefined" ? undefined : window.matchMedia(mediaQuery),
    [mediaQuery]
  );

  const subscribe = useCallback(
    (onStoreChange: () => void) => {
      matchMediaList?.addEventListener("change", onStoreChange);
      return () => matchMediaList?.removeEventListener("change", onStoreChange);
    },
    [matchMediaList]
  );

  return useSyncExternalStore(
    subscribe,
    () => matchMediaList?.matches ?? initialState,
    () => initialState
  );
}
```

動作確認できる codesandbox 埋め込みは下のほうにあります。

## `matchMedia` とは

`useSyncExternalStore` 以前に `matchMedia` が何者なのかわからないと理解できないので簡単に説明しましょう。

ブラウザの Web API のひとつで CSS のメディアクエリで判定できることを JavaScript でもできるようにする API です。

https://developer.mozilla.org/ja/docs/Web/API/Window/matchMedia

`window.matchMedia(query)` によって `MediaQueryList` オブジェクトを生成し、その `.matches` プロパティを確認することで指定したクエリにマッチしているかを判定できます。

```ts
const mediaQueryList = window.matchMedia("(min-width: 599px)");
console.log(mediaQueryList.matches); // true or false
```

CSS のメディアクエリの書き方を知っていれば、非常にわかりやすい API になっていますね。

`MediaQueryList` のパワフルなところは、判定結果の変化を検知できることです。

```ts
// 画面サイズが 599px を前後するたびにログが記録される
mediaQueryList.addEventListener("change", () => {
  console.log(mediaQueryList.matches);
});
```

これも JavaScript によくあるイベントリスナーの形式なので直感的ですね。もちろんリスナー解除の `removeEventListener` も用意されています。

## `useSyncExternalStore` とは

https://beta.reactjs.org/reference/react/useSyncExternalStore

Redux のように React 外で管理されているステートを React コンポーネントで購読できるようにするフックです(上のドキュメントはベータ版サイトです)。

React コンポーネントは基本的に React が管理している props, state, context 以外のデータを読むことができません(適当なグローバル変数を参照することはできますが、その値が変化してもコンポーネントが再レンダリングできません)。それだと React 非依存の独自ステートを持つライブラリなどとの統合が難しくなります。それを解決するために用意されたのが `useSyncExternalStore` です。

`useSyncExternalStore` がなかった時代は `useState` の `setState` を強制再レンダリング関数とみなし、外部ステートの変更検知イベントに `setState` を仕込むような書き方がされていました。ただしそれは React が意図した使い方ではなく、React に Transition の概念が導入されて破綻しました([参考](https://qiita.com/uhyo/items/6a3b14950c1ef6974024))。`useSyncExternalStore` は言わば避難ハッチとして実装された経緯があります。

インターフェイスは次のようになっています。

```ts
export function useSyncExternalStore<Snapshot>(
  subscribe: (onStoreChange: () => void) => () => void,
  getSnapshot: () => Snapshot,
  getServerSnapshot?: () => Snapshot
): Snapshot;
```

第 1 引数の `subscribe` は `onStoreChange` 関数を受け取って関数を返す関数になっています(超わかりにくい)。名前の通り対象の外部ステートの変更を購読する処理を渡す口です。外部ステートが変化したら `onStoreChange` を実行することで React に再レンダリングを依頼することができます。返す関数は外部ステートの購読を終了するために実行するクリーンアップ関数です(`useEffect` みたいですね)。

第 2 引数の `getSnapshot` は外部ステートの値を取得するための関数です。ステートは同一性を担保する必要があり、常に別のオブジェクトを返すような次のような関数を渡すと不具合に繋がります。

```ts
const getSnapshot = () => {
  return { foo: "bar" };
};
```

第 3 引数の `getServerSnapshot` は Server Side Rendering 時のステートを決定する関数です。オプショナルですが Next.js のような SSR/SSG フレームワークが主流の現代ではほぼ指定必須でしょう。渡していないのにサーバーサイドで実行されるとエラーを吐きます。

## `useMatchMedia` を実装する

言葉では `useSyncExternalStore` の使い方をイメージしにくいと思います。`matchMedia` と `useSyncExternalStore` を組み合わせて、宣言的にメディアクエリを指定できるカスタムフック `useMatchMedia` を作ってみましょう。

:::message
汎用カスタムフックライブラリには大抵 `useMatchMedia` に相当するフックが用意されているので、車輪の再発明と言えます。例えば [react-use](https://github.com/streamich/react-use/blob/master/docs/useMedia.md) には `useMedia` という名前のカスタムフックが提供されています。ただし `useSyncExternalStore` が使われていることは少ないです。
:::

実装する関数インターフェイスは次のようなイメージです。

```ts
function useMatchMedia(mediaQuery: string, initialState = false): boolean;
```

`initialState` はメディアクエリ判定ができない、つまりサーバーサイドでコードが実行されるときに使用される想定です。

このようなカタチをしたカスタムフックなら、次のように簡単かつ宣言的に使うことができるでしょう。

```tsx
const MyApp: FC = () => {
  const isMobileSize = useMatchMedia("(max-width: 599px)");

  return <p>{isMobileSize ? "mobile" : "not mobile"}</p>;
};
```

それでは順番に作っていきます。

### `useMemo` で `MediaQueryList` オブジェクトを保持する

```ts
const matchMediaList = useMemo(
  () =>
    typeof window === "undefined" ? undefined : window.matchMedia(mediaQuery),
  [mediaQuery]
);
```

`window.matchMedia` を実行するたびに新しい `matchMediaList` オブジェクトが生成されます。それでは勝手が悪いので引数の `mediaQuery` が変化した時だけ再生成されるように `useMemo` で括っておきます。

SSR のときは window オブジェクトが存在しないので 存在チェックもしておきましょう。

### subscribe 関数を定義

```tsx
const subscribe = useCallback(
  (onStoreChange: () => void) => {
    matchMediaList?.addEventListener("change", onStoreChange);
    return () => matchMediaList?.removeEventListener("change", onStoreChange);
  },
  [matchMediaList]
);
```

先程宣言した `matchMediaList` オブジェクトを使って、 `useSyncExternalStore` の第 1 引数となる `subscribe` 関数を定義します。`subscribe` 関数の引数には React に再レンダリング依頼をするための `onStoreChange` 関数を渡してもらえるので、それをメディアクエリの判定結果が変わったタイミングで発火します。そのタイミングを捕捉できるのはもちろん `addEventListener` で登録する change イベントです。`onStoreChange` 関数をそのまま `addEventListener` に渡します。

`subscribe` 関数からは外部ステートの購読を停止する時に呼ぶクリーンアップ関数の返却が求められます。`addEventListener` で開始した購読は、`removeEventListener` で解除することが可能です。戻り値のクリーンアップ関数内で `removeEventListener` を実行しましょう。

`useSyncExternalStore` に渡す `subscribe` 関数は参照が同一であることが大切です。`useSyncExternalStore` は再レンダリング時に `subscribe` 関数を `Object.is` で比較して、異なるときに前回のクリーンアップ関数を発火します。つまり再レンダリングのたびに違う `subscribe` 関数を渡すと、購読開始と購読解除を何度も実行することになります。引数や他のステートに依存しないのであれば、`useMatchMedia` の関数スコープの外で `subscribe` 関数を宣言すればいいですが、今回は引数に依存するため `useCallback` で参照同一を担保します。`matchMediaList` を `useMemo` で括ったのも `subscribe` 関数を固定するためです。(パフォチュ以外に `useMemo` or `useCallback` を使っていいんだっけなと思いつつも)この方法はドキュメントにも記載されていました。

https://beta.reactjs.org/reference/react/useSyncExternalStore#my-subscribe-function-gets-called-after-every-re-render

### useSyncExternalStore で購読を開始

```tsx
return useSyncExternalStore(
  subscribe,
  () => matchMediaList?.matches ?? initialState,
  () => initialState
);
```

いよいよ本題の `useSyncExternalStore` を使います。

第 1 引数は 用意しておいた `subscribe` 関数です。再びの言及ですが、参照が無駄に変化しないような渡し方をしてください。

第 2 引数は外部ステートとなる値を取得する関数です。今回外部ステートとみなしているのは `matchMediaList.matches` です。それを return するように関数を書いておきます。一応 `matchMediaList` は `undefined` の可能性があるので、オプショナルチェーンと null 合体演算子で必ず boolean 値が返るようにしておきます。`subscribe` とは異なり第 2 引数に渡す関数それ自体は参照を安定させる必要はないですが、その戻り値は React ステートとして扱われるので、やはり `Object.is` で比較して変化したときに再レンダリングされることを念頭に置いてください。毎回新しいオブジェクトが返されるような書き方では無限ループになり得ます。

第 3 引数は SSR 時(SSG 含む)とそれを hydrate するときにだけ使用される関数です。どのみち クライアントサイドまで到達しないとメディア判定はできませんので、`initialState` を固定で返しておきます。そう言った意味では引数の `initialState` は `valueOnSSR` みたいな変数名でもいいかもしれません。

### 動かしてみる

画面サイズを判定するメディアクエリで試してみましょう。せっかくなので新しい Range Syntax のメディアクエリを使ってみます(動かないブラウザがあります。手元の Safari はだめでした。[caniuse](https://caniuse.com/css-media-range-syntax) 参考)。

```tsx
export default function App() {
  const isMobileSize = useMatchMedia("(width < 600px)");
  const isTabletSize = useMatchMedia("(600px <= width <= 1024px)");
  const isDesktopSize = useMatchMedia("(1024px < width)");

  return (
    <div className="App">
      <p>
        isMobileSize: <b>{String(isMobileSize)}</b>
      </p>
      <p>
        isTabletSize: <b>{String(isTabletSize)}</b>
      </p>
      <p>
        isDesktopSize: <b>{String(isDesktopSize)}</b>
      </p>
    </div>
  );
}
```

@[codesandbox](https://codesandbox.io/embed/zenn-use-sync-external-store-with-match-media-et0n4x?fontsize=14&hidenavigation=1&theme=dark)

上の埋め込み要素で [Open Sandbox] をクリックすれば新規タブで codesandbox のプレビューが開きます。PC の方はブラウザサイズをグリグリして試してみてください。スマホやタブレットでご覧の方はデバイスの方向を変えて画面幅を変化させてみてください。いずれかの値が `false` と `true` を行き来していれば正しく動作しています。

以上で実装は終了です！「意外と使い所がありそう！」と感じていただけたら本望です。

## 注意点

これもドキュメントで言われていることですが、 `useState` や `useReducer` でできることはそちらを優先して使ってください。`useSyncExternalStore` はその名の通り外部のデータストアと同期を取ることを目的としたフックです。あなたが新しいステートを用意するときに、わざわざ React の管理外にステートを宣言する必要はありません。ちょっとしたグローバルステートがほしいのであれば `useState` で宣言したものを Context で配信するなどで対応しましょう。もしくは Recoil 使いましょう。

## まとめ

`useMatchMedia` フックを作ることで `useSyncExternalStore` の使い方を紹介しました。

第 1 引数の `subscribe` 関数の参照や `getSnapshot` の戻り値の同一性には注意してください。
`useState` でできることをわざわざ `useSyncExternalStore` でやる必要はありません。まず `useState` でできるか検討して下さい。
でも `useSyncExternalStore` の使い所は意外と多そうです。色々用途を考えてみてください。ぜひそれを教えてください。

それではよい React ライフを！
