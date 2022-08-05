---
title: "Meta の新しいリッチテキストエディターフレームワーク Lexical を調べる(実践編)"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lexical", "react"]
published: true
---

# Lexical で簡単なリッチテキストエディターを作ってみよう

Lexical は Meta が開発したリッチテキストエディターフレームワークです。以前 Lexical のアーキテクチャを調べてまとめた記事を執筆したので、概要や設計についてはそちらをご覧ください。

https://zenn.dev/stin/articles/getting-started-with-lexical

Lexical のコアライブラリは `EditorState` の差分から最も効率のいい方法で DOM を更新する Reconciler がメインで、エディターの機能やビューはプラグインという形で各々実装する必要があります。

ただ、コアライブラリだけ提供しても使う側が戸惑ってしまうので、すでに多くの公式プラグインが同じリポジトリで提供されています。

そして注目すべきは Playground の充実度です。

https://playground.lexical.dev/

もはや Playground をそのまま使っても十分なほど多機能なエディターが実装されています。もちろんソースコードは同じリポジトリに公開されているので、実装の参考にできます。

本記事では Lexical Playground を参考にしつつより簡易版のリッチテキストエディターを作ってみることで Lexical の理解を深めることを目標とします。

## 完成イメージ

本記事で作成するエディターの完成イメージはこちらになります。

![見出しブロック、インラインスタイル、コードブロック、リストブロックが入力されたリッチテキストエディター](https://storage.googleapis.com/zenn-user-upload/0cd4cb8956c7-20220804.png)

実際に触ってみたい方はこちらの URL からどうぞ。

https://y-hiraoka.github.io/lexical-rich-editor-trial/

ソースコードを見たい方はこちらのリポジトリを御覧ください。

https://github.com/y-hiraoka/lexical-rich-editor-trial

## 用語定義

この記事だけでの用語の意味を定義しておきます。 Lexical 公式で使用されている単語ではありません。

テキストエディターの 1 行分の入力を**ブロック**と呼びます。おおよそ HTML の `p` 要素 1 つ分とか `h1` 要素 1 つ分に相当する単位だと捉えていただければいいです。

主にボタン操作によってエディターの状態を変換する UI コンポーネントを**ツールバー**と呼ぶことにします。リッチテキストエディターの上部によくある Paragraph ブロックを Heading ブロックに変換するボタンなどを含むコンポーネントのことですね。

## 作ってみよう

### プロジェクト作成

今回は React アプリケーションとして作っていきます。 Vite を使って爆速開発していきましょう。

```bash
npm create vite@latest lexical-rich-editor-trial -- --template react-ts
```

使っている npm バージョンによってパラメータの渡し方が異なるようです。詳しくは [vite 公式ページ](https://vitejs.dev/guide/#scaffolding-your-first-vite-project)を確認してください。

コアパッケージと React 用バインドをインストールしましょう。

```bash
npm install lexical @lexical/react
```

Lexical には既にたくさんの公式プラグインが提供されていますが、そのほぼすべてが `lexical/react` の依存パッケージとして指定されています。ですので明示的にインストールする必要はありません。

本記事ではスタイリングについては特に解説しませんが、私は SCSS による CSS Modules で書くので `sass` をインストールしておきます。

```bash
npm install sass
```

CSS in JS ライブラリを使ってもいいですが、 class 名の文字列が取れる必要があるのでご注意ください。CSS を考えるのが面倒なときは僕のリポジトリからコピペしちゃってください。

アイコンコンポーネントライブラリとして react-icons を使用するのでインストールしておきます。

```bash
npm install react-icons
```

### 最小構成

[こちらの記事](https://zenn.dev/stin/articles/getting-started-with-lexical)では vanilla JS での最小構成のコードを紹介しましたが、今回は React アプリケーションとしての最小構成です。

```tsx:src/Editor.tsx
import { ComponentProps, FC } from "react";
import { LexicalComposer } from "@lexical/react/LexicalComposer";
import { ContentEditable } from "@lexical/react/LexicalContentEditable";
import { RichTextPlugin } from "@lexical/react/LexicalRichTextPlugin";

const initialConfig: ComponentProps<typeof LexicalComposer>["initialConfig"] = {
  namespace: "MyEditor",
  onError: (error) => console.error(error),
};

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      <RichTextPlugin
        contentEditable={<ContentEditable />}
        placeholder={<div>いまなにしてる？</div>}
      />
    </LexicalComposer>
  );
};
```

`LexicalComposer` は Lexical のコア API のひとつである `LexicalEditor` インスタンスを `createEditor` によって生成して、それを `Context.Provider` で配信する役割を担います。生成する JSX は `Context.Provider` だけで具体的な DOM 要素は含みません。 `initialConfig` props は必須なのでオブジェクトで渡しておきましょう。 `namespace` の用途は不明です。

`RichTextPlugin` はその名の通り rich text なエディターの機能をセットアップしてくれるプラグインです。対して `@lexical/react/LexicalPlainTextPlugin` というモジュールも存在し、より機能の少ないエディターのセットアップを行います。 contenteditable な div 要素を生成するための `ContentEditable` コンポーネントも公式で用意されています。`RichTextPlugin` の `contentEditable` props から差し込めるようになっているので自作することも可能ですが、 `ContentEditable` には `className` が渡せるようになっているため、よっぽど特殊な処理を実装したいわけじゃなければ公式コンポーネントで十分でしょう。一方 `RichTextPlugin` の　`placeholder` props に渡すための要素はコンポーネントとして用意されていません。 `ContentEditable` の次の位置に描画されることだけを頭の片隅に置いて、自由な要素を渡してください。

上記の `Editor` コンポーネントを `App.tsx` で呼び出しておきましょう。

```tsx:src/App.tsx
import "./App.css";
import { Editor } from "./Editor";

function App() {
  return (
    <div className="App">
      <Editor />
    </div>
  );
}

export default App;
```

この状態で vite の開発サーバーを起動すれば、次のようなアプリを起動できます。

![スタイルのついていない編集エリアとプレースホルダーを持つエディター](/images/lexical-rich-editor-trial/01_editor_sample.png)

この HTML 構造は次のようになっています。

```html
<div class="App">
  <div
    contenteditable="true"
    spellcheck="true"
    data-lexical-editor="true"
    style="user-select: text; white-space: pre-wrap; word-break: break-word"
    role="textbox"
  >
    <p><br /></p>
  </div>
  <div>いまなにしてる？</div>
</div>
```

エディター部分になにか入力すると「いまなにしてる？」という placeholder が消えるのが確認できます。最小構成のテキストエディターができましたね。

`RichTextPlugin` を使っていますが、装飾するためのボタンなどがないので `textarea` と大差ありません。これをベースに、リッチにしていきましょう！

### プレースホルダーの位置を変える

そのままではエディターの下にプレースホルダーが表示されてしまいます。プレースホルダーはエディターの上に少し薄いグレーで表示したいですね。 CSS を当てましょう。そのまえにリセット CSS を適用します。

```bash
npm install ress
```

```tsx:src/main.tsx
import React from "react";
import ReactDOM from "react-dom/client";
import "ress";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

なんでもいいですが、僕がいつも使うリセット CSS は `ress` です。これをインストールして `src/main.tsx` で import します。

次に `Editor.module.scss` を用意して `Editor.tsx` で使用します。

```tsx:src/Editor.tsx
import styles from "./Editor.module.scss";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      <div className={styles.editorContainer}>
        <RichTextPlugin
          contentEditable={<ContentEditable className={styles.contentEditable} />}
          placeholder={<div className={styles.placeholder}>いまなにしてる？</div>}
        />
      </div>
    </LexicalComposer>
  );
};
```

先述の通り `LexicalComposer` は DOM 要素を描画しないので、 `RichTextPlugin` を `div` で括っておきます。また、 `ContentEditable` と `placeholder` にクラス名を付与しておきます。これらのクラスに対して CSS を書きましょう。

```scss:src/Editor.module.scss
.editorContainer {
  position: relative;
  padding: 24px;
  min-height: 240px;
}

.contentEditable {
  outline: none;
}

.placeholder {
  position: absolute;
  color: #888888;
  top: 24px;
  left: 24px;
  pointer-events: none;
  user-select: none;
}
```

`position` で `placeholder` をエディター本体である `contentEditable` に重ねるようにします。 `placeholder` 自体はユーザーのインタラクションに反応してほしくないので、 `pointer-events` と `user-select` を `none` にしておきましょう。細かい見た目はおまかせします。おしゃれにやっちゃってください。

![CSSが適用されてプレースホルダーが丁度いいポジションに重なっているエディター](/images/lexical-rich-editor-trial/02_editor_sample.png)

はいかっこいい(？)

### 初めてのプラグインを実装する

最初のプラグインを実装しましょう。内容は、差し込むだけで「エディターマウント時に自動でフォーカスしてくれる」機能です。

```tsx:src/plugins/AutoFocusPlugin.tsx
import { FC, useEffect } from "react";
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";

export const AutoFocusPlugin: FC = () => {
  const [editor] = useLexicalComposerContext();

  useEffect(() => {
    editor.focus();
  }, [editor]);

  return null;
};
```

Lexical with React の文脈でプラグインというと、このように `useLexicalComposerContext` で `LexicalEditor` のインスタンスを取得して、 `useEffect` の中で作用を起こしたり処理を登録したり、ボタンのイベントでインスタンスメソッドを呼び出したりするコンポーネントを指します。インスタンスに対して作用させるだけで特に DOM 要素を描画する必要がない場合は `null` を return しておきます。

`useLexicalComposerContext` は名前の通り内部で `useContext` を使っており、対応する Provider は `LexicalComposer` です。なので、その子孫コンポーネントとして差し込みます。

```tsx:src/Editor.tsx
// 省略
import { AutoFocusPlugin } from "./plugins/AutoFocusPlugin";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      <div className={styles.editorContainer}>
        <RichTextPlugin
          contentEditable={<ContentEditable className={styles.contentEditable} />}
          placeholder={<div className={styles.placeholder}>いまなにしてる？</div>}
        />
      </div>
      <AutoFocusPlugin />
    </LexicalComposer>
  );
};
```

これでエディターがマウントされるとエディターにフォーカスが当たった状態になります。ちょっと便利になりました。

### undo, redo をつける

最小構成のテキストエディターになにか入力して `cmd + z` を押してもテキストをもとに戻すことはできません。戻すことができなければ進むことも当然できません。 undo, redo のできないテキストエディターは、非常に不便で誰も使ってくれないでしょう。作っていきましょう。

undo 機能はユーザーインターフェイスとして頻出の割に実装が難しいことで有名です。が、Lexical においては一瞬です。

```tsx:src/Editor.tsx
// 省略
import { HistoryPlugin } from "@lexical/react/LexicalHistoryPlugin";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      {/*省略*/}
      <HistoryPlugin />
    </LexicalComposer>
  );
};
```

`HistoryPlugin` を `LexicalComposer` の子要素として差し込むだけで undo, redo 機能が実装できます。まじありがたい。

せっかくなので、 `HistoryPlugin` が何をしているか見てみましょう。

https://github.com/facebook/lexical/blob/6daa62dfa637d1f698391765d9f16aed72eb4dbc/packages/lexical-react/src/LexicalHistoryPlugin.ts

`useLexicalComposerContext()` で `LexicalEditor` インスタンスを取得して `useHistory` フックに渡しています。どうやら具体的な処理は `useHistory` に実装してあるのでそちらのコードも見てみます。

https://github.com/facebook/lexical/blob/6daa62dfa637d1f698391765d9f16aed72eb4dbc/packages/lexical-react/src/shared/useHistory.ts

受け取った `LexicalEditor` インスタンスを `useEffect` の中で `registerHistory` に渡しています。この `registerHistory` が `LexicalEditor` に対して履歴の管理やキーボードショートカットの**機能を登録する関数**になっています(`registerHistory` の実装まで見るのは負担が大きいのでここまで)。よく見ると `registerHistory` の戻り値を `useEffect` から `return` しているのがわかります。 `registerHistory` の戻り値は関数になっていて、実行することで**機能の登録を解除する**ことができます。それを `useEffect` のクリーンアップ関数とすることで、アンマウント時にエディターから機能を剥がすことが可能になります。

このように、Lexical では `registerXXX` という名前の関数で機能を追加することが多いです。register というプレフィックスは、 `LexicalEditor` のメソッド名(`registerCommand`, `registerUpdateListener` など)が由来です。また、戻り値は必ず `() => void` 関数を返します。これは unregister 処理をする関数になっており、そのまま useEffect のクリーンアップ関数にできます。独自機能を実装する場合も公式プラグインに倣って register プレフィックスの命名と戻り値を unregister 関数にすることをルールにするといいでしょう。

### 見出し入力機能を実装する

リッチテキストエディターといえば見出しを入力できる機能ですね。HTML の `h1` 〜 `h6` に対応し、(一般的には)大きなフォントサイズで太字のテキストを入力できるようにしましょう。

Lexical の入力データの単位は `LexicalNode` クラス(を継承して作られた具象クラス)で表現されます。DOM と同様に `LexicalNode` を木構造に持つことで入力全体のステートを管理します。コアパッケージ `lexical` から export されている Node クラスは `ElementNode`, `TextNode`, `DecoratorNode` の 3 つだけなので、それらを継承してより具体的な Node クラスを作成する必要があります。見出しも例外ではありません。

しかし、Node クラスを全部自作する必要はなく、公式提供の Node クラスがたくさんあります。見出しブロックに対応する `HeadingNode` は `@lexical/rich-text` パッケージから export されています。

デフォルトの Node である `ParagraphNode` 以外の Node クラスは `LexicalEditor` インスタンスに予め登録する必要があります。公式プラグインである `@lexical/rich-text` も例外ではないので、登録するコードを書きましょう。 `src/nodes.ts` ファイルを作成して次のコードを書きます。

```tsx:src/nodes.ts
import { HeadingNode } from "@lexical/rich-text";
import { Klass, LexicalNode } from "lexical";

export const nodes: Klass<LexicalNode>[] = [HeadingNode];
```

`Klass` はインスタンス化可能なクラスだけに限定するためのユーティリティ型です(TypeScript では class の定義と同時に interface が定義されるので、 class に限定する意図があります)。 `HeadingNode` 以外にも Node クラスの登録をする場合はここに追記していくようにします。これを `src/Editor.tsx` で使用しましょう。

```tsx:src/Editor.tsx
// 省略
import { nodes } from "./nodes";

const initialConfig: ComponentProps<typeof LexicalComposer>["initialConfig"] = {
  namespace: "MyEditor",
  onError: (error) => console.error(error),
  nodes: nodes,
};
```

`initialConfig` の `nodes` プロパティが `LexicalEditor` に Node クラス一覧を渡す口になっています。ここにエディターで使用する Node クラスを渡します。

`HeadingNode` が `LexicalEditor` インスタンスに認識されるようになったので、続いて見出しに変換するボタンを提供するツールバー `src/plugins/ToolbarPlugin.tsx` を作成します。

```tsx:src/plugins/ToobarPlugin.tsx
const SupportedBlockType = {
  paragraph: "Paragraph",
  h1: "Heading 1",
  h2: "Heading 2",
  h3: "Heading 3",
  h4: "Heading 4",
  h5: "Heading 5",
  h6: "Heading 6",
} as const;
type BlockType = keyof typeof SupportedBlockType;
```

まずはサポートするブロックの種類と表示名をプレーンオブジェクトで持っておきます。ランタイムで変更することがないので `as const` も付けておきましょう。そして、その `keyof` 型を `BlockType` としておきます。

続いてツールバーのビューを作ります。ボタン要素を並べるだけです。

```tsx:src/plugins/ToobarPlugin.tsx
import { FC, useState } from "react";
import { TbH1, TbH2, TbH3 } from "react-icons/all";
import styles from "./ToolbarPlugin.module.scss";

export const ToolbarPlugin: FC = () => {
  const [blockType, setBlockType] = useState<BlockType>("paragraph");

  return (
    <div className={styles.toolbar}>
      <button
        type="button"
        role="checkbox"
        title={SupportedBlockType["h1"]}
        aria-label={SupportedBlockType["h1"]}
        aria-checked={blockType === "h1"}
      >
        <TbH1 />
      </button>
      <button
        type="button"
        role="checkbox"
        title={SupportedBlockType["h2"]}
        aria-label={SupportedBlockType["h2"]}
        aria-checked={blockType === "h2"}
      >
        <TbH2 />
      </button>
      <button
        type="button"
        role="checkbox"
        title={SupportedBlockType["h3"]}
        aria-label={SupportedBlockType["h3"]}
        aria-checked={blockType === "h3"}
      >
        <TbH3 />
      </button>
    </div>
  );
};
```

`useState` で `BlockType` 型のステートを宣言します。これでどのブロックタイプが指定されているかを保持します。値の切り替えロジックは後ほど。

`<button>` は on/off のチェックボックス的に使用するので、 `role="checkbox"` と `aria-checked` を渡しておきます。 `aria-checked` を渡せば CSS のセレクターに利用でき、on/off の見た目をスタイリングすることもできます。

ここまでで一旦画面に表示してみます。 `src/Editor.tsx` で `ToolbarPlugin` コンポーネントを使います。

```tsx:src/Editor.tsx
import { ToolbarPlugin } from "./plugins/ToolbarPlugin";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      <ToolbarPlugin />
      {/*省略*/}
    </LexicalComposer>
  );
};
```

CSS を適当につけておけばこんな感じの見た目になります。さらにリッチテキストエディターっぽくなりましたね。

![エディター上部に H1, H2, H3 ボタンを持つツールバーが表示されている様子](/images/lexical-rich-editor-trial/03_editor_sample.png)

さて、ロジックを作っていきましょう。まず、ボタンのクリックによって指定したレベルの見出しブロックに変換する処理を書きます。

```tsx:src/plugins/ToolbarPlugin.tsx
// 省略
import { HeadingTagType, $createHeadingNode } from "@lexical/rich-text";
import { $wrapLeafNodesInElements } from "@lexical/selection";

export const ToolbarPlugin: FC = () => {
  const [blockType, setBlockType] = useState<BlockType>("paragraph");
  const [editor] = useLexicalComposerContext();

  const formatHeading = useCallback(
    (type: HeadingTagType) => {
      if (blockType !== type) {
        editor.update(() => {
          const selection = $getSelection();
          if ($isRangeSelection(selection)) {
            $wrapLeafNodesInElements(selection, () => $createHeadingNode(type));
          }
        });
      }
    },
    [blockType, editor],
  );

  return (/*省略*/);
};
```

`useLexicalComposerContext` で `LexicalEditor` インスタンスを取得します。`HeadingTagType` を引数に取る `formatHeading` 関数を定義して、その中で `editor.update()` を呼び出します(`HeadingTagType` は `@lexical/rich-text` から import できます)。Lexical では `$` プレフィックス関数は特別な場所でしか呼び出せないようになっています。 `editor.update()` はそのひとつで、`EditorState` の更新を行えます。

具体的には、まず `$getSelection()` で現在の `Selection` を取得します。`Selection` とはエディター上での選択状態を管理するクラスです。Lexical の `Selection` クラスはいくつか種類がありますが、今回の操作は `RangeSelection` の場合だけ、つまりただキャレットがピコピコしているかテキストを選択している状態のときだけ行います。それを `$isRangeSelection()` で判定します(TypeScript 的には Type-Guard 関数になっていて便利です)。続いて `$wrapLeafNodesInElements()` を使います。これを使うと、指定した `RangeSelection` が指している Node を 指定した Node でラップするようです。ドキュメントがないので厳密な振る舞いはわかっていません。新しい `HeadingNode` を作成するには `$createHeadingNode(type)` を使用します。

これをそれぞれのボタンのクリックイベントに渡しましょう。

```tsx:src/plugins/ToolbarPlugin.tsx
<button
  // 省略
  onClick={() => formatHeading("h1")}
>
  <TbH1 />
</button>
```

`h2`, `h3` も同様にクリックイベントに渡しておきます。

![H1〜H3ボタンをクリックすることでテキストが対応するレベルのHeadingに変換されるアニメーション](/images/lexical-rich-editor-trial/04_editor_sample.gif)

見出しブロックへ変換できるようになりました！

続いて対応するボタンがアクティブな見た目になるように、 `blockType` が切り替わるロジックを書いていきましょう。

```tsx:src/plugins/ToolbarPlugin.tsx
// 省略

export const ToolbarPlugin: FC = () => {
  // 省略

  useEffect(() => {
    return editor.registerUpdateListener(({ editorState }) => {
      editorState.read(() => {
        const selection = $getSelection();
        if (!$isRangeSelection(selection)) return;

        const anchorNode = selection.anchor.getNode();
        const targetNode =
          anchorNode.getKey() === "root"
            ? anchorNode
            : anchorNode.getTopLevelElementOrThrow();

        if ($isHeadingNode(targetNode)) {
          const tag = targetNode.getTag();
          setBlockType(tag);
        } else {
          const nodeType = targetNode.getType();
          if (nodeType in SupportedBlockType) {
            setBlockType(nodeType as BlockType);
          } else {
            setBlockType("paragraph");
          }
        }
      });
    });
  }, [editor]);

  return (/*省略*/);
};
```

`editor.registerUpdateListener()` は Lexical の `registerXXX` の一種で、 `EditorState` が変更されると実行されるリスナー関数を登録しておくことができます。もちろん戻り値は unregister 関数なので、そのまま `useEffect` のクリーンアップ関数にできます。

`editorState.read()` は `editor.update()` と同様に `$` プレフィックス関数が使用できるスコープです。この中でステートを読み、 `setBlockType` に値を渡します。

先ほどと同様に、 `$getSelection()` で現在の `Selection` を参照し、 `RangeSelection` かどうかをチェックしてそうでなければ無視します。 `selection.anchor` は選択しているテキスト範囲の先端の位置情報を持つオブジェクトです。末端の位置情報は `selection.focus` で、ただのキャレットの時は `anchor` と `focus` が同じ位置を指します(JavaScript 的に同じオブジェクトになるわけではないです)。 `selection.anchor.getNode()` で `anchor` が指している Node を取得し、その `getKey()` が `"root"` の場合はそれ自身を、そうでない場合は `getTopLevelElementOrThrow()` で親方向に Node を上まで探索して取得できたものを `targetNode` とします。 `getKey() === "root"` を満たす Node は Lexical コアパッケージの `RootNode` クラスだけですが、ここを `Selection` が指すことは恐らくないでしょう。

`targetNode` が `HeadingNode` かどうかを確認します。 `$isHeadingNode` は `@lexical/rich-text` に含まれる Type-Guard 関数です。 `HeadingNode` は見出しレベルに関わらず `getType()` で `"heading"` を返すので、ブロックタイプの判定に `getTag()` を使います。こちらは `"h1"` 〜 `"h6"` を返すので、 そのまま `setBlockType` の引数に渡すことができます。

`HeadingNode` ではない場合は `getType()` を使うようにします。あとで Node の種類を追加するので、意図しないブロックタイプが混入しないように `"paragraph"` にフォールバックするような条件分岐で書いておきます。

![H1〜H3ボタンをクリックすると、クリックされたボタンがアクティブになる様子](/images/lexical-rich-editor-trial/05_editor_sample.gif)

ブロックタイプの切り替えによってボタンのアクティブが変化するようになりました！複数の色々なタイプのブロックをエディターに入力して、カーソルキーでキャレットを上下してもちゃんとボタンのアクティブが追随することが確認できるので試してみてください。

次はエディターに描画される `HeadingNode` に CSS を当ててみます。 `editorTheme.ts` ファイルを用意して次のオブジェクトを定義します。

```tsx:src/editorTheme.ts
import { EditorThemeClasses } from "lexical";
import styles from "./editorTheme.module.scss";

export const theme: EditorThemeClasses = {
  heading: {
    h1: styles.h1,
    h2: styles.h2,
    h3: styles.h3,
    h4: styles.h4,
    h5: styles.h5,
    h6: styles.h6,
  },
};
```

`EditorThemeClasses` は `LexicalEditor` に渡す CSS クラスを一括で受け取るオブジェクト型です。プロパティの命名から想像できるように、他の要素へのスタイリングもこのオブジェクトから受け取ります。また、ただの文字列を受け取るだけなので CSS Modules でもプレーンな CSS でも (クラス名を生成できる)CSS in JS でも使えます。

そしてこの `theme` オブジェクトを `LexicalEditor` に渡します。例によって `initialConfig` が受け渡し口になっています。

```tsx:src/Editor.tsx
// 省略
import { theme } from "./editorTheme";

const initialConfig: ComponentProps<typeof LexicalComposer>["initialConfig"] = {
  namespace: "MyEditor",
  onError: (error) => console.error(error),
  nodes: nodes,
  theme: theme,
};

// 省略
```

これで `HeadingNode` が描画するビューに対してスタイリングされるようになりました。CSS は各自でご用意ください。

注意点として、こちらのソースコードを御覧ください。 `LexicalComposer` の内部で `createEditor` を読んでいる部分になります。

https://github.com/facebook/lexical/blob/6daa62dfa637d1f698391765d9f16aed72eb4dbc/packages/lexical-react/src/LexicalComposer.tsx#L50-L87

`useMemo` の中で `initialConfig` を使用していますが、その依存配列には `initialConfig` が含まれていません。コードのコメントにもある通り init のときだけ値を参照するので妥当だとは思いますが、このように書くと HMR(Hot Module Replacement) も意味がなくなるんですね。 例えば `editorTheme.module.scss` を変更すると `editorTheme.tsx` の `theme` も別のオブジェクトになるので、 `LexicalEditor` が読み込んでいる `theme` が新しくなってほしいところですが無視されてしまいます。結果としてただ前のスタイリングが剥がれるだけになります。これを避けるには `theme` オブジェクトが変更される必要がないプレーン CSS を使うことですが、本質的な解決ではありません。改善されると信じて、今は甘んじて受け入れましょう…。

### 引用ブロックを実装する

引用ブロックもリッチテキストエディターの定番機能ですね。 HTML の `blockquote` に対応するブロックで、他人の発言や文章を紹介する目的で使用します。

といっても `HeadingNode` を追加する手順とまったく同じです。サクッと作っていきましょう。

`LexicalEditor` に `QuoteNode` を登録します。 `QuoteNode` クラスは `@lexical/rich-text` パッケージに含まれています。

```tsx:src/nodes.ts
import { HeadingNode, QuoteNode } from "@lexical/rich-text";
export const nodes: Klass<LexicalNode>[] = [HeadingNode, QuoteNode];
```

`SupportedBlockType` に `quote` プロパティを追加します。

```tsx:src/plugins/ToolbarPlugin.tsx
const SupportedBlockType = {
  paragraph: "Paragraph",
  h1: "Heading 1",
  h2: "Heading 2",
  h3: "Heading 3",
  h4: "Heading 4",
  h5: "Heading 5",
  h6: "Heading 6",
  quote: "Quote",
} as const;
```

`QuoteNode` に変換する関数とそれをクリックで実行するツールバーボタンを用意します。 `$createQuoteNode` も `@lexical/rich-text` パッケージから import できます。

```tsx:src/plugins/ToolbarPlugin.tsx
// 省略

export const ToolbarPlugin: FC = () => {
  // 省略

  const formatQuote = useCallback(() => {
    if (blockType !== "quote") {
      editor.update(() => {
        const selection = $getSelection();
        if ($isRangeSelection(selection)) {
          $wrapLeafNodesInElements(selection, () => $createQuoteNode());
        }
      });
    }
  }, [blockType, editor]);

  // 省略

  return (
    <div className={styles.toolbar}>
      {/*省略*/}
      <button
        type="button"
        role="checkbox"
        title={SupportedBlockType["quote"]}
        aria-label={SupportedBlockType["quote"]}
        aria-checked={blockType === "quote"}
        onClick={formatQuote}
      >
        <MdFormatQuote />
      </button>
    </div>
  );
};
```

`QuoteNode` は `.getType()` で `"quote"` の文字列が取得できるので、 `setBlockType` でブロックタイプを更新するための `useEffect` は修正する必要がありません。

`QuoteNode` がエディター上に描画する `blockquote` に対して当てるクラス名を設定します。

```tsx:src/editorTheme.ts
export const theme: EditorThemeClasses = {
  heading: {
    /*省略*/
  },
  quote: styles.quote,
};
```

はい、爆速で引用ブロックが使えるようにできました！

### リスト入力機能を実装する

箇条書きを入力するのに使用するリストブロックを実装しましょう。HTML では `ul`, `ol`, `li` に対応するブロックになります。

一般的なリッチテキストエディターにはネストしたリストを入力できる機能があります。ただ、それを実装するにはかなり複雑なステート管理の処理を書く必要があり、けっこう大変です。しかしそこは Meta エンジニアの手厚い配慮から(？)、 undo redo などと同様に公式プラグインを差し込むだけでロジックが実装されるようになっています。

おまけに GitHub の Markdown にあるようなチェックリストも実装できるようになっています。それに乗っかって Ordered List, Unordered List, Check List を実装していきましょう。

まず例に漏れず Node クラスの登録をします。リスト機能用の Node クラスは `ListNode` と `ListItemNode` 2 つに分かれており、両方とも `@lexical/list` から import できます。

```tsx:src/nodes.ts
import { ListItemNode, ListNode } from "@lexical/list";

export const nodes: Klass<LexicalNode>[] = [
  HeadingNode,
  QuoteNode,
  ListItemNode,
  ListNode,
];
```

そして先程述べた、複雑なリストのロジックを実装してくれている公式プラグインを使います。

```tsx:src/Editor.tsx
// 省略

import { CheckListPlugin } from "@lexical/react/LexicalCheckListPlugin";
import { ListPlugin } from "@lexical/react/LexicalListPlugin";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      {/*省略*/}
      <ListPlugin />
      <CheckListPlugin />
    </LexicalComposer>
  );
};
```

`CheckListPlugin` と `ListPlugin` です。チェックリスト機能が不要な場合は `ListPlugin` だけ反映させれば良いです。

続いて `ToolbarPlugin` を編集し、ボタンをクリックすることでリストブロックに変換できるようにします。

```tsx:src/plugins/ToolbarPlugin.tsx
const SupportedBlockType = {
  // 省略
  number: "Numbered List",
  bullet: "Bulleted List",
  check: "Check List",
} as const;
```

Ordered List は `number` ブロックで、 Unordered List は `bullet` ブロックです。 `ol`, `ul` じゃないのかよと思われたらすみません(？)。 Playground に倣っており、 `listNode.getListType()` の戻り値と一致するので扱いやすいという理由があります。

次に現在の `Selection` が指すブロックをそれぞれのリストブロックに変換する処理を書きます。

```tsx:src/plugins/ToolbarPlugin.tsx
const formatBulletList = useCallback(() => {
  if (blockType !== "bullet") {
    editor.dispatchCommand(INSERT_UNORDERED_LIST_COMMAND, undefined);
  }
}, [blockType, editor]);

const formatNumberedList = useCallback(() => {
  if (blockType !== "number") {
    editor.dispatchCommand(INSERT_ORDERED_LIST_COMMAND, undefined);
  }
}, [blockType, editor]);

const formatCheckList = useCallback(() => {
  if (blockType !== "check") {
    editor.dispatchCommand(INSERT_CHECK_LIST_COMMAND, undefined);
  }
}, [blockType, editor]);
```

`INSERT_UNORDERED_LIST_COMMAND`, `INSERT_ORDERED_LIST_COMMAND`, `INSERT_CHECK_LIST_COMMAND` は `@lexical/list` パッケージから import できます。

はい出ました `editor.dispatchCommand` です。[前回の記事](https://zenn.dev/stin/articles/getting-started-with-lexical)を読んでくださっている人は雰囲気だけでも知っているかもしれません。Flux アーキテクチャにおいて Action を dispatch すると reducer が新しい state を生成するように、Lexical には Command を dispatch すると予め登録(`editor.registerCommand`)しておいた処理が実行されて `EditorState` が更新されるという処理フローが備わっています。上記コードでは Command を dispatch する部分を行っているというわけですね。では dispatch された `INSERT_UNORDERED_LIST_COMMAND` などの Command に対応する処理はどこで登録されているかというと、さっき差し込んだ `ListPlugin` の中で登録されています。

`ListPlugin` の内部(`useList` が呼び出されている)
https://github.com/facebook/lexical/blob/6daa62dfa637d1f698391765d9f16aed72eb4dbc/packages/lexical-react/src/LexicalListPlugin.ts#L15-L29

`useList` の実装(`editor.registerCommand` が呼び出されている)
https://github.com/facebook/lexical/blob/6daa62dfa637d1f698391765d9f16aed72eb4dbc/packages/lexical-react/src/shared/useList.ts#L30-L88

プラグインの方で Command に対するステート更新処理を登録してくれるので、 dispatch するだけでいい感じにブロック変換してくれるのです。

Flux において Action には payload を一緒に dispatch することができますが、 Lexical の `dispatchCommand` においては第二引数がそれに当たります。今回は 3 つの Command すべてが `LexicalCommand<void>` 型、つまり payload 不要の Command であるため、 `undefined` を渡しています。

`editor.dispatchCommand` を内包した関数はボタン要素に渡しておきましょう。

```tsx:src/plugins/ToolbarPlugin.tsx
return (
  <div className={styles.toolbar}>
    {/*省略*/}
    <button
      type="button"
      role="checkbox"
      title={SupportedBlockType["bullet"]}
      aria-label={SupportedBlockType["bullet"]}
      aria-checked={blockType === "bullet"}
      onClick={formatBulletList}
    >
      <MdFormatListBulleted />
    </button>
    <button
      type="button"
      role="checkbox"
      title={SupportedBlockType["number"]}
      aria-label={SupportedBlockType["number"]}
      aria-checked={blockType === "number"}
      onClick={formatNumberedList}
    >
      <MdFormatListNumbered />
    </button>
    <button
      type="button"
      role="checkbox"
      title={SupportedBlockType["check"]}
      aria-label={SupportedBlockType["check"]}
      aria-checked={blockType === "check"}
      onClick={formatCheckList}
    >
      <MdChecklist />
    </button>
    {/*省略*/}
  </div>
);
```

次はブロックタイプを検知する箇所(`useEffect` で `setBlockType` を実行する部分)です。

```tsx:src/plugins/ToolbarPlugin.tsx
//省略
import { $isListNode, ListNode } from "@lexical/list";
import { $getNearestNodeOfType } from "@lexical/utils";

// 省略
useEffect(() => {
  return editor.registerUpdateListener(({ editorState }) => {
    editorState.read(() => {
      // 省略

      if ($isHeadingNode(targetNode)) {
        // 省略
      } else if ($isListNode(targetNode)) {
        const parentList = $getNearestNodeOfType(anchorNode, ListNode);
        const listType = parentList ? parentList.getListType() : targetNode.getListType();

        setBlockType(listType);
      } else {
        // 省略
      }
    });
  });
}, [editor]);
```

`HeadingNode` 同様、Ordered List, Unordered List, Check List をひとつの `ListNode` で担うため、単純な `node.getType()` では `"list"` の文字列しか取得できません。ですので、 `$isListNode` で分岐してからごにょごにょとします。

また、リストはネストしている可能性があります。例えば Unordered List の子 Node として Ordered List が存在し、現在の Selection が子の Ordered List を指している場合は、アクティブなブロックは Ordered List と判定されるように考慮します。

上記コードでは、 `targetNode` が `ListNode` の場合、その子要素にリストを持っている可能性があるので `selection.anchor` が指す木構造上の末端 Node から親方向に `ListNode` を探索します。そこで `ListNode` が取れたらその `getListType()` を使い、取れなかったら `targetNode.getListType()` を使うようにします。 `getListType()` の戻り値は `"number" | "bullet" | "check"` なので、そのまま `setBlockType` に渡すことができます(というかこれに合わせています)。

最後にエディターに描画される `ol`, `ul`, `li` に対するスタイリングです。

```tsx:src/editorTheme.ts
export const theme: EditorThemeClasses = {
  // 省略

  list: {
    ul: styles.ul,
    ol: styles.ol,
    listitem: styles.listitem,
    nested: {
      listitem: styles.nestedListItem,
    },
    listitemChecked: styles.listitemChecked,
    listitemUnchecked: styles.listitemUnchecked,
  },
};
```

チェックリストやネストがあるのでクラスが多めに用意されています。上記コードでは使用していないクラス名もあるので型定義を参考にベストなスタイルを探ってみてください。

自分はこんな感じの見た目にしました。ネスト段階の調整は Tab と Shift+Tab で可能です。

![リストブロックが正常に使用できる様子](/images/lexical-rich-editor-trial/06_editor_sample.gif)

以上でリスト機能の実装は終わりです。

### コード入力機能を実装する

エンジニアのみなさんがブログを書くときに必ず必要なものがコードブロックですね。指定したプログラミング言語でシンタックスハイライトができ、コードエディターのように等幅フォントで表示される編集エリアを実装していきましょう。

いつものように、 Node クラスの登録からです。コードブロック単位とシンタックスハイライトされるときの単語単位の 2 つの Node が `@lexical/code` に用意されています。

```tsx:src/nodes.ts
import { CodeNode, CodeHighlightNode } from "@lexical/code";

export const nodes: Klass<LexicalNode>[] = [
  HeadingNode,
  QuoteNode,
  ListItemNode,
  ListNode,
  CodeNode,
  CodeHighlightNode,
];
```

List の複雑なロジックを公式プラグインとして用意してくれていたように、コード入力についてもシンタックスハイライトをするためのロジックを作ってくれています。

ただ、なぜか `ListPlugin` のような React コンポーネントにはしてくれていないので自分で作る必要があります。

```tsx:src/plugins/CodeHighlightPlugin.tsx
import { FC, useEffect } from "react";
import { registerCodeHighlighting } from "@lexical/code";
import { useLexicalComposerContext } from "@lexical/react/LexicalComposerContext";

export const CodeHighlightPlugin: FC = () => {
  const [editor] = useLexicalComposerContext();

  useEffect(() => {
    return registerCodeHighlighting(editor);
  }, [editor]);

  return null;
};
```

`registerCodeHighlighting` がシンタックスハイライト用のロジックを登録するための関数です。内部では [Prism.js](https://prismjs.com/) が使用されています。 `registerXXX` なので戻り値は unregister 関数で、そのまま `useEffect` のクリーンアップにできます。

公式プラグイン同様に `LexicalComposer` の子コンポーネントとして使います。

```tsx:src/Editor.tsx
// 省略

import { CodeHighlightPlugin } from "./plugins/CodeHighlightPlugin";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      {/*省略*/}
      <CodeHighlightPlugin />
    </LexicalComposer>
  );
};
```

続いてツールバーにボタンを増やす作業です。ブロックタイプの種類は `"code"` にします。

```tsx:src/plugins/ToolbarPlugin.tsx
const SupportedBlockType = {
  // 省略
  code: "Code Block",
} as const;
```

そして、現在のキャレット位置のブロックをコードブロックに変換する関数と、それを呼び出すボタンです。これらは引用ブロックと全く同じ形です。 `$createCodeNode` は `@lexical/code` パッケージから import できます。

```tsx:src/plugins/ToolbarPlugin.tsx
export const ToolbarPlugin: FC = () => {
  // 省略

  const formatCode = useCallback(() => {
    if (blockType !== "code") {
      editor.update(() => {
        const selection = $getSelection();
        if ($isRangeSelection(selection)) {
          $wrapLeafNodesInElements(selection, () => $createCodeNode());
        }
      });
    }
  }, [blockType, editor]);

  // 省略

  return (
    <div className={styles.toolbar}>
      {/*省略*/}
      <button
        type="button"
        role="checkbox"
        title={SupportedBlockType["code"]}
        aria-label={SupportedBlockType["code"]}
        aria-checked={blockType === "code"}
        onClick={formatCode}
      >
        <MdCode />
      </button>
    </div>
  );
};
```

そして描画されるコードブロックのスタイリングです。これはちょっと厄介で、 Prism.js の実装を知る必要があります…。というのも [Prism.js の CSS](https://github.com/PrismJS/prism/blob/master/themes/prism.css) をそのまま読み込んでおけば色付けされるようにはなっておらず、 Prism.js が本来付与するであろうクラス名をキーに、自分で用意したクラス名を渡します。日本語じゃわかりにくいと思うのでコード例。

```tsx:src/editorTheme.ts
export const theme: EditorThemeClasses = {
  // 省略

  code: styles.code,
  codeHighlight: {
    atrule: styles.tokenAttr,
    attr: styles.tokenAttr,
    boolean: styles.tokenProperty,
    builtin: styles.tokenSelector,
    cdata: styles.tokenComment,
    char: styles.tokenSelector,
    class: styles.tokenFunction,
    "class-name": styles.tokenFunction,
    comment: styles.tokenComment,
    constant: styles.tokenProperty,
    deleted: styles.tokenProperty,
    doctype: styles.tokenComment,
    entity: styles.tokenOperator,
    function: styles.tokenFunction,
    important: styles.tokenVariable,
    inserted: styles.tokenSelector,
    keyword: styles.tokenAttr,
    namespace: styles.tokenVariable,
    number: styles.tokenProperty,
    operator: styles.tokenOperator,
    prolog: styles.tokenComment,
    property: styles.tokenProperty,
    punctuation: styles.tokenPunctuation,
    regex: styles.tokenVariable,
    selector: styles.tokenSelector,
    string: styles.tokenSelector,
    symbol: styles.tokenProperty,
    tag: styles.tokenProperty,
    url: styles.tokenOperator,
    variable: styles.tokenVariable,
  },
};
```

`theme.code` はコードブロック全体の要素に当たります。そして `theme.codeHighlight` の各クラス名はシンタックスハイライトで色付けされる単語単位に当たります。 `theme.codeHighlight` の型定義は(執筆時点で) `Record<string, string>` になっているので、本当に Prism.js の実装を調べてどんなクラス名を付与するのか知っている必要があるのです。流石にそれはめんどくさすぎるので、 [Playground のソースコード](https://github.com/facebook/lexical/blob/6daa62dfa637d1f698391765d9f16aed72eb4dbc/packages/lexical-playground/src/themes/PlaygroundEditorTheme.ts#L15-L47)を参考にして書くといいです。

そして対応する CSS について、 CSS の内容はこれまで省いていたのですが、コードブロックについては注目すべき点があるので紹介します。それは CSS を工夫するだけで行番号が表示できるようになっていることです。

```scss:src/editorTheme.module.scss
.code {
  background-color: #f7fafb;
  font-family: Menlo, Consolas, Monaco, monospace;
  display: block;
  padding: 8px 8px 8px 52px;
  line-height: 1.6;
  font-size: 14px;
  margin: 8px 0;
  tab-size: 2;
  overflow-x: auto;
  position: relative;

  &::before {
    content: attr(data-gutter);
    color: #999;
    position: absolute;
    top: 0;
    left: 0;
    background-color: #d9dddf;
    padding: 8px;
    min-width: 32px;
    height: 100%;
    text-align: right;
  }
}
```

上記コードからわかるように、 `data-gutter` 属性に行番号が改行された状態で突っ込んであります。それを `::before` の `content` に `attr()` で渡し、いい感じにスタイリングすることで行番号を見せることができます。行番号が不要なら `::before` を書かなければいいだけなので、CSS だけで調整できるのが簡単でいいですね。

ここまでで、シンタックスハイライト付きコードブロックで入力できるようになりました。プログラミング言語選択機能は未実装ですが、 `CodeNode` の言語初期値が `"javascript"` なので最低限 JavaScript のハイライトはできます。

![コードブロックにJavaScriptコードがシンタックスハイライトされた状態で入力されている](/images/lexical-rich-editor-trial/07_editor_sample.png)

いい感じですね！

それではプログラミング言語の選択機能を実装していきましょう。ツールバーのコードブロックボタンの横に言語選択のドロップダウンリストを配置することにします。

実装は Command 方式でやってみます。Command と、それを受け取ったら発火されるプログラミング言語変更処理を実装しておき、ドロップダウンリストの変更イベント時に Command を dispatch するイメージです。

まずは Command とそれに対応する処理を書きましょう。

```tsx:src/plugins/CodeHighlightPlugin.tsx
//省略

export const CODE_LANGUAGE_COMMAND = createCommand<string>();

function registerCodeLanguageSelecting(editor: LexicalEditor): () => void {
  return editor.registerCommand(
    CODE_LANGUAGE_COMMAND,
    (language, editor) => {
      const selection = $getSelection();
      if (!$isRangeSelection(selection)) return false;

      const anchorNode = selection.anchor.getNode();
      const targetNode = $isCodeNode(anchorNode)
        ? anchorNode
        : $getNearestNodeOfType(anchorNode, CodeNode);
      if (!targetNode) return false;

      editor.update(() => {
        targetNode.setLanguage(language);
      });

      return true;
    },
    COMMAND_PRIORITY_CRITICAL
  );
}
```

Command は `createCommand` で宣言します。 dispatch のときに一緒に送信してほしい payload がある場合は Generics でその型を指定しておきます。今回はどの言語を選択したのかを文字列で教えてもらいたいので `createCommand<string>()` としました。他のファイルで使用するので Command は export しておきます。

Command に対応する処理は `editor.registerCommand` で登録します。 `LexicalEditor` を受け取って unregister 関数を返す関数 `registerCodeLanguageSelecting` を定義します。React hooks を使う関数の命名は `useXXX` とするように、 `registerCommand` や `registerUpdateListener` を呼ぶ関数の命名は `registerXXX` に統一しておくといいでしょう。また、戻り値も必ず `() => void` 関数にするルールにすると迷うことなく扱えます。

実装は単純で、 `RangeSelection` が指している Node が `CodeNode` ならばそれを、そうでなければ親方向に `CodeNode` を探索して見つけたらそれをターゲットとして `setLanguage` メソッドを `editor.update` の中で実行します。 `registerCommand` のコールバック関数の戻り値は `boolean` を求められるので、更新した場合は `true` を、していない場合は `false` を返すようにしておきます。

Command には Priority(優先度) の設定ができて、それが `registerCommand` の第 3 引数に当たります。Priority を比較するための specific な名前のついた定数は `lexical` パッケージに含まれています。同じ Command に対して複数の処理を登録している場合、 Priority を見て Lexical が実行してくれます。今回は自作の `CODE_LANGUAGE_COMMAND` に対して 1 つの処理しか登録しないので何でもいいです。

そしたら、定義した `registerCodeLanguageSelecting` を `editor` に反映させるようにしましょう。

```tsx:src/plugins/CodeHighlightPlugin.tsx
export const CodeHighlightPlugin: FC = () => {
  const [editor] = useLexicalComposerContext();

  useEffect(() => {
    return mergeRegister(
      registerCodeHighlighting(editor),
      registerCodeLanguageSelecting(editor)
    );
  }, [editor]);

  return null;
};

// 省略
```

公式の関数である `registerCodeHighlighting` と同様に `useEffect` 内で使用します。ただ、複数の unregister 関数を何回も実行するのは面倒なので、 `@lexical/utils` で提供される `mergeRegister` を使いましょう。これで `registerXXX` をラップしてやることで複数の unregister 関数を 1 個の関数にまとめてくれます。それを `useEffect` のクリーンアップとすることで不整合がなくなります。

続いてツールバーにドロップダウンリスト(`select`, `option`)を配置します。まずは サポートしている言語の数だけ `option` 要素を生成する元データを定義します。

```tsx:src/plugins/ToolbarPlugin.tsx
const CodeLanguagesOptions = Object.entries(CODE_LANGUAGE_FRIENDLY_NAME_MAP).map(
  ([value, label]) => ({ value, label })
);
```

`@lexical/code` パッケージにサポートしている言語のキーと表示名の組み合わせを持つ `CODE_LANGUAGE_FRIENDLY_NAME_MAP` があるので、それを `Object.entries` で `{value, label}` 配列に変換します。

キャレット位置があるコードブロックが現在選択している言語のステートを宣言しておきましょう。これを `select` の `value` props に渡して制御コンポーネントとします。

```tsx:src/plugins/ToolbarPlugin.tsx
export const ToolbarPlugin: FC = () => {
  const [codeLanguage, setCodeLanguage] = useState("");

  // 省略
};
```

先にビューを作っておきます。 `blockType` が `"code"` のときだけ表示されるドロップダウンリストとします。

```tsx:src/plugins/ToolbarPlugin.tsx
export const ToolbarPlugin: FC = () => {
  // 省略

  return (
    <div className={styles.toolbar}>
      {/*省略*/}
      {blockType === "code" && (
        <div className={styles.select}>
          <select
            aria-label="code languages"
            value={codeLanguage}
            onChange={event =>
              editor.dispatchCommand(CODE_LANGUAGE_COMMAND, event.target.value)
            }>
            <option value="">select...</option>
            {CodeLanguagesOptions.map(item => (
              <option key={item.value} value={item.value}>
                {item.label}
              </option>
            ))}
          </select>
          <MdExpandMore />
        </div>
      )}
    </div>
  );
};
```

事前に宣言していた通り、 `value` には `codeLanguage` ステートを渡します。 `onChange` では `setCodeLanguage` に値をセットするのではないことに注意してください。 `editor.dispatchCommand` で先程の `CODE_LANGUAGE_COMMAND` を dispatch することで Lexical の `EditorState` の更新をします。

Lexical の `EditorState` が更新されれば `editor.registerUpdateListener` が動くので、そこで `setCodeLanguage` で値を取得します。

```tsx:src/plugins/ToolbarPlugin.tsx
useEffect(() => {
  return editor.registerUpdateListener(({ editorState }) => {
    editorState.read(() => {
      if ($isHeadingNode(targetNode)) {
        // 省略
      } else {
        if ($isCodeNode(targetNode)) {
          setCodeLanguage(targetNode.getLanguage() || "");
        }

        // 省略
      }
    });
  });
}, [editor]);
```

`CodeNode` のメソッドに `getLanguage` があるのでこれで取得した値を `setCodeLanguage` に渡します。ちょっとややこしいですが、 Lexical のステートを source of truth としたいので、ビュー → Lexical → useState → ビュー の流れにします。Lexical が framework-agnostic ゆえに、 React のレンダリングを発火できないので `useState` を挟む必要があるのですね。

これでコードブロックの言語選択機能の実装は完了です。

![言語を切り替えるドロップダウンリストでCSSが選択され、コードブロックに入力されたCSSコードがシンタックスハイライトされている](/images/lexical-rich-editor-trial/08_editor_sample.png)

ちなみにサポートされている言語の数は今の所かなり少ないです。

https://github.com/facebook/lexical/blob/6e8c81c0ba741200ac5c76121d1989ea0210ff26/packages/lexical-code/src/CodeHighlightNode.ts#L23-L65

増やすには `import 'prismjs/components/prism-javascript'` のような Prism.js の言語定義ファイルを適当に import して `CODE_LANGUAGE_FRIENDLY_NAME_MAP` を拡張することになると思います。ただ、 Prism.js の作りが悪くて import の順番を間違えるとうまくいかないという問題があるので注意が必要です(そもそも [import するのが間違っている](https://github.com/PrismJS/prism/issues/2617#issue-735192448)という話もあります)。

### インラインスタイル機能を実装する

インラインレベルのスタイルを付け外しできる機能を実装しましょう。といっても `RichTextPlugin` を使っているだけですでにショートカットキーが実装されているので、現時点でエディター上に入力されたテキストを何文字か選択し、 cmd+B(ctrl+B) を押すとボールドになります。

Lexical が標準でサポートしているインラインスタイルは次の 7 種類です。

- bold
- underline
- strikethrough
- italic
- code
- subscript
- superscript

https://github.com/facebook/lexical/blob/6e8c81c0ba741200ac5c76121d1989ea0210ff26/packages/lexical/src/nodes/LexicalTextNode.ts#L73-L80

標準以外のインラインスタイル(カラフルなテキストにするなど)を付けたい場合は、Node クラスの独自実装が必要になりそうです。

ここで実装するのはボタンクリックによって選択したテキストをインライン装飾する機能です。リッチテキストエディターのインライン装飾でよくあるのが、テキストを選択するとそのテキストの近くにフローティングするパネルが出現する UI です。が、今回は~~めんどくさい~~簡単のためにブロックレベルのツールバーの下に固定でインラインツールバーを配置しましょう。

`InlineToolbarPlugin` というコンポーネントを用意します。そして、現在の `Selection` が指す Node がそれぞれのインラインスタイルを持っているかのステートを宣言します。インラインスタイルはブロックタイプと異なり重複して適用できるので、各スタイルが当たっているかどうかの `boolean` をひとつずつ宣言します。

```tsx:src/plugins/InlineToolbarPlugin.tsx
export const InlineToolbarPlugin: FC = () => {
  const [editor] = useLexicalComposerContext();

  const [isBold, setIsBold] = useState(false);
  const [isUnderline, setIsUnderline] = useState(false);
  const [isStrikethrough, setIsStrikethrough] = useState(false);
  const [isItalic, setIsItalic] = useState(false);
  const [isCode, setIsCode] = useState(false);
  const [isSubscript, setIsSubscript] = useState(false);
  const [isSuperscript, setIsSuperscript] = useState(false);

  return <div className={styles.inlineToolbar}></div>;
};
```

クリックすることでテキストを装飾するボタンを作ります。テキストを装飾するには、 `FORMAT_TEXT_COMMAND` という Command と一緒にインラインスタイル名を dispatch してやります。 `FORMAT_TEXT_COMMAND` は `lexical` パッケージから import できます。

```tsx:src/plugins/InlineToolbarPlugin.tsx
export const InlineToolbarPlugin: FC = () => {
  // 省略

  return (
    <div className={styles.inlineToolbar}>
      <button
        type="button"
        aria-label="format bold"
        role="checkbox"
        aria-checked={isBold}
        onClick={() => editor.dispatchCommand(FORMAT_TEXT_COMMAND, "bold")}>
        <MdFormatBold />
      </button>
      <button
        type="button"
        aria-label="format underline"
        role="checkbox"
        aria-checked={isUnderline}
        onClick={() => editor.dispatchCommand(FORMAT_TEXT_COMMAND, "underline")}>
        <MdFormatUnderlined />
      </button>
      {/*以下似通ってるので省略*/}
    </div>
  );
};
```

これで、各ボタンをクリックすることで選択しているテキストのインラインスタイルを変換することができます。

この `InlineToobarPlugin` も `Editor.tsx` で呼び出しておきましょう。

```tsx:src/Editor.tsx
// 省略
import { InlineToolbarPlugin } from "./plugins/InlineToolbarPlugin";

// 省略

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      <ToolbarPlugin />
      <InlineToolbarPlugin />
      {/*省略*/}
    </LexicalComposer>
  );
};
```

最後に、各インラインスタイルが適用されているかどうかを更新する部分です。今回も `editor.registerUpdateListener` を使用して、 `EditorState` が変化したら毎回実行されるリスナーを登録しておきます。 `Selection` は `EditorState` に含まれるので、その変化も検知することができます。

```tsx:src/plugins/InlineToolbarPlugin.tsx
useEffect(() => {
  editor.registerUpdateListener(({ editorState }) => {
    editorState.read(() => {
      const selection = $getSelection();

      if (!$isRangeSelection(selection)) return;

      setIsBold(selection.hasFormat("bold"));
      setIsUnderline(selection.hasFormat("underline"));
      setIsStrikethrough(selection.hasFormat("strikethrough"));
      setIsItalic(selection.hasFormat("italic"));
      setIsCode(selection.hasFormat("code"));
      setIsSubscript(selection.hasFormat("subscript"));
      setIsSuperscript(selection.hasFormat("superscript"));
    });
  });
}, [editor]);
```

`RangeSelection` が `selection.hasFormat` という、インラインスタイルをチェックするメソッドを持っています。これを実行した結果を各ステートの更新関数に渡してやることで、各インラインスタイルが適用されているかどうかを確認できます。

インラインスタイルに対する CSS は次のように適用します。

```ts:src/editorTheme.ts
export const theme: EditorThemeClasses = {
  // 省略

  text: {
    bold: styles.textBold,
    code: styles.textCode,
    italic: styles.textItalic,
    strikethrough: styles.textStrikethrough,
    subscript: styles.textSubscript,
    superscript: styles.textSuperscript,
    underline: styles.textUnderline,
    underlineStrikethrough: styles.textUnderlineStrikethrough,
  },
};
```

`"underline"` と `"strikethrough"` が同時に当てられた時専用のクラスが用意されているようです。 CSS で `text-decoration` を複数指定したい場合、

```css
.textUnderlineStrikethrough {
  text-decoration: underline line-through;
}
```

このようにひとつのプロパティに複数指定するように書く必要があることから、 `underlineStrikethrough` というクラス名が用意されていると考えられます。

![7つすべてのインラインスタイルが適用されたテキストを持つエディター](/images/lexical-rich-editor-trial/09_editor_sample.png)

`"subscript"` と `"superscript"` は重複して良いのかはさておき、インラインスタイルも実装できました。

### マークダウンライクなショートカットを導入する

エンジニアのみなさんは普段からマークダウン形式でドキュメントを残すことが多く、書き慣れているかと思います。また、作業効率を重視すると、いちいちツールバーにマウスを移動してクリックなんて面倒だと思うかも知れません。そんなときにマークダウンライクなテキストを入力するとブロックタイプが変換されたりインラインスタイルが適用されると嬉しいのではないでしょうか。

嬉しいことにその機能も公式プラグインで実装されていて、プラグインを差し込むだけで利用できるようになっています。

```tsx:src/plugins/MarkdownPlugin.tsx
import { FC } from "react";
import { TRANSFORMERS } from "@lexical/markdown";
import { MarkdownShortcutPlugin } from "@lexical/react/LexicalMarkdownShortcutPlugin";

export const MarkdownPlugin: FC = () => {
  return <MarkdownShortcutPlugin transformers={TRANSFORMERS} />;
};
```

`MarkdownShortcutPlugin` にマークダウンライクな入力を検知してスタイルを変換する機能が実装されています。 props として渡されている `TRANSFORMERS` は変換ルールの配列で、 `@lexical/markdown` からはより細かく指定するための、インラインスタイルだけ変換できるルールとかブロックレベルだけ変換できるルールなどが用意されています。また、 `Transformer` 型オブジェクトを自分で用意してやれば、オリジナルの変換ルールも実装できます。

`MarkdownPlugin` を `Editor` に差し込みます。

```tsx:src/Editor.tsx
// 省略
import { MarkdownPlugin } from "./plugins/MarkdownPlugin";

export const Editor: FC = () => {
  return (
    <LexicalComposer initialConfig={initialConfig}>
      {/*省略*/}
      <MarkdownPlugin />
    </LexicalComposer>
  );
};
```

![# を入力するとHeadingに、> を入力するとQuoteに、``` が入力されるとCodeに変換される様子](/images/lexical-rich-editor-trial/10_editor_sample.gif)

マークダウンのような書き心地にできましたね！

### 完成版(再掲)

この記事ではここまでで完成にしておきます。完成したエディターは次の URL にデプロイしてあるので試してみてください。

https://y-hiraoka.github.io/lexical-rich-editor-trial/

めちゃくちゃ余談ですが、デプロイには先日ベータ版がリリースされた [actions/deploy-pages](https://github.com/actions/deploy-pages) を使ってみました。

## まとめ

この記事では Lexical を使って簡単なリッチテキストエディターを作ることで Lexical の API や使い方について学びました。

需要がありそうな機能は公式プラグインとして豊富に提供されているのが嬉しいですね。

[Lexical Playground](https://playground.lexical.dev/) にはもっとたくさんの機能が実装されたリッチすぎるテキストエディターがデプロイされています。この記事では紹介できなかった画像や YouTube の埋め込みブロック機能などもあります。いずれこのあたりの実装方法も調べたら別記事で紹介したいと思います(予定は未定)。

それではよい Lexical ライフを！
