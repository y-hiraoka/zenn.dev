---
title: "Tailwind CSS の Plugin を駆使してグリッドレイアウトの auto-fill するやつ作る"
emoji: "🍱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tailwindcss"]
published: true
publication_name: chot
published_at: 2023-12-06 10:00
---

![特に追加の情報がなく視覚的インパクトを与えるのが目的のキャッチ画像](/images/grid-template-auto-fill-with-tailwind/eyecatch.png)

ちょっと株式会社 Advent Calendar 2023 12 月 6 日の記事です。

https://adventar.org/calendars/8910

## こんなレイアウトありますよね

![グリッドレイアウトを例示するgifアニメーション。画面幅を徐々に大きくしていくと、グリッドのカラム数が自動で増加していく。ただしカラムの幅は最低240pxが維持されている](/images/grid-template-auto-fill-with-tailwind/grid-template-auto-fill-example.gif)

1 カラムは 240px 以上を維持しつつ、横幅に入るだけのカラム数を自動で増やしてくれるようなグリッドレイアウトです。画面幅ではなく親要素の幅によってカラム数が決まるので、コンポーネント指向の Web アプリ開発に向いていて重宝しています。

### CSS で書く場合

CSS では次のように `display: grid` を使います。

```css
.autoFill {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 24px;
}
```

`grid-template-columns` の値が複雑で覚えにくいですね。愛用しているくせに毎回調べています。

### Tailwind CSS でそのまま書くなら

標準のユーティリティには用意されていないので、Arbitrary values を使います。CSS の覚えにくい文字列をそのまま `[]` に渡します。

```tsx
<div className="grid grid-cols-[repeat(auto-fill,minmax(theme(spacing.60),1fr))] gap-6">
```

余計覚えにくくなった気がします。VSCode の Tailwind CSS IntelliSense を使っていても入力補完が効かないため、やはり調べて入力するしかありません。また、上記コード例では Tailwind CSS の数値トークンを利用するため、`theme(spacing.60)` という記法を使っています。普通は `w-60` などと書くのであまり馴染みがないかもしれません。これがさらにクラスを分かりにくくしたり、そもそも Arbitrary values の中なのでトークンとして管理されていない `240px` のような直接指定もできてしまいます。

## Tailwind Plugin を作ろう

Tailwind CSS は Plugin で自作のユーティリティクラスを自由に追加できます。今回取り上げている自動カラムレイアウトは、ユーティリティにするのにふさわしい粒度だと思います。

次のような指定ができると嬉しいのではないでしょうか。

```tsx
<div className="grid grid-cols-auto-fill-60 gap-6">
```

`grid` とともに `grid-cols-auto-fill-60` を指定します。これは `grid-cols-2` などが `grid` と一緒に指定されることに倣っています(`grid-cols-auto-fill-60` に `display:grid` を含めないことで粒度を揃える)。

また、`-60` の部分は Tailwind CSS が用意してくれる spacing トークンを使う想定です。デフォルトでは 1=4px なので、`-60` は 240px を指定していることになります。これで、 `w-60` と同じ感覚でグリッドカラムの最小幅を指定可能です。

### こんな Plugin を用意する

`matchUtilities` を使うことで、動的な値を指定するユーティリティを追加できます。

```tsx:tailwind.config.ts
import { Config } from "tailwindcss";
import plugin from "tailwindcss/plugin";

const config: Config = {
  content: ["./src/**/*.{js,ts,jsx,tsx}"],
  theme: {},
  plugins: [
    plugin(({ matchUtilities, theme }) => {
      matchUtilities(
        {
          "grid-cols-auto-fill": (value) => ({
            gridTemplateColumns: `repeat(auto-fill, minmax(${value}, 1fr))`,
          }),
        },
        { values: theme("spacing") }
      );
    }),
  ],
};

export default config;
```

`matchUtilities` 関数の第 2 引数で指定する `values` に `theme("spacing")` を指定しています。これによって、`w-` と同じトークンからユーティリティが作成されます。`matchUtilities` の第 1 引数では、自作ユーティリティの名前と、`theme("spacing")` の実際の値を受け取って CSS を返す関数をセットで指定します。

これを `tailwind.config.ts` に追加しておけば、`grid-cols-auto-fill-60` などのユーティリティが使えるようになります。

```tsx
<div className="grid grid-cols-auto-fill-60 gap-6">
```

また、VSCode の Tailwind CSS Intellisense で入力補完してくれるようになります！

![Visual Studio Code で自作のユーティリティクラス grid-cols-auto-fill-n が Intellisense で一覧されている様子](/images/grid-template-auto-fill-with-tailwind/vscode-intellisense.png)

ついでに `auto-fit` 版なども作っておくと良いかもしれませんね。

## まとめ

Tailwind CSS の自作 Plugin で自動カラムレイアウトのユーティリティクラスを作る方法をご紹介しました。他にも色々と自作 Plugin が活かせる場面があると思うので、使いこなしていきたいですね。

それでは良い Tailwind CSS ライフを！

## 参考

https://tailwindcss.com/docs/plugins#dynamic-utilities
