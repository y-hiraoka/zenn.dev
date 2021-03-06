---
title: "初学者の頃に作ったクソアプリをアップデートした"
emoji: "🆙"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["react", "ポエム", "駆け出しエンジニア"]
published: true
---

# 当時のソースコードを書き直すの楽しいぞ

1 年半ほど前、無職で JavaScript の ジャ の字も知らない頃に React の勉強がてら色々アプリを作っていました。
それなりの知識を習得した今、当時作ったアプリを書き直す作業が楽しかったのでポエムを残します。

## 初学者あるある(？)

### そのへんの技術ブログを信用しすぎ問題

React の勉強を進めていて、コンポーネントの外で useState を呼んでエラーでコケていたのでしょう。
グローバルステートってどうやって管理するんだろうと考えた当時の僕は、「React グローバル ステート」なんかでググっていました。
「Redux は難しい」とか 「context でグローバルステート管理」などの記事を読み、 unstated というステート管理ライブラリにたどり着いていました。

unstated は React context をラップしたライブラリなのですが、購読側は `context.Consumer` ベースの API なので現代の `useContext` ベースの API よりもごちゃごちゃしています。それでも当時の僕は `useContext` の存在も知らないまま unstated を気に入ってしまい、3 つの自作アプリの中で使用していました。

そのおかげで(？)いまだに Redux でアプリを作ったことなく、ステート管理ライブラリを自作する始末です。

### react-router を知らないままアプリを作る

react-router や history API を使用せずにアプリを作っていました。

```tsx
const [page, setPage] = useState<"foo" | "bar">("foo");

return (
  <div>
    <button onClick={() => setPage("foo")}>foo へ</button>
    <button onClick={() => setPage("bar")}>bar へ</button>
    {page === "foo" && <Foo />}
    {page === "bar" && <Bar />}
  </div>
);
```

みたいなコードを愚直に書いていました。そして「リロードしたら全部の状態が戻って画面を維持できない……。SPA ってそういうもんなのかぁ〜」なんて思っていました。 UX のカケラもない。

### create-react-app しか使えないんだ

create-react-app めちゃくちゃ便利なんです。今後もシングルページで済むアプリならば使うつもりです。

でも React がサーバーサイドでもレンダリングできるという点を学べませんでした。 Next.js の存在は再就職してから知ったと思います(Gatsby は辛うじて名前だけ知ってた気がする)。

Firebase Hosting に create-react-app で作成したアプリをデプロイして、Firebase Functions に meta タグだけの HTML を返す HTTP 関数を置いて動的 OGP 生成を実現してました。 Next.js 使えばもっと簡単にできるのに。

## アップデートした内容

### ステート管理

unstated で書かれていたグローバルステート管理を Recoil で書き直しました。
そもそも Recoil を実践してみたくて使い所を考えた結果、過去のアプリの改修を始めたというのもあります。

Recoil の書き心地が本当にいいですね。 React の標準フックとほとんどかわらない API で、使い方を悩むことがほとんどありません。

また、インフラ費用を払いたくなくて Firestore のアクセス数をなるべく減らすため、 unstated にデータをキャッシュする処理を自力で書いていました。愚かですね。
Recoil ならばそのへんを意識しなくても全部勝手にやってくれます。コードが非常にシンプルになってリファクタリングさいこ〜という気持ちになれました。

### URL ルーティング

react-router を導入してアプリの状態と URL を対応付けました。ブラウザリロードしても同じ画面を維持できるようになったり、「戻る・進む」ボタンで画面の状態を戻せるようになりました。

React を使えるようになることに必死で、アプリの操作感はまったく意識していませんでした。他の Web サイトと同様のインタラクションを提供することは大切ですね。

### 動的ページが必要なアプリを Next.js に載せ替える

そのアプリというのがこちらの記事で長々と説明しているものです。
https://qiita.com/stin_dev/items/41ac4acb6ee7e1bc2d50

Firebase Hosting に React SPA を載せて、 Firebase Functions で動的 OGP 生成してますということが長々と書かれています。

ソースコードを整理した上で Next.js に載せ替えました。OGP 用の meta タグを返すのも Next.js で実装することで Firebase Functions を停止できました。アプリ自体は vercel にデプロイしています。
(ドメインを接続せずに Firebase の `.web.app` でそのまま公開していたので、ドメインは変わってしまっています)

Firebase Functions を停止することで、無料枠で運用できるようになりました！これで請求に怯える必要がなくなりますね(？)

ちなみに、アップデート前は Firebase Storage の容量を気にしてユーザー 1 人につき 1 枚しか OGP 画像を保持していませんでしたが、キャッシュによって画像を更新しても OGP 画像が変わらない問題が発生していました。(そしてそれを放置していました。初学者っぽいですね)
今回のアプデによって仕様も修正し、1 ユーザーにつき何枚でも画像を生成できるようになりました 🎉

https://thinking-generator.stin.ink

![demo](https://storage.googleapis.com/zenn-user-upload/jtdtwd7bwb29j2pewx9ctdsozkej)

## まとめ

1 年半ほど前、まだ自分が初学者だった頃に作成したアプリをアップデートしたというお話をしました。

「当時の自分はこんなにひどいコードを書いていたけどそれなりに書けるようになったんだなぁ」と実感できるいい機会になりました。
初学者の方がもし読んでくださっていれば、コードを書く勉強だけではなく実際にアプリを作成して公開することをおすすめします。数年後に読み返したりリファクタリングするのが楽しいです。

「偉そうに言えるほど技術力上がってなくない？」などのツッコミは非常に効くのでやめてください。
