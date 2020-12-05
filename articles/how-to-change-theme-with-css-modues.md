---
title: "ダーク or ライトテーマの状態を React.useState で管理しながら CSS Modulesを使用する"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "css", "cssmodules"]
published: true
---

これは [React #2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/react-2) 5 日目の記事です。

# 目的

CSS Modules を使いつつ、ダーク or ライトテーマの状態は React でステート管理する(`@media (prefers-color-scheme: dark)` ではなく)場合の実装方法を考えてみました。
よくある要件だと思ったので似た記事が調べて出てくると思ったのですが、きっと CSS in JS でやっちゃうからか、なかなか見つかりませんでした。

## 要件

- テーマは React のステートで持ちたい

  現在表示中のテーマを React のステートとして管理しておきたい。OS のダークモード設定に依存せずにダーク or ライトを切り替えたいと思うことがあるので(僕だけかもしれない)。

```ts
const [theme, setTheme] = React.useState<"light" | "dark">("light");
```

- CSS Modules を使用する

  Next.js が押してるらしいです。
  僕がよく使う CSS in JS は [@material-ui/styles](https://material-ui.com/styles/basics/) です。というかこれ以外使ったことがありません。
  @material-ui/styles の `makeStyles` は CSS を JS のプレーンなオブジェクトとして定義していきます。ただし、あくまで JS オブジェクトなので VSCode で CSS を書くときの入力補完が弱いと感じています(TypeScript だとしても)。
  その点で言えば普通の CSS ファイルを書いていく CSS Modules はエディターの機能を最大限活かすことができます。

# 実装

## CSS

テーマで差し替えたい部分の色の定義に CSS の変数を使用します。

```css:styles.module.css
.app {
  font-family: sans-serif;
  text-align: center;
}

.element {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100px;
  background-color: var(--background-color);
  color: var(--font-color);
}
```

## React hook

React のステートに応じて CSS 変数を差し替えるカスタムフック `useTheme` を定義します。

```tsx:theme.tsx
import React from "react";

const useTheme = () => {
  const [theme, setTheme] = React.useState<"light" | "dark">("light");

  React.useEffect(() => {
    switch (theme) {
      case "light": {
        document.documentElement.style.setProperty("--background-color", "#efefef");
        document.documentElement.style.setProperty("--font-color", "#222222");
        break;
      }
      case "dark": {
        document.documentElement.style.setProperty("--background-color", "#222222");
        document.documentElement.style.setProperty("--font-color", "#efefef");
        break;
      }
    }
  }, [theme]);

  const toggleTheme = () => setTheme((prev) => (prev === "light" ? "dark" : "light"));

  return toggleTheme;
};
```

`useTheme` フックの内部で`theme` ステートを管理します。その値は `"light"` か `"dark"` になります。
続いて `theme` の値の変更によって実行される `useEffect` で CSS 変数を設定していきます。
`document.documentElement.style.setProperty("--your-color", "#123456")` と指定することで、

```css
:root {
  --your-color: #123456;
}
```

と指定するのと同じように CSS 変数を参照できるようになります。

`useTheme` からはテーマを切り替える関数 `toggleTheme` を返します(例として交互に変化させる関数にしていますが、ここは好きな関数インターフェイスで定義してください)。この関数を下位コンポーネントから使用するために、React context を定義してラップします。
`ThemeProvider` の内部で `useTheme` を実行した上で、その戻り値である `toggleTheme` 関数を `themeContext.Provider` で配信します。
`ThemeProvider` の購読用フック `useToggleTheme` も定義しておきます。

```tsx:theme.tsx
const themeContext = React.createContext<ReturnType<typeof useTheme>>(() => {});

export const ThemeProvider: React.FC = (props) => {
  const toggleTheme = useTheme();

  return (
    <themeContext.Provider value={toggleTheme}>
      {props.children}
    </themeContext.Provider>
  );
};

export const useToggleTheme = () => React.useContext(themeContext);
```

## CSS と hook を使用する側

トップレベルにて `ThemeProvider` で子コンポーネントを括ります。

```tsx:index.tsx
import React from "react";
import { render } from "react-dom";
import { ThemeProvider } from "./theme";

import App from "./App";

const rootElement = document.getElementById("root");
render(
  <ThemeProvider>
    <App />
  </ThemeProvider>,
  rootElement
);
```

通常通り `styles.module.css` を CSS Modules として使用します。
また、`useToggleTheme` で取得する `toggleTheme` 関数をボタンのクリックイベントに仕込みます。

```tsx:App.tsx
import React from "react";
import styles from "./styles.module.css";
import { useToggleTheme } from "./theme";

export default function App() {
  const toggleTheme = useToggleTheme();

  return (
    <div className={styles.app}>
      <div className={styles.element}>Hoge Hoge</div>
      <button onClick={toggleTheme}>Toggle Theme</button>
    </div>
  );
}
```

## デモアプリで動作確認

@[codesandbox](https://codesandbox.io/embed/change-theme-with-css-modules-r5lfh?fontsize=14&hidenavigation=1&theme=dark)

ボタンをクリックすると背景色と文字色がうまく変更されますね。

# まとめ

CSS Modules を用いながらテーマを React のステートとして保持する方法を紹介しました。
css 変数の定義が JS ファイルに紛れ込んでくるのが気持ち悪いかなとも思いましたが、そのへんを意識するのはアプリ全体でトップレベルの実装しているときだけで、CSS やコンポーネントを書くときは気にかける必要はないので許容範囲でしょう。
もっとスッキリ書ける方法があればコメントいただけると嬉しいです。
