---
title: "『ポケットモンスター ダイヤモンド・パール』のBGMを無限ループ再生できるサイトを作った話"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "nextjs", "recoil", "chakraui"]
published: true
---

# ポケモン DP の BGM を無限ループで聴こう！

2021 年 12 月 24 日、『ポケットモンスター ダイヤモンド・パール』(以下ポケモン DP)の BGM を視聴できる公式サイト Pokémon DP Sound Library (以下公式サイト) が公開されました。

https://soundlibrary.pokemon.co.jp/

このサイトでは全 149 曲の BGM が公開されており、気分にあったプレイリストなども用意され、ポケモン DP の BGM を存分に楽しむことができます。しかしゲームをプレイしているときとは異なり、途切れなく無限ループ再生されるのではなく、ループ区間を過ぎるとフェードアウトして 1 曲が終了します。

僕はゲームプレイ中のように 1 曲を再生し続けて作業 BGM のように聴きたいなと思いましたが、公式サイト上にその機能は提供されていません。なるほど、ないなら自分で作ろう。

ということで作ったのが次の Web サイトです。

https://dp-soundlibrary.stin.ink/

この記事ではこのサイトの技術的な側面を中心に説明していきます。

## サイトでできること

ポケモン DP の BGM を途切れることなく無限ループで再生できます。音楽プレーヤーらしく、前後のスキップや一時停止も可能です。

各 BGM には公式サイトと同じカテゴリーが設定してあり、カテゴリー別に絞り込むこともできます。

また公式サイトとの差別化として、お気に入り機能を搭載しています。星マークをクリック(タップ)することで BGM がお気に入り一覧に追加されます。ログイン機能は持っていないため、お気に入り情報はそのブラウザにのみ保存されます。

## ソースコード

https://github.com/y-hiraoka/dp-soundlibrary

ソースコードは MIT ライセンスで公開しています。ただ、公式サイトの利用規約によると、音声ファイルの再配布は禁止されています。 GitHub の公開リポジトリに音声ファイルを保存しておくことも再配布に該当するため、渋々音声ファイルを git 管理対象外にしています。

ソースコードをクローンして動作確認する場合は、[ダウンロードページ](https://soundlibrary.pokemon.co.jp/download)からご自身で BGM をダウンロードして[この JSON データ](https://github.com/y-hiraoka/dp-soundlibrary/blob/a84fccc94065fd1bcf56dc4b4dbd0242d0bdd66b/src/data/sounds.ts)にしたがってリネームして保存してください。

```
public/sounds/1.wav
public/sounds/2.wav
...
public/sounds/149.wav
```

## 採用した技術スタック

- Vercel
- Next.js (TypeScript)
- Recoil
- Chakra UI

## デプロイについて

最初は、音声ファイルも含めて GitHub リポジトリに放り込んで GitHub Actions でビルドからの GitHub Pages にデプロイ、というのを想定していました。しかし途中で「公開リポジトリに保存したものを他人が `git clone` することって再配布に該当するのでは？」ということに気づいて考え直しました。

ソースコードを公開することは譲れなかったため、結局手元のターミナルから `git push` とは別に `vercel` コマンドを叩いて都度自力で Vercel にリリースする方式を採用しました。 `vercel` コマンドならローカル PC でビルドしたものを Vercel に直接アップロードするので、音声ファイルがローカル PC にさえ存在すれば正常にデプロイできます。

## お気に入り機能について

BGM が 149 曲もあるので、お気に入りに登録することによってすぐに探し出せるようにしています。

実装は単純で、 `localStorage` に BGM の id リストを入れているだけです。

ただ、 Next.js を使っているため Recoil の `atom` の初期値として `localStorage` から取得した値を入れることはできません(`localStorage` はブラウザ JavaScript の API であるのに対して、 `atom()` は Node.js 上でも実行されるため)。

この事情を考慮するために、ブラウザ側でマウント時に useEffect を発火させて `localStorage` から取り出した値を `atom` に反映させるだけのコンポーネントを用意して `_app.tsx` に仕込みました。ついでに `atom` ステートから `localStorage` に反映させる処理もここでやっています。

```tsx
export const FavoritesEffect: VFC = () => {
  const [favorites, setFavorites] = useRecoilState(favoritesAtom);

  useEffect(() => {
    const storageValue = window.localStorage.getItem(STORAGE_KEY);
    const parsed = JSON.parse(storageValue ?? "[]");
    if (isStringArray(parsed)) {
      setFavorites(parsed);
    }
  }, [setFavorites]);

  // マウント時は `favorites` が必ず空配列であるため useUpdateEffect を使う
  useUpdateEffect(() => {
    window.localStorage.setItem(STORAGE_KEY, JSON.stringify(favorites));
  }, [favorites]);

  return null;
};
```

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/state/favoritesState.ts#L53-L69

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/pages/_app.tsx#L69

中では `useRecoilState` と `useEffect` しか呼んでいないので、コンポーネントではなくカスタムフックでいいのではと思われるかもしれません。これには理由があって、 `useRecoilState` が `RecoilRoot` の内側でしか使用できないのですが、その `RecoilRoot` は `_app.tsx` でセットしているため、 `_app.tsx` で `useRecoilState` を内包するカスタムフックを呼ぶことができません。そのため副作用だけ起こして何も描画しないコンポーネントにまとめて `RecoilRoot` の内側に差し込んでいます。

## Google Analytics について

ページビュー計測については Next.js のお決まりの書き方でやっています。

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/lib/analytics.tsx

今回は BGM リストに置いてあるボタンをクリックしたことによる再生をイベントとしてカウントしています。

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/components/SoundItem.tsx#L16-L22

このイベント計測の集計結果をポケモン DP の BGM 人気ランキングにできるんじゃないかと考えています。投票をするという意味でもぜひ色んな人に使っていただきたいです(懇願)。

## ステート管理について

サーバーからデータをフェッチしてキャッシュして…みたいなものはないので、 Recoil ですべてのステートを持っています。

基本方針として、コンポーネントは一切 `import {...} from "recoil";` をしないようにしています。 `atom` や `selector`、それらを更新する関数はすべてカスタムフックに閉じ込めて適切な名前をつけてから `export` して、コンポーネントからはそのカスタムフックを使用します。

例としてお気に入り機能のステートを軽く説明します。

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/state/favoritesState.ts

ベースとなるステートは BGM の ID を配列として保持する `atom` 一つだけです。これを Single Source of Truth として、 `selector` で派生させたり `useRecoilCallback` で更新します。

```tsx
const favoritesAtom = atom<string[]>({
  key: "favoritesAtom",
  default: [],
});
```

`favoritesAtom` は `export` せずにこのモジュール内からのみ参照できるようにします。これを更新する関数を、 `useRecoilCallback` をラップして作ります。

```tsx
export const useToggleFavorite = () => {
  return useRecoilCallback(
    ({ set }) =>
      (soundId: string) => {
        set(favoritesAtom, (prevState) => {
          if (prevState.includes(soundId)) {
            return prevState.filter((fav) => fav !== soundId);
          } else {
            return prevState.concat(soundId);
          }
        });
      },
    []
  );
};
```

`useRecoilCallback` を使うことで `favoritesAtom` の値を購読することなく(`useSetRecoilState` すら不要)ステート更新が可能です。 `useToggleFavorite` を使うコンポーネントは `favoritesAtom` が変更されても再レンダリングしないということですね。

続いて、指定した BGM の ID がお気に入り登録されているかをチェックできるカスタムフックを作ります。

```tsx
const isFavoriteSoundSelectorFamily = selectorFamily({
  key: "isFavoriteSoundSelectorFamily",
  get:
    (soundId: string) =>
    ({ get }) => {
      const favorites = get(favoritesAtom);
      return favorites.includes(soundId);
    },
});

export const useIsFavoriteSound = (soundId: string) => {
  return useRecoilValue(isFavoriteSoundSelectorFamily(soundId));
};
```

BGM の ID を引数に取る `selectorFamily` の中で `favoritesAtom` の値を取得して、その配列に指定した BGM の ID が含まれているかを return します。
ここでも `selectorFamily` は `export` せずに、それをラップしたカスタムフックを `export` します。名で体を表すのが大切です。
`selectorFamily` を使うことで例えば `useIsFavoriteSound("dp-10")` を実行しているコンポーネントは `favoritesAtom` に変更があったとしても `"dp-10"` の追加削除がなければ再レンダリングをスキップします。

最後にお気に入り登録された BGM だけを含む配列を得られるカスタムフックを用意します(`sounds` が BGM の情報を含むオブジェクトの配列です)。

```ts
export const useFavoriteSounds = () => {
  const favorites = useRecoilValue(favoritesAtom);
  return useMemo(
    () => sounds.filter((sound) => favorites.includes(sound.id)),
    [favorites]
  );
};
```

`useMemo` を使うことで `favorites` が変化したときだけ結果が再計算されます。 `array.prototype.filter` × `array.prototype.includes` と計算量は多いので `useMemo` の効果は大きいです。そうでなくとも、 return する値に無意味な参照の変化がないことも `useMemo` の価値になります。
次の例のように `selector` を使えば、上と同じことを表現することも可能です。

```ts
const favoritesSoundsSelector = selector({
  key: "favoritesSoundsSelector",
  get: ({ get }) => {
    const favorites = get(favoritesAtom);
    return sounds.filter((sound) => favorites.includes(sound.id));
  },
});

export const useFavoriteSounds = () => {
  return useRecoilValue(favoritesSoundsSelector);
};
```

`selector` を使うメリットは、 `atom` の値が変化しても `selector` で計算している値に変化がない場合は `selector` を使用しているコンポーネントが再レンダリングされないことにありますが、今回は「`favoritesAtom` が変化したのに `favoritesSoundsSelector` が変化しない」ということがありえないので前者のよりスッキリした書き方を採用しました。

以上のように Recoil で「少数の `atom` と、それから派生した `selector` や更新関数をカスタムフックに隠蔽してから `export` する」ことでコードを追跡しやすく、かつパフォーマンスを落とさずにステートを管理ことができます。

## 音声ファイルをループ再生する方法について

今回作成した Web サイトの肝の部分になります。ブラウザ JavaScript で音声を扱う技術 [Web Audio API](https://developer.mozilla.org/ja/docs/Web/API/Web_Audio_API) を使用しました。

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/state/playerState.ts

Web Audio API は音声を再生するだけではなく、エフェクトを付けたり正確なタイミングで再生時間を制御できるものです。奥がとても深くて組み合わせ次第で色々できそうなのですが、僕の知識は浅いため今回使ったものだけを紹介します。

```tsx
// Web Audio API のすべての始まり AudioContext
const context = new AudioContext();

const start = async () => {
  // 音声データを ArrayBuffer として取得
  const arrayBuffer = await fetch("/sound.mp3").then((r) => r.arrayBuffer());
  // ArrayBuffer から AudioBuffer へ変換
  const audioBuffer = await context.decodeAudioData(arrayBuffer);
  // AudioBuffer を音源として扱うためのオブジェクト
  const sourceNode = context.createBufferSource();
  sourceNode.buffer = audioBuffer;
  // ループ設定。時間の単位は「秒」
  sourceNode.loop = true;
  sourceNode.loopStart = 1;
  sourceNode.loopEnd = 3;
  // Node の接続。 destination はスピーカーのイメージ。
  sourceNode.connect(context.destination);
  // 再生開始
  sourceNode.start();
};

// 再生中の音声を一時停止
const suspend = () => context.suspend();
// 一時停止中の音声を再開
const resume = () => context.resume();
```

Web Audio API で音声を再生する場合、サイト訪問者のアクションをきっかけにする必要があります。ページを開いただけで音声が再生開始されるようなサイトは作れないということですね。(参考: https://developer.mozilla.org/ja/docs/Web/Media/Autoplay_guide)
上記のサンプルコードでは、ボタンのクリックイベントに `start` を仕込むことで再生ができます。ユーザーのアクションなしに発火する `useEffect` の中などで呼び出しても再生できないことがあります。

ただし、Safari はもっと意味不明で、 `context.createBufferSource()` を実行するタイミングでも再生できるかどうかが変わったりします。僕はどうすれば Safari 様が満足できるか把握するのを諦めました。

今回作成した Web サイトでは React のステート管理と絡ませるため、 Recoil のコードを含む部分で Web Audio API の処理もゴリゴリ書いてしまっています。うまく Web Audio API だけをくくり出せたらコードの見通しが改善されると思うのですが、これは今後の課題にします。

## その他

[Web Share API](https://developer.mozilla.org/ja/docs/Web/API/Navigator/share) を使ってみました。スマホでシェアボタンを押すと色んな Web サービスがシェア先として提示されるやつですね。PC ブラウザのサポートが弱いので、 `window.navigation.share` に関数が入っているかを確認した上で使います。

Web Share API をサポートしていないブラウザの場合は、シェアを選択する UI を自前で用意するのが理想なのかもしれないですが、さぼって Twitter のシェアだけにしています。

https://github.com/y-hiraoka/dp-soundlibrary/blob/621f1cfc933e4b223c61ed1e159c998ef6228d22/src/components/ShareButton.tsx

## まとめ

『ポケットモンスター ダイヤモンド・パール』の BGM を無限ループ再生できるサイトの技術的な側面について雑に紹介してきました。
Next.js に Chakra UI で見せて Recoil で管理して Web Audio API で音鳴らしてるやつを Vercel にデプロイしているよという話でした！

リンク再掲。気にいっていただければぜひシェアしてください！
https://dp-soundlibrary.stin.ink/

それではよいポケモンライフを！
