---
title: "useMediaQuery は最終手段にしよう"
emoji: "📵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
publication_name: chot
---

こんにちは、エンジニアです。

本記事では`useMediaQuery`を使うべきではない理由を説明します。

## `useMediaQuery`とは

`window.matchMedia`の判定結果を取得するカスタムフックを指します。

```tsx
const isWideScreen = useMediaQuery("(min-width: 768px)");
```

`window.matchMedia`はCSSでできるメディアクエリの判定をJavaScriptでも可能にするブラウザAPIです。それをReact Hooksと組み合わせることで、宣言的に判定を行えるようなカスタムフックとなります。

過去に`useSyncExternalStore`を使って実装する記事を書いたので参考にしてみてください。

https://zenn.dev/stin/articles/use-sync-external-store-with-match-media

上の記事では`useSyncExternalStore`の使い道を説明するのが目的だったのですが、Reactのフックとしてメディアクエリを使える、使っても良いと捉えられてしまったようで、題材選びを少し反省しています(？)

以下はユーティリティ系ライブラリによる提供例です。関数名は提供元によってバラツキがありますが、おおよそ同じことをしています。

- [react-responsive - useMediaQuery](https://github.com/yocontra/react-responsive/blob/516ef9b9b39966f399356c1389bf871321846b1a/src/useMediaQuery.ts)
- [react-use - useMedia](https://github.com/streamich/react-use/blob/ade8d3905f544305515d010737b4ae604cc51024/src/useMedia.ts)
- [usehooks-ts - useMediaQuery](https://github.com/juliencrn/usehooks-ts/blob/20667273744a22dd2cd2c48c38cd3c10f254ae47/packages/usehooks-ts/src/useMediaQuery/useMediaQuery.ts)
- [@uidotdev/usehooks - useMediaQuery](https://github.com/uidotdev/usehooks/blob/90fbbb4cc085e74e50c36a62a5759a40c62bb98e/index.js#L785-L857)
- その他色々

## 問題点

`useMediaQuery`は`window`オブジェクトに依存しています。つまりブラウザ上でしか正しく判定できません。

一方で昨今はサーバーサイドレンダリング(SSR)でHTMLを生成してからブラウザ上でイベントハンドラーを復元するhydrationを行います。サーバーサイドにはwindowオブジェクトがないため、固定値を返すかエラーを投げるしかありません。

サーバーサイドで固定値を返す場合、その固定値と実際のブラウザ上で計算される値が異なる可能性があります。作りが悪いカスタムフックだと、これがhydrationエラーとしてレポートされることになります。作りが良くて(？)`useSyncExternalStore`や`useEffect`を上手く駆使しているとしても、それをスタイリングの制御に使っている場合はレイアウトシフトとして悪影響を及ぼします(本記事では適用されるCSSが突然変化してチラチラして見える現象を総称してレイアウトシフトと呼ぶことにします)。

レイアウトシフトになるのは、以下の順序で処理が行われるためです。

1. サーバーサイドで固定値を使ってHTMLが生成される
2. ブラウザが固定値を使って生成されたHTMLを画面に反映する
3. 画面反映後`useEffect`で`matchMedia`の計算を実行する
4. `matchMedia`の計算結果を改めて画面に反映する

敢えて`useEffect`と書いたのは`useSyncExternalStore`を使っても`useEffect`と変わらないと言えるためです。それは React17 以前のバージョン向けに用意された`useSyncExternalStore`のポリフィルから判断できます。ポリフィルのソースコードを見ると、`useSyncExternalStore`は`useEffect`(または`useLayoutEffect`)で代替実装ができるフックです。

[https://github.com/facebook/react/blob/main/packages/use-sync-external-store/src/useSyncExternalStoreShimClient.js](https://github.com/facebook/react/blob/main/packages/use-sync-external-store/src/useSyncExternalStoreShimClient.js)

## 解決策

スタイリングの制御に使うなら、まずCSSのメディアクエリで実装できないか検討してください。CSSで実装すれば最初からブラウザのサイズが考慮されたスタイリングが当たり、レイアウトシフトの原因にはなりません。

## 使っても良い箇所

SSR対応フレームワークを使用していない(ブラウザで空のHTMLにSPAをマウントして使っている)のであれば、どんな箇所でも使用可能です。

SSR対応フレームワークでも、クライアントサイドレンダリングだけで使用されるコンポーネントなら使っても悪い現象は起きません。ユーザーインタラクションによって初めて表示されるダイアログなどが該当します。ちなみに、「クライアントサイドレンダリングだけ」というのは`"use client"`を付けることではありません。

dynamic importされるコンポーネントでも良いです。dynamic importされたコンポーネントはクライアントサイドレンダリングであることが保証されます。Next.js なら`next/dynamic`も含みます。

@uidotdev/usehooksには`useIsClient`というカスタムフックがあり、その戻り値でクライアントサイドかを判定してコンポーネントを表示するかどうか決めることもできます。

色々回避策はありますが、スタイリングを切り替えたいだけならやはりCSSだけで実装できないか検討してください。スタイリングはCSSの役割です。

## @uidotdev/usehooksの方針

@uidotdev/usehooksはブラウザJavaScriptのAPIを使用するカスタムフックがサーバーサイドレンダリング中に実行される場合、エラーをthrowする方針のようです。

https://github.com/uidotdev/usehooks/issues/254#issuecomment-1778088998

「hydrationエラーをごまかすためだけの作り方は好きではない」という強い意思を持ってエラーとしているようで、僕はとても好きな方針です。

## まとめ

- `useMediaQuery`は使わずに、CSSのメディアクエリで実現できないかを検討する
- 仮に使うことになっても、必ずユーザーに違和感を与えるチラツキを抑える
- ライブラリがSSRでエラーをthrowするのは思想なので従おう
- 回避策はあるけど回避するな

それでは良いReactライフを！
