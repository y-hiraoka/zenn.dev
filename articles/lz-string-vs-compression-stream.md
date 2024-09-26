---
title: "続・URLシェアを支える技術 CompressionStream"
emoji: "🔗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "javascript"]
published: false
publication_name: chot
---

以前TypeScript PlaygroundやReact Compiler PlaygroundがURLシェア時のソースコードの圧縮に使用している、lz-stringというライブラリを紹介しました。

https://zenn.dev/chot/articles/what-is-lz-string

すると、jser.infoで有名なazuさんから次のような反応をいただきました。

https://x.com/azu_re/status/1832249554517029209

確かに！

しかし、CompressionStreamは文字列から圧縮済み文字列を返す単純なAPIではないため、lz-stringと同じ使い勝手にするには、薄いラッパーを用意してやる必要があります。

ということで、lz-stringが提供している処理と**近い**ことをCompressionStreamで実現できるかやってみましょう。

## ソースコード、リンク

lz-stringに近い型の関数をこちらに用意しました。

https://github.com/stinbox/lz-string-vs-compression-stream/blob/749d2e30c73d869f3ccfcc232500b39e89988e74/src/compression.ts

そのテストです。

https://github.com/stinbox/lz-string-vs-compression-stream/blob/749d2e30c73d869f3ccfcc232500b39e89988e74/src/compression.test.ts

lz-stringとCompressionStreamの比較サイトを用意しました。

https://stinbox.github.io/lz-string-vs-compression-stream

## lz-stringとの違い

lz-stringは文字列の圧縮処理を同期的に行います。

```tsx:lz-string版
function compressToBase64(input: string): string;
```

しかしCompressionStreamはその名の通りWeb Stream APIなのでどう頑張っても出力は非同期です。なので関数名と引数は同じですが戻り値は`Promise<string>`になります。

```tsx:CompressionStream版
function compressToBase64(input: string): Promise<string>;
```

また、アルゴリズムが違うので互換はありません。今までlz-stringで圧縮・解凍していた箇所をいきなりCompressionStream版に置き換えても機能しませんのでご注意ください。

ちなみに、lz-stringの圧縮アルゴリズムはドキュメントでLZ-basedと書かれており、おそらく作者独自のアルゴリズムです。

CompressionStreamで選択できるアルゴリズム（語弊あり）は `gzip`, `deflate`, `deflate-raw`です。`gzip`はgzipですが、`deflate`はzlibで`deflate-raw`が純粋なdeflateです。命名については歴史的経緯があるようです。

## 前提知識

CompressionStreamはWeb Stream APIというストリーム処理を提供するAPIをベースとしています。

ストリームとは、簡単に言えば「データをちょっとずつ、流れるように順番に処理する」プログラミングのパターンです。CSVファイルや動画ファイルなど、大きなデータでもちょっとずつ処理するのでメモリを圧迫せずに効率よく実行できます。

Web Stream APIについては次の記事が非常にわかりやすかったのでリンクを貼ります。

https://zenn.dev/kojiroueda/articles/e5a18b2c0dc3d4

CompressionStreamは、Web Stream APIのうち**TransformStream**に分類されます。

ちなみに、ストリーム処理とは言うものの、今回作る関数はほとんどストリームの効率性は享受しません。ただComressionStreamの圧縮部分だけを借りることになります。

## 実装

lz-stringが提供している次の関数を、CompressionStreamまたはDecompressionStreamを使って実装します。

- `compressToBase64`
- `decompressFromBase64`
- `compressToEncodedURIComponent`
- `decompressFromEncodedURIComponent`
- `compressToUTF16`
- `decompressFromUTF16`
- `compressToUint8Array`
- `decompressFromUint8Array`

### compressToBase64

base64形式で出力する関数を次のように書きます。

```tsx
const textEncoder = new TextEncoder();
const textDecoder = new TextDecoder();

const createUpstream = (value: unknown) => {
  return new ReadableStream({
    start(controller) {
      controller.enqueue(value);
      controller.close();
    },
  });
};

export async function compressToBase64(input: string): Promise<string> {
  const upstream = createUpstream(textEncoder.encode(input));
  const compression = new CompressionStream("deflate");
  const stream = upstream.pipeThrough(compression);
  const compressed = await new Response(stream).arrayBuffer();
  return btoa(
    new Uint8Array(compressed).reduce(
      (acc, c) => acc + String.fromCharCode(c),
      "",
    ),
  );
}
```

`createUpstream()` は最上流の ReadbleStream で、データの開始地点としています。別の関数でも開始地点として同じ記述をするので、関数に括りだしています。見ての通り、一回だけデータを送り出してすぐクローズしているので、ストリームの意味はないです。

`upstream`で流すデータは、`textEncoder.encode()`で文字列をバイナリデータに変換したものになります。CompressionStreamは`Uint8Array`を処理対象としているためです。

`compression`を宣言し、`upstream.pipeThrough`で`upstream`に繋ぎます。これで上流のReadableStreamからCompressionStreamにデータが流れます。その戻り値もReadableStreamです(`compression.reable`にくっついているインスタンスと同じです)。

次の行で唐突にHTTPの`Response`が現れますが、実は`Response.body`はReadbleStreamなのです。ここでは、中身の`arrayBuffer`を簡単に取り出すユーティリティとして使っています。

最後に`arrayBuffer`をbase64に変換して返します。

### decompressFromBase64

`compressToBase64`で圧縮した文字列を解答する処理を書きます。CompressionStreamで圧縮したデータは、同じアルゴリズムのDecompressionStreamで解凍処理ができます。

```tsx
export async function decompressFromBase64(input: string): Promise<string> {
  const compressedBytes = Uint8Array.from(atob(input), (c) => c.charCodeAt(0));
  const upstream = createUpstream(compressedBytes);
  const decompression = new DecompressionStream("deflate");
  const stream = upstream.pipeThrough(decompression);
  const decompressed = await new Response(stream).arrayBuffer();
  return textDecoder.decode(decompressed);
}
```

最初に、引数はBase64で渡されるはずなので、それをバイナリに戻します。

戻したバイナリを`createUpstream`で上流から送り出します。

`decompression`を宣言し、`upstream.pipeThrough`で`upstream`に繋ぎます。これでDecompressionStreamにバイナリデータが流れていきます。戻り値はReadableStreamです。

同じように`Response()`をユーティリティとして利用して、`arrayBuffer`として取り出します。これは解凍済みのバイナリデータです。

最後に、`textDecoder.decode()`によってバイナリデータから文字列に戻します。ここで得られる文字列が、圧縮前の文字列と一致しています(リポジトリに含まれるテストで確認していますので動かしてみてください)。

### compressToEncodedURIComponent

続いてURLセーフな文字列を出力する`compressToEncodedURIComponent`を実装します。といっても、`compressToBase64`を流用できます。

そもそもBase64とは、`a~z`, `A~Z`, `0~9`, `+`, `/` の64文字種と、桁調整のための文字`=`を使って任意のデータを表現するフォーマットです。これら文字種のうち、URLとして使用できないのは`+`, `/`,`=`だけです。この3つさえ別のURLセーフな文字に変換すれば、全体がURLセーフになります（encodedURIComponentとは名ばかり）。

一般的に、Base64をURLセーフにする場合、次のように置換します。

- `+` -> `-` (ハイフン)
- `/` -> `_` (アンダースコア)
- `=` -> 取り除く

この置換方法はMDNにも記載されていました。

https://developer.mozilla.org/ja/docs/Glossary/Base64

ということで、次のような実装になります。

```ts
export async function compressToEncodedURIComponent(
  input: string,
): Promise<string> {
  const withBase64 = await compressToBase64(input);
  return withBase64.replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
}
```

単に`compressToBase64`の結果を`.replace()`しているだけです。

### decompressFromEncodedURIComponent

次に、`decompressFromEncodedURIComponent`を実装します。`compressToEncodedURIComponent`の大元は`compressToBase64`ですので、`decompressFromEncodedURIComponent`も`decompressFromBase64`を流用します。

```ts
export async function decompressFromEncodedURIComponent(
  input: string,
): Promise<string> {
  let base64 = input.replace(/-/g, "+").replace(/_/g, "/");
  while (base64.length % 4) {
    base64 += "=";
  }
  return decompressFromBase64(base64);
}
```

まず引数を純粋なBase64に戻します。それを`decompressFromBase64`に渡すだけです。簡単ですね。

### その他出力形式について

解説できるほど文字のエンコードに詳しくないので、ソースコードを御覧ください（実装が正しいかもわからん）。

https://github.com/stinbox/lz-string-vs-compression-stream/blob/749d2e30c73d869f3ccfcc232500b39e89988e74/src/compression.ts#L35-L87

## lz-stringとCompressionStreamの比較サイトを作りました

https://stinbox.github.io/lz-string-vs-compression-stream

左側のテキストエディターでテキストを編集すると、入力文字列をlz-stringとCompressionStreamの各種関数それぞれで圧縮した圧縮率を表示します。

圧縮率は次の計算式で、lz-string版とCompressionStream版のそれぞれを算出しています。

```
compressToBase64の圧縮率 = compressToBase64(入力文字列).length / btoa(入力文字列).length

compressToEncodedURIComponentの圧縮率 = compressToEncodedURIComponent(入力文字列).length / encodeURIComponent(入力文字列).length

compressToUTF16の圧縮率 = compressToUTF16(入力文字列).length / 入力文字列.length

compressToUint8Arrayの圧縮率 = compressToUint8Array(入力文字列).length / new TextEncoder().encode(入力文字列).length
```

各関数は、文字種制限があるなど用途によって使い分けるため、UTF16以外は圧縮率の計算も圧縮なしでそれら文字種やデータ型に変換したもののサイズを分母としています。

そして、このサイトはTypeScript PlaygroundやReact Compiler Playground同様、圧縮した文字列をURLに埋め込むことで、テキストエディターの入力状態をシェアできるようにしています。面白いですね。遊んでみてください。

入力文字列にもよりますが、CompressionStreamのほうが圧縮効率が良いように感じますね。僕の実装があっていればだけど。

## ランタイムについて

CompressionStreamは[Web API](https://developer.mozilla.org/ja/docs/Web/API)なので、モダンブラウザであれば利用可能です。

ブラウザ以外のランタイムでは、使えない場合があります。Node.jsはv18から使えます。[Cloudflare Workers](https://developers.cloudflare.com/workers/runtime-apis/web-standards/#compression-streams)も使えるようです。その他JSランタイムで使用する場合は、サポート状況をご確認ください。

## まとめ

CompressionStreamを使って、lz-stringの使い勝手に近い圧縮処理を実装しました。lz-stringは同期処理ですが、CompressionStreamベースの実装は非同期処理になる点には注意が必要ですが、代替実装として使えると思いました。

lz-stringとCompressionStreamはアルゴリズムが異なるため、互換性はありません。lz-stringの置き換えではなく、新規で圧縮処理が必要なときに検討してみてください。

lz-stringはnpmライブラリですが、CompressionStreamは[Web API](https://developer.mozilla.org/ja/docs/Web/API)の一つなのでラッパー関数の分しかバンドルサイズに含まれません。フロントエンドでも気軽に使えますね。

サイトでlz-stringとCompressionStreamの圧縮率も比較できますので、興味があれば遊んでみてください。

それでは鱸ᬁ꾉臣Ꞝ룧ꦃ苣閃苣膼䰉됒
