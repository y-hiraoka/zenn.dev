---
title: "GitHub の草の色を自由に選択できる Chrome 拡張を作った話"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chromeextension", "react"]
published: true
---

Chrome 拡張を作ってみたかったので、GitHub の Commit Graph、日本人エンジニアは草と呼ぶアレの色を自由に変更する Chrome 拡張を作りました。n 番煎じ(n は 2 以上の自然数)なのは承知です。

https://chrome.google.com/webstore/detail/kaleido-github-graph/oadddbdnbhmabamipbcdpljiedghaifh

ソースコードは公開しています。リポジトリはこちら。

https://github.com/y-hiraoka/kaleido-gh-graph

この Chrome 拡張の作成秘話(？)を書き記しておきます。ちなみに新しい内容はまったく含みません。Chrome 拡張で最近新たにできるようになったこととかは存じ上げないし、React の最新機能を Chrome 拡張でこんな風に使える！的な話でもありません。本当にただの作ってみたブログです。

## 概要

カラーピッカー UI から色を選択して保存してから GitHub のユーザーページを開くと、Commit Graph の色が指定した色で表示されます。Commit Graph は 0 ~ 4 の 5 段階で色が変化するので、ユーザーに指定してもらった色をそれっぽく 4 段階に変換して表示します(0 レベルはオリジナルのままです)。

また Gaming Mode にチェックを入れると、Commit Graph がゲーミング PC のように虹色に光ります。面白い！！！！

![GitHub の Commit Graph がゲーミングPCのように七色に変化するアニメーションgif](/images/about-kaleido-github-graph/gaming-commit-graph.gif)

Chrome 拡張の名前や説明文は全部 ChatGPT に提案してもらいました。Kaleidoscope って英単語かっこいいですね。

## 使用技術

### Vite

https://vitejs.dev/

言わずと知れた爆速ビルドツールです。Chrome 拡張は通常の Web サイトと同じく HTML,CSS,JS で動くので、Vite でビルドすることができます。テンプレートはもちろん react-ts で、React の Single Page Application として作りました。

### CRXJS

https://crxjs.dev/vite-plugin

Vite のビルドを Chrome 拡張向けに制御してくれる Vite Plugin です。TypeScript から Chrome 拡張 manifest を生成できたり、manifest の内容からビルドの起点を決めて必要な数のビルドを走らせてくれたりします。

また Chrome 拡張開発中の Hot Module Replacement も提供しています。が、Chrome 拡張のアイコンをクリックして表示されるポップアップは一度閉じないと変更が反映されませんでした。また各タブに対して CSS や JS を挿入する Content Scripts の変更では無関係のタブまでリロードされていました。開発は続けられているようなので改善されるのを期待します。

## 実装方法

### 選択した色は HSL で管理する

色は HSL 空間で管理することにしました。HSL は Hue(色相)、Saturation(彩度)、Lightness(明度) の 3 つの値で色を表現する方法です。Lightness を 4 段階にずらすことで Commit Graph の 4 つのレベルを表現できると思った^[GitHub の本当の草の色は Lightness をずらすだけの単純な表現ではありませんでした。]のと、カラーピッカーが直感的に表現できることが理由になります。Lightness は選択してもらう必要がないので、横軸が Hue、縦軸が Saturation を表す平面から 1 点を選択してもらうカラーピッカーになりました。

### カラーピッカーの実装方法

Hue は 0 ~ 360 の整数値を取ります。Saturation は 0 ~ 100% の値を取ります。なのでカラーピッカーのコンポーネントは `width: 360px; height: 100px;` で作りました。Chrome 拡張のポップアップサイズ的にちょうどよかったので固定サイズです。

カラーピッカーは横方向に Hue のグラデーションを、縦方向に Saturation のグラデーションを写したいのですが、CSS では 2 方向のグラデーションを実現できません。そこで、横方向に Hue のグラデーションを写した要素の上に、縦方向に透明からグレー(`hsl(0, 0, 50%)`)にグラデーションした要素を重ねて表現することにしました。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/src/components/ColorPicker.module.scss#L12-L37

また、カラーピッカーの操作性に少しこだわって実装しました。

色の選択は `mousedown`, `mousemove`, `mouseup` イベントでマウス位置を取得することでカラーピッカー上のどの色を選んだかを計算します。ここで、カラーピッカーの `div` の`mousemove` をリッスンしてしまうと、マウスカーソルがカラーピッカーの外に出たときに色が選択できなくなります。そこで、`mousemove` と `mouseup` イベントは `document` のものをリッスンするようにしました。これでマウス操作で選択状態がつっかえる感じがなくなり体験悪化を回避できます。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/src/components/ColorPicker.tsx#L63-L85

もちろんキーボード操作も可能にしました。マーカー部分(適切な名称かは不明)は `div` ですが `tabindex="0"` をつけることでフォーカス可能にし、`keydown` で矢印キーによる色の選択ができます。また Figma ライクに Shift キーを押しながら矢印キーを押すと 10 ずつ移動できるようになっています。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/src/components/ColorPicker.tsx#L110-L144

### ステート管理

選択した色、または Gaming Mode にチェックを入れたかどうかのステートをどこかに保存する必要があります。これは Chrome 拡張環境で使える `chrome.storage.sync` で永続化することにしました。このストレージに保存されたデータは同じ Google アカウントでログインしている Chrome 全てで共有されます。容量や書き込み回数の制限は厳しいですが、今回は Hue と Saturation の 2 つの値、または Gaming Mode を選択しているかどうかを保存するだけなので余裕で使えます。

Chrome 拡張のポップアップを開き直した時、カラーピッカーの初期値は最後に選択した色を指していてほしいですよね。`chrome.storage.sync.get` は非同期で値を取得するので素直に `useState` の引数に渡すことができません。かと言ってこのためだけに `useSWR` とか `react-query` をインストールするのは嫌でした。また `useEffect` でゴニョゴニョするのも負けた気がして嫌でした(？)。ということで自分で `Promise` をぶん投げています。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/src/PickedColor.ts#L23-L36

`Promise` を投げることで非同期処理の結果を同期的に扱える React はとても便利ですね(？)

### 実際に草の色を変更する部分

GitHub のユーザーページをブラウザの開発者ツールで眺めていたら草の色を CSS Variables で管理しているのが確認できました。

```scss
--color-calendar-graph-day-L1-bg: #9be9a8;
--color-calendar-graph-day-L2-bg: #40c463;
--color-calendar-graph-day-L3-bg: #30a14e;
--color-calendar-graph-day-L4-bg: #216e39;
```

Chrome 拡張からユーザーが閲覧している HTML に JavaScript や CSS を注入する Content Scripts の機能で、この CSS Variables を上書きできれば Commit Graph の見た目を変更できるとわかりました。固定のスタイルを当てたいだけなら manifest.json の `content_scripts` に `css` を指定するだけでよかったのですが、今回はユーザーが選択した色を使いたいので `js` を指定しています。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/vite.config.ts#L19-L24

そして `js` で指定されたファイルの拡張子は `tsx` となっていますね。はい、スタイルを当てるためだけに React アプリをマウントしています。単に `style.setProperty` を実行するスクリプトを仕込むだけで完了できればよかったのですが、Gaming Mode で CSS Variables を徐々に変化する必要があり、`@keyframes` を使いたかったのです。なので React アプリとしてマウントして、`@keyframes` を含む `style` 要素を `head` 要素に挿すような構成にしています(純粋な JS を書きたくなかっただけという噂もあり)。

React アプリを直接 `head` にマウントできれば良いなと思ったのですが、 `createRoot` は指定した要素の子要素を全て削除してしまうので、例えば次のような書き方だと GitHub が本来提供する `head` 要素の中身を吹き飛ばしてしまいます。

```tsx
const root = document.querySelector("head");
createRoot(root).render(<App />);
```

ですので一度 `body` に空の `div` を挿入してそこに React アプリをマウントします。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/src/style-injector.tsx#L5-L15

そして `StyleInjector` コンポーネントが `head` に `style` 要素を挿入します。ここで使用するのが `createPortal` です。`createPortal` を使えば好きな HTMLElement の子要素にコンポーネントを挿入することができ、その子要素を削除することはありません。これで GitHub がもともと提供する `head` の子要素を維持したまま `style` 要素を別途追加することができました。

https://github.com/y-hiraoka/kaleido-gh-graph/blob/f43612880fd260347182f54dd84012a1eb3250c3/src/components/StyleInjector.tsx#L6-L39

また、ポップアップ側で色を変更した時にすでに開いている GitHub ページをリロードしなくてもいいように `chrome.storage.sync.onChanged` で `storage` の変更に即座に反応できるようにしています。少しでも体験を損ねないためにですね。

## まとめ

GitHub の草の色を自由に選択できる Chrome 拡張の実装についてお話しました。自分は Chrome 拡張を作るのが初めてだったので微妙な点があるかもしれませんが、参考になれば幸いです。

この ~~おもちゃ~~ Chrome 拡張をリリースするために Google に $5 支払いました！！！！！ぜひ使ってください！！！！

ちなみにこの記事は ChatGPT には書いてもらっていません。自分で書いてます。

それではよい React ライフを！
