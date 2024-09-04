---
title: "URLシェアを支える技術 lz-string"
emoji: "🔗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "javascript"]
published: false
publication_name: chot
---

WebアプリでURLシェアを実装する際に、URLにすべての情報を持たせてしまいたい場合があります。そのとき、情報をそのままクエリ文字列に渡してしまうとURLの文字数制限に引っかかってしまうかもしれません(厳密にはURLに上限はないようですが、現実はいつもブラウザ実装依存)。

そんなときURLセーフな文字列形式で圧縮してくれるライブラリがあります。lz-sringです。

https://github.com/pieroxy/lz-string

## 変換の例

ライブラリで `compressToEncodedURIComponent` というAPIが提供されているのでこれを使用します。標準の`encodeURIComponent`でURLセーフな文字列に変換した場合とサイズ比較をしてみましょう。

```jsx
import lzstring from "lz-string";

const rawData =
  "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.";

console.log(encodeURIComponent(rawData).length); // 素のencodeURIComponentの場合: 589文字

const compressed = lzstring.compressToEncodedURIComponent(rawData);

console.log(compressed.length); // lz-stringを使用した場合: 407文字
```

元のサイズの7割程度まで小さくなっているのがわかります(圧縮対象の文字列によって変わります)。

これならURLに持たせられる情報量を増やすことができますね。

共有先のページでURLから取り出したデータの復元には`decompressFromEncodedURIComponent`を使用します。

## 身近なツールで使用されている例

### TypeScript Playground

[TypeScript Playgroud](https://www.typescriptlang.org/play)は簡易的なソースコード共有機能があります。URLにtsconfig.jsonの設定値やソースコードの情報を載せて、URLを開くだけで同じ状態を別のブラウザで復元できるものです。例↓

[https://www.typescriptlang.org/play/?moduleResolution=99&target=99&jsx=4&module=199#code/GYVwdgxgLglg9mABANRgEwKZwBQG9EBu6WiAvgJSK4BQiiAThlCPUtrXYgDxowEB8HTtwAqACxABbAEZgAhjAA2hYnAC8uIpjilEAekHC6XOYjGNgGrVgB0LRaUNHjYgMz9Nqm7CiKMpLj03J2cuAAcPazgbTABnCHoYMNgEAL0IoU5AuRCsgBkYAGsMACEQKCgEFW0rVV0DTMDeAQ5yAG5qUiA](https://www.typescriptlang.org/play/?moduleResolution=99&target=99&jsx=4&module=199#code/GYVwdgxgLglg9mABANRgEwKZwBQG9EBu6WiAvgJSK4BQiiAThlCPUtrXYgDxowEB8HTtwAqACxABbAEZgAhjAA2hYnAC8uIpjilEAekHC6XOYjGNgGrVgB0LRaUNHjYgMz9Nqm7CiKMpLj03J2cuAAcPazgbTABnCHoYMNgEAL0IoU5AuRCsgBkYAGsMACEQKCgEFW0rVV0DTMDeAQ5yAG5qUiA)

このソースコード部分の圧縮かつURLセーフな文字列への変換にlz-stringが使用されています。

その該当コードが以下です。

https://github.com/microsoft/TypeScript-Website/blob/ac68b8b8e4a621113c4ee45c4051002fd55ede24/packages/sandbox/src/compilerOptions.ts#L130

もともとTypeScript PlaygroundがソースコードのURLシェアをどのように実装しているか気になり調べた結果、lz-stringにたどり着きました。

### React Compiler Playground

React CompilerもPlaygroundを提供していて、同じくURLシェア機能があります。URLをクリックするだけで共有元と同じソースコードがエディターに入力された状態でWebページが開きます。例↓

[https://playground.react.dev/#N4Igzg9grgTgxgUxALhAMygOzgFwJYSYAEAangCYIQAUwRAbhVUQL4CURwAOsUTAjljFqPImKIAecnnoA+UeLESAKgAsoAWwBGmAIZ4ANgyYQAvMEaUILIgHp5vRRN1FV-NOctUAdLAMsHRSdVAGZZCxNvfBwDBBYJW1DAoKUAB3CvCG9KMDgYPFT8QnjbdIUnW11kpwAZPABrBAAhKBwcQmMrTxMbe3LJW2k5BTYAbh4WEBYgA](https://playground.react.dev/#N4Igzg9grgTgxgUxALhAMygOzgFwJYSYAEAangCYIQAUwRAbhVUQL4CURwAOsUTAjljFqPImKIAecnnoA+UeLESAKgAsoAWwBGmAIZ4ANgyYQAvMEaUILIgHp5vRRN1FV-NOctUAdLAMsHRSdVAGZZCxNvfBwDBBYJW1DAoKUAB3CvCG9KMDgYPFT8QnjbdIUnW11kpwAZPABrBAAhKBwcQmMrTxMbe3LJW2k5BTYAbh4WEBYgA)

このソースコード圧縮にもlz-stringが使用されていることが確認できました。

https://github.com/facebook/react/blob/d1afcb43fd506297109c32ff462f6f659f9110ae/compiler/apps/playground/lib/stores/store.ts#L8-L35

全然関係ないけどReact Compiler PlaygroundはTypeScriptで実装されているんですね。Flowじゃなかった。

## 他のAPI

ライブラリはJavaScriptで書かれていますが、`.d.ts`ファイルが同梱されています。それを引用。名前と型定義で明らかですね。

https://github.com/pieroxy/lz-string/blob/45cfe5e365c3e435f2465acad1c43ebb681b203f/typings/lz-string.d.ts

`compressTo~` で圧縮したデータは同じ形式の`decompressFrom~`で復元できます。

開発者はもともと `localStorage` に保存するデータを増やす目的でlz-stringを開発したそうです。

`compress`が生成する文字列はinvalidなUTF-16のようで、動作確認はwebkitブラウザの`localStorage`でしか行われていないようです。`compressToUTF16`ならすべてのブラウザでテスト済みとのこと。

本記事冒頭に出てきた`Lorem ipsum...`サンプルテキストを、戻り値がstring型である関数でそれぞれ圧縮すると次のような結果になります。

```js
console.log(lzstring.compress(rawData).length);
// 153
console.log(lzstring.compressToUTF16(rawData).length);
// 164
console.log(lzstring.compressToBase64(rawData).length);
// 408
console.log(lzstring.compressToEncodedURIComponent(rawData).length);
// 407
```

`compress`が最も圧縮効率が良く、続いて`compressToUTF16`が効率が良いです。`compressToBase64`と`compressToEncodedURIComponent`は効率が悪いですが、URLなど文字種に制約がある場合には選択せざるを得ないでしょう。

## 使い時

- LocalStorageに保存する
  開発者がもともと希望していた使い方です。データを`JSON.stringify`でstring型にした後 `compressToUTF16`を使用することで、容量を節約しつつLocalStorageに保存できます。LocalStorageから取り出したら`decompressFromUTF16`で復元しましょう。
- URLの一部にする
  `compressToEncodedURIComponent`を使用することでURLの一部として利用できる文字列だけを使って圧縮できます。パスパラメーターにも、クエリパラメーターにも、URLハッシュにも安全に使える文字列になります。TypeScriptとReact CompilerのPlaygroundと同じ目的です。
- HTMLの一部にする
  リッチテキストエディターなどを開発中、それが吐き出すHTMLにまるっとステートを持たせておきたいことがありました。そのときに`compressToBase64`で変換して`data-`属性にぶち込む方法で乗り切りました。（多分`compressToUTF16`でも問題ないが未確認）

## 注意点

lz-stringは圧縮ツールであり、暗号化や改ざん防止機能は付いていません。圧縮後の文字列は人間可読ではありませんが、平文と変わらないと言えます。

機密データを共有する目的に使ってはいけません。また、`decompressFrom~`で復元したデータが意図した形式になっているかのチェックは必要です。

そして、シェアされたURLを復元して画面に表示するならば、クロスサイトスクリプティングの余地ができていないか必ず確認しましょう(lz-stringに限った話ではないですね)。

## まとめ

JavaScriptの文字列圧縮ライブラリとしてlz-stringを紹介しました。

大きなデータをURLだけでシェアしたい場合に利用できます。LocalStorageでも使えます。

TypeScriptやReact CompilerのPlaygroundで、ソースコードをURLだけでシェアする機能の実装にlz-stringが使われています。

lz-stringは圧縮ツールであり、暗号化や改ざん防止機能はありません。

それでは！
