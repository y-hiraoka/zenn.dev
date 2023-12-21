---
title: "【Next.js】Google Analytics も YouTube iframe 埋め込みも公式ライブラリでいけるようになるぞ"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
published_at: 2023-12-24 11:00
publication_name: chot
---

!!!!!!!!!!!!!!!!!!!!!!!!(ここにアイキャッチ画像)!!!!!!!!!!!!!!!!!!!!!!!!

ちょっと株式会社 Advent Calendar 2023 12 月 24 日の記事です。

https://adventar.org/calendars/8910

みなさんこんにちは、chot Inc. の Web エンジニアです。

Next.js で Google Analytics を導入するとき、どうしていますか？僕は毎度「nextjs google analytics」でググって「こうやるのか〜」と適当に作っています。本当にちゃんと計測されているのか疑心暗鬼です。

また、YouTube の iframe 埋め込みはどうでしょう。普通に iframe を埋め込むと PageSpeed Insights のスコアをごっそり奪っていきます。恐ろしいですね。

これらのサードパーティリソースを SPA である Next.js に導入するには注意点やベストプラクティスがいくつもあります。しかしそれらをすべて把握するのは大変でしょう。それを解決するために、サードパーティリソースを Next.js に導入するための公式ライブラリがリリースされていました。本記事はそのライブラリ @next/third-parties のご紹介です。

## @next/third-parties

@next/third-parties はサードパーティのスクリプトや iframe を Next.js に簡単に導入するためのライブラリです。執筆時点(2023/12/22)で提供されているのは次のサービスたちです。

- Google Tag Manager
- Google Analytics
- Google Maps Embed
- YouTube Embed

https://nextjs.org/docs/app/building-your-application/optimizing/third-party-libraries

Google Analytics については、バージョン指定なしのインストールで追加される最新版に含まれていませんでしたので、今すぐ試す場合は canary 版をインストールしてください。

```bash
npm install @next/third-parties@canary
```

### 公式提供の経緯を見る

https://github.com/vercel/next.js/discussions/48256

Next.js のリポジトリで @next/third-parties 提供を検討する Discussion が作成されていました。それによれば、実に 95％の Next.js 製サイトがサードパーティのリソースを読み込んでいるとのこと。よく使われるサードパーティリソースを公式サポートすることで、Next.js ユーザーの開発者体験の向上と、効率的なリソース読み込みによるエンドユーザー体験の向上を狙っているようです。

実際、Google Analytics 導入は面倒だし YouTube の埋め込みはすぐにパフォーマンスを悪化させます。開発者にとってのつらさもユーザーにとっての煩わしさも同時に解決してくれる手段を提供してくれるのは嬉しいですね。

## 使ってみる

実際に Google Analytics と YouTube Embed を使ってみました。Google Tag Manager と Google Maps は省略します。なぜなら僕が必要になったことがないからです(？)

### Google Analytics

まずは Google Analytics についてです。[ドキュメント](https://nextjs.org/docs/app/building-your-application/optimizing/third-party-libraries)にも記載がありますが、サイトで Google Tag Manager を使っている場合はそれ経由で Google Analytics を導入できるため、@next/third-parties の Google Analytics サポートは不要です。あくまで単独で Google Anayltics を導入する場合に使うコンポーネントです。

使い方はめちゃくちゃ簡単で、`GoogleAnalytics` コンポーネントに Google Analytics のトラッキング ID を渡すだけです。

```tsx
import { GoogleAnalytics } from "@next/third-parties/google";

const RootLayout: FC<{ children: ReactNode }> = ({ children }) => {
  return (
    <html>
      <head>
        <GoogleAnalytics gaId="G-XXXXXXXXXXX" />
      </head>
      <body>{children}</body>
    </html>
  );
};

export default RootLayout;
```

上記コードでは App Router の `app/layout.tsx` で使っている想定ですが、特定のパス配下でのみ計測したい場合は、対応する `layout.tsx` に `<GoogleAnalytics />` を移動するだけで良いです。`head` の中である必要もありません。プロダクション環境でのみ計測したい場合はコンポーネントをマウントしないようにするだけです。

```tsx
<head>
  {process.env.NODE_ENV === "production" && (
    <GoogleAnalytics gaId="G-XXXXXXXXXXX" />
  )}
</head>
```

また、独自のイベント送信もサポートされています。特定のボタンのクリックを計測したい場合は、`sendGAEvent` が使えます。

```tsx
"use client";

import { sendGAEvent } from "@next/third-parties/google";

export const GAButton: FC = () => {
  return (
    <button
      onClick={() =>
        sendGAEvent({
          event: "buttonClicked",
          value: "xyz",
        })
      }
    >
      Send GA Event
    </button>
  );
};
```

少し話が逸れるのですが、Next.js (または SPA)での Google Analytics 導入手順を紹介する記事でよく「SPA は HTML を一度しか読み込まないので、ページ遷移時にページビューイベントを能動的に送信する必要がある」と説明されていることがあります。僕もそうだと思っていましたし、[Next.js リポジトリに含まれる Examples](https://github.com/vercel/next.js/blob/14052c052e0408021973fa9d2e0f2797f3a53b29/examples/with-google-analytics/pages/_app.js) にもそんなサンプルコードが残っていたりします。

しかし、@next/third-parties の `GoogleAnalytics` コンポーネントは能動的なページビューイベントの送信を行っていません。というのも、GA4 はすでに SPA での計測にも対応していて、スクリプトを読み込んでおくだけで SPA のページ遷移でも計測を行ってくれているようです。むしろ今能動的にページビューイベントを送信している SPA は二重に計測してしまっているかもしれません。僕のサイトでは二重に計測していたようでした。

Google Analytics コンソールで「管理 > データストリーム > ウェブストリームの詳細 > 拡張計測機能」の「ブラウザの履歴イベントに基づくページの変更」にチェックが入っていたら SPA でもページ遷移を計測しているので、確認してみるとよいかもしれません。

![Google Analytics コンソールのスクリーンショット。「拡張計測機能」が表示され、「ページビュー数」という項目のブラウザの履歴イベントに基づくページの変更」にチェックが入っている状態](/images/introduction-of-next-third-parties/ga-console-screenshot.png)

@next/third-parties の `GoogleAnalytics` を実際に使って計測しているサイトはこちらです(個人ブログの宣伝)。ちゃんとページビューの計測が出来ていることが確認できています。

https://blog.stin.ink/

個人ブログでは `GoogleAnalytics` を全ページで有効にするために `app/layout.tsx` でマウントしています。ソースコードは次の箇所。

https://github.com/y-hiraoka/stin-blog/blob/5285c7af933c7696b19c356da2b4f583014a1de9/src/app/layout.tsx#L38

公式ドキュメントのコード例や僕の個人ブログは App Router ですが、`GoogleAnalytics` の内部実装はただの next/script なので Pages Router でも問題なく使用できます。

### YouTube Embed

YouTube Embed も使い方は超簡単で、`YouTubeEmbed` コンポーネントに埋め込みたい動画の `videoId` を渡すだけです。

```tsx
import { YouTubeEmbed } from "@next/third-parties/google";

const Page: FC = async () => {
  return <YouTubeEmbed videoid="Ux2k6X-sh8k" />;
};

export default Page;
```

`videoId` は YouTube の URL に含まれていて、次のような箇所に埋まっている値です。

```
https://www.youtube.com/watch?v={videoId}
https://youtu.be/{videoId}
https://www.youtube.com/embed/{videoId}
```

:::details 任意の URL から videoId を取得する

YouTube の動画 URL は種類が多くて厄介ですね(？)

どんな URL からでも `videoId` を取得できる関数を用意したのでお役に立てれば幸いです(考慮漏れがあったらすみません)。

```tsx: 実装
export function extractYouTubeVideoId(url: string): string | null {
  const matched =
    /^https?:\/\/(www\.)?youtube\.com\/watch\?(.*&)?v=(?<videoId>[^&]+)/.exec(
      url
    ) ??
    /^https?:\/\/youtu\.be\/(?<videoId>[^?]+)/.exec(url) ??
    /^https?:\/\/(www\.)?youtube\.com\/embed\/(?<videoId>[^?]+)/.exec(url);

  if (matched?.groups?.videoId) {
    return matched.groups.videoId;
  } else {
    return null;
  }
}
```

```tsx: テストコード
import { describe, expect, it } from "vitest";
import { extractYouTubeVideoId } from "./extractYouTubeVideoId";

describe("extractYouTubeVideoId", () => {
  it("通常の視聴ページURLから videoId を取得する", () => {
    const videoId = extractYouTubeVideoId("https://www.youtube.com/watch?v=cyFB7sB6CYs");

    expect(videoId).toBe("cyFB7sB6CYs");
  });

  it("www なしの通常の視聴ページURLから videoId を取得する", () => {
    const videoId = extractYouTubeVideoId("https://youtube.com/watch?v=cyFB7sB6CYs");

    expect(videoId).toBe("cyFB7sB6CYs");
  });

  it("短縮URLから videoId を取得する", () => {
    const videoId = extractYouTubeVideoId("https://youtu.be/cyFB7sB6CYs");

    expect(videoId).toBe("cyFB7sB6CYs");
  });

  it("埋め込みURLから videoId を取得する", () => {
    const videoId = extractYouTubeVideoId("https://www.youtube.com/embed/cyFB7sB6CYs");

    expect(videoId).toBe("cyFB7sB6CYs");
  });

  it("プレイリストURLから videoId を取得する", () => {
    const videoId = extractYouTubeVideoId(
      "https://www.youtube.com/watch?v=cyFB7sB6CYs&list=PLQJNT2fdCJngOJF9JBwv_EbEkOBJnkJ_M",
    );

    expect(videoId).toBe("cyFB7sB6CYs");
  });

  it("他のパラメーターを含むURLから videoId を取得する", () => {
    const videoId = extractYouTubeVideoId(
      "https://www.youtube.com/watch?si=1234567890&v=cyFB7sB6CYs",
    );

    expect(videoId).toBe("cyFB7sB6CYs");
  });

  it("不正なURLから null を取得する", () => {
    const videoId = extractYouTubeVideoId("https://www.youtube.com/");

    expect(videoId).toBeNull();
  });

  it("ドメインが全く異なるURLから null を取得する", () => {
    const videoId = extractYouTubeVideoId(
      "https://zenn.dev/stin/articles/about-dp-soundlibrary",
    );

    expect(videoId).toBeNull();
  });
});
```

:::

`YouTubeEmbed` は内部で [lite-youtube-embed](https://github.com/paulirish/lite-youtube-embed) が使用されています。lite-youtube-embed は Google Chrome チームに所属するエンジニアさんが開発したライブラリで、通常の YouTube iframe よりもレンダリングが 224 倍速いというカスタム要素(`lite-youtube`)を提供します。

lite-youtube-embed の挙動としては、最初は iframe ではなく動画のサムネイル画像とボタン要素を描画しておき、ボタンがクリックされて初めて iframe を描画する動作をします。これによって、**視覚的な**体感読み込み速度を改善しているようですね。

デモサイトに `YouTubeEmbed` を使ったページと通常の iframe を使ったページを用意しました。

https://stinbox.github.io/next-third-parties-test/

PageSpeed Insights で２つのページの測定結果を比較してみます。

- 通常の iframe を使ったページ
  ![PageSpeed Insights 結果のスクリーンショット。パフォーマンス:31点、ユーザー補助:94点、おすすめの方法:95点、SEO:100点](/images/introduction-of-next-third-parties/psi-normal-iframe.png)
- `YouTubeEmbed` を使ったページ
  ![PageSpeed Insights 結果のスクリーンショット。パフォーマンス:95点、ユーザー補助:100点、おすすめの方法:100点、SEO:100点](/images/introduction-of-next-third-parties/psi-next-third-parties.png)

読み込み速度が段違いに改善されていますね！

#### `YouTubeEmbed` の注意点

`YouTubeEmbed` 使ってみていくつか気づいたことや気になることがありました。@next/third-parties というよりは lite-youtube-embed 起因が多いですが。

##### `style` が `React.CSSProperties` ではなく文字列なのが気持ち悪い

React を使っていると `style` 属性はオブジェクト型(`React.CSSProperties`)で渡せるのが当然と思ってしまうのですが、`YouTubeEmbed` の `style` props は string 型です。`YouTUbeEmbed` の props を `lite-youtube` 要素の属性にそのまま渡すためだと思われます。

```tsx
<YouTubeEmbed videoid="Ux2k6X-sh8k" style="max-width: 100%;" />
```

lite-youtube-embed が提供する CSS の `max-width: 720px` 指定を上書きしたり、`background-image` を差し替えたいことがある(後述)ので `style` をよく使うことになります。その度に「ｳｯ」となります。オブジェクト型から文字列への変換は `YouTubeEmbed` の中で吸収してほしいな〜。

##### youtube-nocookie.com が使われること

YouTube 埋め込み URL には `www.youtube.com/embed` と `www.youtube-nocookie.com/embed` の２種類があり、後者はプライバシー強化モードと呼ばれるものです。`lite-youtube` 要素は後者を使用します。変更する理由はないとは思いますが、手段もありません。 YouTube Premium 会員に広告が表示されるかどうかは資料が見つからず不明です(僕は Premium 会員ですが nocookie でも広告は表示されませんでした)。

##### アスペクト比が 16/9 固定なこと

lite-youtube-embed が提供する CSS によって `lite-youtube` 要素のアスペクト比が 16/9 に固定されています。
https://github.com/paulirish/lite-youtube-embed/blob/f9fc3a2475ade166d0cf7bb3e3caa3ec236ee74e/src/lite-yt-embed.css#L33

`lite-youtube` に当たるスタイルを頑張って上書きすることで解消できそうですが、全てのコンポーネントに適用されてしまうので微妙です。`class` を渡すことで詳細度を上げてスタイルを上書きできればまだマシですが、現状 `YouTubeEmbed` 経由では `class` を渡せません。
最近だとスマホユーザーをターゲットにして縦型動画を公開している動画配信者もいるので、困るケースがありそうです。

##### background-image の画質が粗いこと

YouTube のサムネイル画像はサイズのバリエーションがありますが、`lite-youtube` にデフォルトで表示されるのは `hqdefault` という横幅 480px の画像です。スマホで閲覧する際は気にならないですが、PC サイズに広がるとガビガビに見えてしまいます。これは `style` で `background-image` を指定することで解消はできます。

```tsx
<YouTubeEmbed
  videoid="Ux2k6X-sh8k"
  style="background-image: url('https://i.ytimg.com/vi/Ux2k6X-sh8k/maxresdefault.jpg');"
/>
```

しかしこれではスマホサイズ閲覧時に無駄に大きな画像をダウンロードさせることになります。本物の iframe 内に表示されるサムネイルは、横幅に合わせて勝手にサムネイル画像を切り替えているようなので、lite-youtube-embed でもやってくれたら嬉しいな〜と思っています(できるのか？)。

## @next/third-parties の注意点

### tsconfig.json の `moduleResolution`

Next.js プロジェクトの `tsconfig.json` で `moduleResolution: "node"` となっているとコンパイルエラーになります。これは @next/third-parties が package.json の `exports` フィールドでモジュールを指定しているため、`moduleResolution: "node"` なプロジェクトからは読み込めないからです。コンパイルを通すには、`moduleResolution: "bundler"` を指定する必要があります。僕は JavaScript のモジュール周りは全く詳しくないので、次の記事を貼り付けるに留めて、より詳しい言及は避けます…。

https://blog.s2n.tech/articles/dont-use-moduleresolution-node

最近の create-next-app は最初から `moduleResolution: "bundler"` が指定された `tsconfig.json` を生成しますが、古くから動いている Next.js プロジェクトだともしかしたらまだ `moduleResolution: "node"` になっているかもしれません。僕の個人ブログは `"node"` でした。時間を溶かしました(？)

### 絶賛 Experimental です

@next/third-parties をインストールすると v14 で降ってくるので安定しているのかと誤解されそうですが、Next.js 本体とモノレポになっているから同じメジャーバージョンになっているだけで絶賛 Experimental の絶賛開発中とのことです。

ただ、これは個人的見解ですが、ひとつひとつは小さいラッパーモジュールだし散々再発明されたものだと思うので、そんなに破壊的変更が加えられることはないのではと思います。あってもすぐ修正できるレベル。

それ以上に、対応されるサードパーティサービスがどんどん増えていくのが楽しみですね！

## まとめ

Next.js プロジェクトにサードパーティリソースを導入するための公式ライブラリ @next/third-parties を紹介しました。

Google Analytics はコンポーネントひとつで直感的に入れられるようになるし、YouTube Embed は視覚的な読み込み速度が大幅に改善されるのでどんどん活用できそうですね。Google Tag Manager と Google Maps Embed は試していませんが、ドキュメントを見る限りこれらもコンポーネントひとつで簡単に導入できるようになっているようです。Next.js チームが開発者体験を重視していることが伺えます。素敵。

今後このライブラリにどんなサードパーティリソースが追加されるかも、とても楽しみです。

それでは良い Next.js ライフを！メリークリスマス 🎄
