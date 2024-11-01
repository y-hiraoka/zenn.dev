---
title: "2つのJSONの差分を動的に表示する。シンタックスハイライトもする。powered by shiki"
emoji: "🎓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "shiki"]
published: true
publication_name: chot
---

２つのJSON文字列の差分をシンタックスハイライト付きで表示したいケースがありました。

Zennでも同じ差分かつシンタックスハイライトができますね。下のようなコードブロックがそうです。

```diff json: example.json
{
-  "name": "Bob",
+  "name": "Alice",
  "age": 20
}
```

Zennでは行頭に`+`や`-`をつけることで差分としてハイライトされるようになっています。

このようなシンタックスハイライトかつ差分ハイライトを、2つのJSON文字列の差分に対して行いたいと思いました。つまり、差分を表示したい箇所に明示的かつ静的にマークしていくのではなく、2つのテキストから動的に差分を計算して差分ハイライトを表示してくれる機能です。

この記事ではその方法を紹介します。

なお、僕がJSONの差分を表示したかったのでJSONで例を出しますが、好きな言語で、なんならプレーンテキストでも応用可能です。好きなだけdiff表示してください。

## 先出し結論

jsdiffでJSONの差分トークンを取得し、shikiの差分表示対応テキストに変換する。

jsdiff：
https://github.com/kpdecker/jsdiff

shiki：
https://shiki.matsu.io/

サンプルコードのリポジトリ：
https://github.com/stinbox/shiki-diff

## shikiとは

shikiはJavaScriptのシンタックスハイライトライブラリです。

```bash
npm install shiki
```

VS Codeと同じ言語解析エンジンを使用しているので、出力がVS Codeに近いカラーリングになります。テーマや解析可能な言語も豊富です。

また、バンドラーフレンドリーに設計されており、使う分だけのテーマや言語定義をバンドルに取り込めるようになっています。

### shikiの簡単な使い方

`codeToHTML`を使用するだけで、スタイリング済みのHTMLが得られます。あとはこれをブラウザで表示するだけ。style属性で色付けがされているため、CSSの読み込みも不要です。簡単！

```ts
import { codeToHtml } from "shiki";

const code = `const foo = document.createElement("div")`;

const highlighted = await codeToHtml(code, {
  lang: "javascript",
  theme: "light-plus",
});
console.log(highlighted);
// '<pre class="shiki light-plus" style="background-color:#FFFFFF;color:#000000" tabindex="0"><code><span class="line"><span style="color:#0000FF">const</span><span style="color:#0070C1"> foo</span><span style="color:#000000"> = </span><span style="color:#001080">document</span><span style="color:#000000">.</span><span style="color:#795E26">createElement</span><span style="color:#000000">(</span><span style="color:#A31515">"div"</span><span style="color:#000000">)</span></span></code></pre>'
```

ただし上記の方法では全てのテーマや言語定義が内部でロードされているため、ブラウザに届くJavaScriptコードでの使用には向いていません。間違って使ってしまうと、shikiの分だけで1.2MB(gzipped)のバンドルファイルができあがります(！)

細かくバンドルサイズをチューニングするには、`createHighlighterCore`で読み込むテーマや言語定義を細かく指定します。

```ts
import { createHighlighterCore } from "shiki/core";
import { createOnigurumaEngine } from "shiki/engine/oniguruma";

const highlighter = await createHighlighterCore({
  engine: createOnigurumaEngine(import("shiki/wasm")),
  themes: [
    import("shiki/themes/dark-plus.mjs"),
    import("shiki/themes/light-plus.mjs"),
  ],
  langs: [
    import("shiki/langs/javascript.mjs"),
    import("shiki/langs/typescript.mjs"),
  ],
});

const code = `const foo = document.createElement("div")`;

const highlighted = highlighter.codeToHtml(code, {
  lang: "javascript",
  theme: "light-plus",
});
console.log(highlighted);
// '<pre class="shiki light-plus" style="background-color:#FFFFFF;color:#000000" tabindex="0"><code><span class="line"><span style="color:#0000FF">const</span><span style="color:#0070C1"> foo</span><span style="color:#000000"> = </span><span style="color:#001080">document</span><span style="color:#000000">.</span><span style="color:#795E26">createElement</span><span style="color:#000000">(</span><span style="color:#A31515">"div"</span><span style="color:#000000">)</span></span></code></pre>'
```

`createHighlighterCore`は非同期関数ですが、作られた`ShikiHighlightCore`インスタンスのメソッドは同期的に実行できるのも嬉しいです。Reactコンポーネントのレンダリングフェーズでも実行できるので。

### shikiでのdiff表示方法

shikiでdiffを表示するには`transformerNotationDiff`を使用します。これは`@shikijs/transformers`で提供されているので追加でインストールします。

```bash
npm install @shikijs/transformers
```

そして`codeToHtml`の引数に渡すだけ。HTML生成側の準備はこれだけで、とても簡単です。

```ts
import { codeToHtml } from "shiki";
import { transformerNotationDiff } from "@shikijs/transformers";

const code = `...`;

const highlighted = await codeToHtml(code, {
  lang: "javascript",
  theme: "light-plus",
  transformers: [transformerNotationDiff()],
});
```

シンタックスハイライトされるコード側の記法は、追加された行なら`// [!code ++]` を、削除された行なら `// [!code --]`を行末に書いておきます。`transformerNotationDiff`がこれらの文字列を見つけると、差分を識別できるclass(`diff`, `add`など)として`span`に付与してくれます。

```ts
const code = `
const foo = document.createElement("div") // [!code ++]
const bar = document.createElement("span") // [!code --]
`;
```

注意点としては、`@shikijs/transformers`の変換結果にはスタイリングが含まれないことです。なので、差分のための色付けは自前でCSSを用意する必要があります。サンプルコードリポジトリには実際に差分classに対して色付けするCSSのサンプルもあるので参考になれば幸いです[^1]。

[^1]: このサンプルはダークモードの考慮もしているので少し煩雑になっています。それと、TailwindCSSの`@apply`を使っています

https://github.com/stinbox/shiki-diff/blob/53f866b62a9a552b70ae12c6c05c56a5bf802c11/src/components/syntax-highlighter.css

## jsdiffとは

JavaScriptでテキストの差分をトークン列として計算してくれるライブラリです。

https://github.com/kpdecker/jsdiff

jsdiffの`diffLines`関数を使うことで、行単位でどんなテキストが増えてどんなテキストが削除されたかを判断できます。今回は使用しませんが`diffChars`や`diffWords`などもあって、好きな粒度でテキストの差分比較ができるようになります。

### jsdiffの差分トークンを見てみる

`"a\nb\nc\nd\ne"` と `"a\nb\n\cc\ndd\ne"` の2つを`diffLines`に渡して計算した戻り値は次のようになります(`\n`は改行コードです)。

```js
[
  { count: 3, added: false, removed: false, value: "a\nb\nc\n" },
  { count: 2, added: false, removed: true, value: "d\ne" },
  { count: 2, added: true, removed: false, value: "dd\nee" },
];
```

`added`/`removed`の組み合わせで追加されたテキストなのか、削除されたテキストなのか、現状維持のテキストなのかを判断できるようになっています。`value`には複数行がまとめて格納されています。`value`が何行分のテキスト含んでいるかを`count`で判断できます。

この情報があればテキストdiffビューアが実装できそうな気がしますね！

### jsdiffからshikiの差分テキストを組み立てる

jsdiffによって増えた行、減った行が判断できるようになりました。あとはjsdiffのトークン列からshikiのdiff表示用の文字列を組み立てれば、２つのテキストの実際の差分を表示することができますね。

shikiでdiffを表示するには追加行に`// [!code ++]` を、削除行に `// [!code --]` を行末に付与するのでした。それを実現するソースコードはこちら。

```tsx
const diffTextShiki = (oldText: string, newText: string): string => {
  const diffs = diffLines(oldText, newText);

  return diffs
    .reduce((acc, diff) => {
      if (diff.added) {
        const concat =
          acc +
          diff.value
            .split("\n")
            .map((line) => (line ? line + "// [!code ++]" : ""))
            .join("\n");
        return concat.endsWith("\n") ? concat : concat + "\n";
      } else if (diff.removed) {
        const concat =
          acc +
          diff.value
            .split("\n")
            .map((line) => (line ? line + "// [!code --]" : ""))
            .join("\n");
        return concat.endsWith("\n") ? concat : concat + "\n";
      } else {
        return acc + diff.value;
      }
    }, "")
    .trim();
};
```

`value`には複数行分の文字列が格納されているので、それを改行コードで分解してから `"// [!code ++]"` または `"// [!code --]"` を追加しています。終端に改行を含まない2つのテキストを比較する時、最後のトークンの`value`にも改行が含まれないので強制的に`"\n"`を付け足しています(これをしないと崩れる)。

この`diffTextShiki`を使えば、2つのテキストからshikiのdiff表示に使えるテキストに変換できます。この関数の出力結果をshikiの`codeToHtml`に渡せば、diff関連のclass付きHTMLを得られます。

## Reactで組み合わせて使う

Reactでshikiを使用します。

### React Server Componentの場合

React Server Component(RSC)でレンダリングするだけなら、非同期処理が扱えることとバンドルサイズを気にしなくてもよいことから、単に`codeToHtml`を使用しても良いでしょう。

```tsx
import { codeToHtml } from "shiki";

const code = `const foo = document.createElement("div")`;

const Page: React.FC = async () => {
  const highlighted = await codeToHtml(code, {
    lang: "javascript",
    theme: "light-plus",
    transformers: [transformerNotationDiff()],
  });

  return <div dangerouslySetInnerHTML={highlighted} />;
};
```

Next.jsの場合、ランタイムとして`edge`と`node`が選択可能ですが、Edgeランタイムではshikiの読み込みに不具合が起きる可能性があるとして、Node.jsランタイムでの実行がおすすめされています。

https://shiki.matsu.io/packages/next

### Client Componentの場合

必要な分のテーマや言語だけを読み込み可能な`createHighlighterCore`を使用します。`createHighlighterCore`は非同期処理なのでReactで扱うにはひと工夫必要です。ここではReact 19から使える`use`と`Suspense`でshikiをラップしたコンポーネントを紹介します。

まずは`createHighlighterCore`で使いたい分だけのテーマ・言語を読み込んだ`HighlighterCore`インスタンスの`Promise`をモジュールスコープに定義します。`await`しないのがポイントです。

```tsx
import { createHighlighterCore, HighlighterCore } from "shiki/core";
import { createOnigurumaEngine } from "shiki/engine/oniguruma";

const highlighterPromise = createHighlighterCore({
  engine: createOnigurumaEngine(import("shiki/wasm")),
  themes: [
    import("shiki/themes/dark-plus.mjs"),
    import("shiki/themes/light-plus.mjs"),
  ],
  langs: [
    import("shiki/langs/json.mjs"),
    import("shiki/langs/javascript.mjs"),
    import("shiki/langs/typescript.mjs"),
  ],
});
```

モジュールスコープで`createHighlighterCore`を実行することで実行回数を節約します。また、`themes`と`langs`ではdynamic importでバンドルチャンクの分割をしています。static importしたものを`themes`/`langs`に渡すこともできますが、アプリケーションコードに巻き込まれるにはあまりに大きいのでdynamic importをおすすめします(例えば`shiki/wasm`だけでviteビルドかつgzip後サイズが230kBある)。

そして、次のようにネストした2つのコンポーネントを用意します。

```tsx
export const ShikiHighlighter: React.FC<{
  language: string;
  code: string;
}> = ({ language, code }) => {
  return (
    <Suspense
      fallback={
        <div>
          <pre>{code}</pre>
        </div>
      }
    >
      <ShikiHighlighterInner language={language} code={code} />
    </Suspense>
  );
};

const ShikiHighlighterInner: React.FC<{
  language: string;
  code: string;
}> = ({ language, code }) => {
  const highlighter = use(highlighterPromise);

  const highlighted = highlighter.codeToHtml(code, {
    lang: "javascript",
    theme: "light-plus",
    transformers: [transformerNotationDiff()],
  });

  return <div dangerouslySetInnerHTML={{ __html: highlighted }} />;
};
```

親コンポーネントのほうは子コンポーネントを`Suspense`でくくることが役目です。また、`Suspense`の`fallback`に`<pre>` 要素で`code`を表示することで、shikiがロード完了するまではハイライトされていないコードを代替表示できます。

子コンポーネントは、モジュールスコープの`Promise<HighlighterCore>`から、`React.use`を使って`Promise`の中身を取り出します。shikiのロードが完了していない間はサスペンドし、親の`Suspense`に待機してもらいます。shikiのロードが完了していれば、`HighlighterCore`インスタンスを取得できます。`HighlighterCore`インスタンスのメソッドは同期処理であるため、レンダリングフェーズで実行して、結果を`dangerouslySetInnerHTML`に渡すことで描画完了です！

親コンポーネントである`ShikiHighlighter`だけをexportしています。シンタックスハイライトコンポーネントを使いたい側からすれば、関心があるのは`ShikiHighlighter`だけです。`<ShikiHighlighter />` に色付けしたいコードとその言語の種類を渡すだけでよく、サスペンドするかどうかや、`dangerouslySetInnerHTML`を使っていることも隠蔽します。

### diff用のテキストは使う側で用意する

`diffTextShiki`は`codeToHtml`でも`ShikiHighlighter`でも有効な文字列を生成します。使う側で差分のテキストを生成し、`codeToHtml`や`ShikiHighlighter`にわたすだけでよいです。

ここまで用意すれば、次のようなリアルタイムに差分を表示するサイトも作れます。

https://stinbox.github.io/shiki-diff/#/diff?codeold=eJx9U8tuwjAQvOcrVj5wajZvXicO_Y5KIdmAIcSR7YRWiH57ZZsklFJu8cxoPLMbXzwA1uQnYmtgas-P3C95VbE3g7eS97k2lJYdWagnqbhojDrEEEMn1F-tNTiJsqvJYaqQvNWKreHiAQCwknqj6bl2CgC27XhdGlCrAvwtzGZgaHD4TVTzRhsNKfMFOOCtpJ7TefCE4ewBXG2CklpqSmoKTncxNrblQQVa5o2qhDyRNDT7iDCOMRrsN6aUCuw0DJthjMlAjuhiHAIAk5QX2lktMZmsLO6X4vQvJ0WnSY6SOcaLydYGnhKOeJ_XfCvchSGmJvtd9_79eX03xuDgOq8wSibHW-fHHo8zedVnY3ZxUEFbdzve-E6qzoWVJphNl-WdFq2kin-SdEYhphiPvAv6JKUj_F837IU4qtuewtdSSZUktR_GhtG41V0ttnl9-xkyjKLJqBVKF8pRS0wxXQyMznl95k05sImxTEfWDMy-BEN-ZzjH-C_n33VdYnRXwL4WVyvFld2vd_V-AHP--ks&codenew=eJyVU8tupDAQvPMVLQ45rXtsHvPIKYf9jkgeaDKeMBjZhskqyn77yjavZKOV9gZV5XJXNbwnAGknb5Q-Qmov6lWxWjVN-sPjvVGjdJ5yZqAAjWSs0p1Xc-TIo9D96oPBTddDSxGzlVG9s-kjvCcAAGlNo9eMyhEw1mvjIOc8OgCk50G1tRc4WwE7w8MDBGnEJ1GrOuc1ZP0T4Iz3hkZF98V_fk8APsI0NfXU1dRVijYjPYXEV7tzRna20eZGxtPps8AsQzHbP_mAdhea8WyJGeYzuaCHpRCA1JCsXLQ6eZyZilWSDvuiIpbxrBA8Kz-JWa1v_3fA6MGRWc7tMTusA4Roa5YFH2WrzjqOxrHwKTctjT-_LyoWvrvGdk4o8tVxameT-Ig5iq_tfQkZNIvEb-1qd307vKiORam9V0GaY7leJgene0ONeiMTjTgWmC18HPSbKSPBPt1w0frVThvl_5YaagzZy1wbimX_L60-y3b6bEoUYjXqtXWVjdQRCywOM-Okau-qq2c295bFwvrCwv_jyd8l7jH7m2ObrEcUmwC-zSlWgaew3-Qj-QPWRwhy&lang=json

![shiki-diff](/images/two-json-diff-with-syntax-highlight/shiki-diff-website.gif)

楽しい！

## まとめ

2つのJSONの差分を表示するために、jsdiffとshikiを組み合わせて使う方法を紹介しました。jsdiffで差分トークンを取得し、shikiの差分表示対応テキストに変換することで、2つのJSONの差分をシンタックスハイライト付きで表示できるようになります。

この方法を使えば、Zennのような差分表示ができるだけでなく、シンタックスハイライトもできるので、コードの変更箇所をわかりやすく表示できます。ぜひお試しください！

それでは良いshikiライフを！
