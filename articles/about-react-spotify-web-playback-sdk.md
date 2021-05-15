---
title: "Spotify Web Playback SDK の React ラッパーライブラリを作った話"
emoji: "🎺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "spotify"]
published: true
---

# React らしい API を目指して

Spotify はサードパーティアプリで音楽を再生できるようにするためのさまざまなプラットフォーム向けの SDK を公開しています。

ブラウザ JavaScript 用の [Web Playback SDK](https://developer.spotify.com/documentation/web-playback-sdk/) もあるのですが、あまり React アプリで扱いやすいとは言えない API になっていました(特定の UI ライブラリ or フレームワークに特化した SDK ではないので当然ではあります)。そこで今後の自分の開発のためにも React アプリ内で扱いやすい形にしたいなと思い、 ラッパーライブラリの作成・公開を決めました。

ライブラリ名は react-spotify-web-playback-sdk で各種リンクは以下のとおりです。

GitHub Repository

https://github.com/y-hiraoka/react-spotify-web-playback-sdk

デモアプリ

https://react-spotify-web-playback-sdk.vercel.app

デモアプリではあなたのデータを保存したり転送したりはしていませんので安心して Spotify アカウントを連携してみてください。ただし、音楽を再生するためには Spotify プレミアム(有料)の加入が必須です(無料アカウントはログインまでなら可能です)。

この記事では Web Playback SDK を React アプリで使用する際の問題点と、ラッパーライブラリでの解決方法を解説していきます。

以降、 Spotify 公式が提供する Web Playback SDK を単に SDK、 react-spotify-web-playback-sdk のことをラッパーライブラリと呼びます。

## ラッパーライブラリが解決していること

### SDK の読み込みについて

#### 問題

SDK はオープンソース化されておらず、公開も npmjs.com ではなく Spotify が構えている CDN 経由になっています。

つまり SDK を Web アプリに導入するためには HTML の `<body>` に script タグを挿入する必要があります:

```html
<body>
  <!-- ... -->
  <script src="https://sdk.scdn.co/spotify-player.js"></script>
</body>
```

しかし React 開発者にとって HTML を直接編集することはあまり好ましくないかもしれません。 Next.js に至っては、 HTML ファイルがソースコードにありません。React ではコンポーネントに `<script>` を忍ばせても実行されることもありません。

`<script>` で読み込むのではなく npm 経由で取得したモジュールから import で利用できるのが理想ではないでしょうか。

#### 解決方法

ラッパーライブラリはもちろん npm でインストールが可能になっています。

```
npm install react-spotify-web-playback-sdk
```

npm でインストールすることで package.json でライブラリの存在を確認できるようにもなります。

ラッパーライブラリはルートコンポーネント内の `useEffect` によって SDK のスクリプトをロードしています。これによって開発者は HTML に script タグを直書きしたり独自に script を読み込む処理を記述する必要がなくなる上に、ルートコンポーネントがマウントされて初めて SDK の読み込みが発生するので `index.tsx` などで気を使う必要もありません。 SDK を読み込んでいるコンポーネントがアンマウントされた場合、 `<script>` の削除も行っています。

実は Web Playback SDK に関心を持つまで外部スクリプトを React アプリ内で読み込む機会がなく、 HTML に直書きする以外にその方法を知りませんでした。ちょうどそのタイミングで Zenn の開発者である catnose さんの下記投稿を拝見して script タグの取り扱いについて学びました。

https://zenn.dev/catnose99/articles/329d7d61968efb#js%E3%83%95%E3%83%AC%E3%83%BC%E3%83%A0%E3%83%AF%E3%83%BC%E3%82%AF%E3%81%A7%E3%83%84%E3%82%A4%E3%83%BC%E3%83%88%E3%82%92%E5%9F%8B%E3%82%81%E8%BE%BC%E3%82%80

### SDK の初期化処理について

#### 問題

スクリプトが読み込まれて SDK が利用可能になると、スクリプト内部で `window.onSpotifyWebPlaybackSDKReady` が実行されることになっています。 `window.onSpotifyWebPlaybackSDKReady` を生やしておき、そこで初期化完了フラグみたいなものを立てる処理を書くことになると思いますが、 `window` オブジェクトに新しい関数を生やすのは気が引けるし(僕だけ？)、HTML を直接編集している場合は関数を生やすタイミングがわからないかと思います。(`index.tsx` で間に合うのだろうか。未検証です)

なにより、React アプリケーションにとっては初期化完了フラグみたいなものはリアクティブな値であってほしいと思うのではないでしょうか。

#### 解決方法

ラッパーライブラリではルートコンポーネント(`<WebPlaybackSDK>`)内で SDK の読み込みと `window.onSpotifyWebPlaybackSDKReady` の用意を吸収しています。

`<WebPlaybackSDK>` は `<Context.Provider>` に相当するコンポーネントでもあります。 例えば react-router なら `<BrowserRouter>` 、 Recoil なら `<RecoilRoot>` などと同様にルートに配置することによって、下位のコンポーネントがライブラリが提供するステートや機能を利用することが可能になります。(厳密には完全にアプリのルートに位置する必要はなく、ライブラリのカスタムフックを利用したい範囲のコンポーネントツリートップという意味でルートと表現しています)

```tsx
import { WebPlaybackSDK } from "react-spotify-web-playback-sdk";

const MyApp = () => {
  return (
    <WebPlaybackSDK deviceName={deviceName} getOAuthToken={getOAuthToken}>
      <ChildComponent />
    </WebPlaybackSDK>
  );
};
```

初期化が完了したかどうか(`window.onSpotifyWebPlaybackSDKReady` が実行済みかどうか)の `boolean` フラグはカスタムフックで取得する形式にしています。

```tsx
import { useWebPlaybackSDKReady } from "react-spotify-web-playback-sdk";

const MyComponent = () => {
  const webPlaybackSDKReady = useWebPlaybackSDKReady();

  if (!webPlaybackSDKReady) return <div>Loading...</div>;

  return <div>Player is ready!</div>;
};
```

これでグローバルオブジェクトに新しい関数を生やす必要もないし、使い慣れたカスタムフックで初期化フラグを扱うことが可能となります。

### Spotify.Player クラスについて

#### 問題

SDK は `Spotify.Player` クラスただひとつを提供しており、インスタンス化したオブジェクトのメソッドを実行することで音楽の再生やポーズを実現できます。

しかし、 React でクラスインスタンスを扱うのはひと工夫が必要です。関数コンポーネントの中で愚直に `new Spotify.Player()` と書くと、再レンダリングのたびにインスタンス生成してしまうことになります。他のクラスでは毎度インスタンスが変わっていてもたまたま動くかもしれませんが(もちろんそうすべきではありませんが)、Spotify Web Playback SDK に関しては 1 インスタンスにつき 1 つの ID が割り当てられるためうまく再生できない不具合が発生します。

#### 解決方法

`useMemo` だったり `useRef` を使えばクラスインスタンスが一つであることを保証するのはそれほど難しくはありませんが、ラッパーライブラリで施されていれば気をつける必要もありません。

ラッパーライブラリでは、常に同じインスタンスを返すカスタムフックを用意しました:

```tsx
import { useSpotifyPlayer } from "react-spotify-web-playback-sdk";

const MyPlayer = () => {
  const player = useSpotifyPlayer();

  if (player === null) return null;

  return <button onClick={() => player.togglePlay()}>toggle play</button>;
};
```

ちなみに `useSpotifyPlayer` は SDK が提供する `Spotify.Player` クラスのインスタンスをそのまま渡しています。ネイティブの機能にアクセスできることが便利なこともあるだろうと思いこの形にしています。

ただ、今後 ~~気が変わって~~ 設計を再検討して、 `player.togglePlay` などのプレーヤーを操作するメソッドだけを含んだ別のオブジェクトを返すようにするかもしれません。理由としては、 `Spotify.Player` クラスには `player.addListener(eventType, callback)` というメソッドが備わっているのですが、どうやら 1 種類のイベントタイプに対して 1 つのコールバック関数しか登録できないようで、ライブラリ内部ですべてのイベントタイプを使い切っているためアプリ層で書き換えられると予期せぬ不具合の原因になるだろうと予想していることです。

### PlaybackState について

#### 問題

`PlaybackState` はその名の通りプレーヤーの再生状態を表すオブジェクトです。現在再生している曲の情報、ポーズしているかどうか、スキップしたら再生される曲の情報などがプロパティに含まれています。これは `player.addListener("player_state_changed", callback)` のコールバックで受け取れる他、 `player.getCurrentState()` でも取得することができます。

しかし、 React 使い的にはステートオブジェクトはカスタムフックからリアクティブな値が取れることを望まれるのではないでしょうか。コールバック関数に `useState` の setter を登録しておけば実現はできますが、最初から `useXXX` があればいいと思うでしょう、そうでしょう。

また、もうひとつ問題があって、 `player.addListener` に渡したコールバック関数は、音楽の再生開始や停止、スキップなどでは実行されますが、1 曲を連続して聴いている数分間は実行されることはありません。しかし `PlaybackState` には再生中の position(ミリ秒) をプロパティとして含みます。 `player.addListener` で自動的に更新されない以上、下の画像のような音楽プレーヤーによくある UI は実装できません。

![プレーヤー再生位置サンプル](https://storage.googleapis.com/zenn-user-upload/35wt6uazsgk5aq7r5ut9ydh1id5x)
_Spotify ブラウザ版アプリのスクリーンショット_

なので `setInterval` と `player.getCurrentState()` を組み合わせて、実装する必要があります。

#### 解決方法

こちらもカスタムフックを提供して、リアクティブなステートを取得できるようにしました。

```tsx
import { usePlaybackState } from "react-spotify-web-playback-sdk";

const CurrentTrackName = () => {
  const playbackState = usePlaybackState();

  return <div>{playbackState?.track_window.current_track.name}</div>;
};
```

`setInterval` によって `player.getCurrentState()` を叩いてオートでステート更新する処理もラッパーライブラリで行っています。
ただし開発者によっては「曲の現在位置を表示する UI なんて用意しないから余計だよ！」という方もいらっしゃるかもしれません。そのため、 `<WebPlaybackSDK>` の `props` で切替可能にしてあります:

```tsx
import { WebPlaybackSDK } from "react-spotify-web-playback-sdk";

const MyApp = () => {
  return (
    <WebPlaybackSDK
      deviceName={deviceName}
      getOAuthToken={getOAuthToken}
      playbackStateAutoUpdate={false} // 初期値: true
      playbackStateUpdateDuration_ms={500}
    >
      <ChildComponent />
    </WebPlaybackSDK>
  );
};
```

(記事を書いていて思ったのですが、 カスタムフックの引数制御でもよかったと気づきました。)

同時に `playbackStateUpdateDuration_ms` という `props` も用意しています。更新間隔をミリ秒で指定してもらうもので、 そのまま `setInterval` の引数に渡されます。(これも `usePlaybackState` の引数でいいのでは？という気持ちになっています。再検討すると思います。)

## まとめ

Spotify Web Playback SDK のラッパーライブラリを作成して解決した問題についてお話しました。記事を書きつついくつか再検討が必要な箇所も発見しましたが、少しずつアップデートしていければと思っています。

React で Spoitify のサードパーティ Web アプリを作成する場合は、 SDK をそのまま使用するよりも圧倒的に書きやすくなっているはずです。ぜひインストールして使ってみてください。

それでは！
