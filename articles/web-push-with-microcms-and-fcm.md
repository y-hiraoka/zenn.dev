---
title: "iOS でも Web Push が送れる！microCMS と Firebase Cloud Messaging を使った実装方法"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "microcms", "nextjs"]
published: false
publication_name: chot
---

お初にお目にかかります(？)

chot Inc. で Web エンジニアをしているすてぃんです。

もう 1 ヶ月前ですが iOS Safari でプッシュ通知に対応したバージョンのベータ版が発表されましたね。

https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/

Web のプッシュ通知は、シェアの大きい iOS が対応してくれなかったのであまり実用的ではありませんでしたが、これからは Web でも多くのユーザーに能動的に新着情報を伝えることができるようになります。

本記事では Firebase Cloud Messaging(以降 FCM と表記) を用いた Web Push 通知の実装方法を、架空のメディアサイトを実装しながら紹介します。

FCM 以外には microCMS と Next.js を使用します。microCMS にはコンテンツ更新時に任意の URL に POST リクエストを送信してくれる Webhook の機能があるので、これを活用します。Next.js は特に通知機能に対して大きな意味はありませんが、いい感じのカスタムフックの作り方もご紹介します。

## 最終的な成果物

架空のメディアサイト

https://microcms-notification-media.vercel.app/

リポジトリ

https://github.com/y-hiraoka/microcms-notification-media

サイトには 1 日 2 回、適当な記事が入稿されるようにしてあります。このサイトのプッシュ通知を ON にしていただくと、プッシュ通知を受け取る体験ができるようになっています！なお、架空のメディアサイトからは通知オフができませんので、~~うざくなったら~~オフにしたい場合はブラウザの設定から通知設定の変更をお願いします。

## iOS プッシュ通知対応の注意点

iOS のプッシュ通知をサポートするにあたり、いくつか注意点があります。

1 つ目は、[**ウェブアプリマニフェスト**](https://developer.mozilla.org/ja/docs/Web/Manifest) が必須であることです。これは 2 つ目の注意点に繋がります。

2 つ目は、Web アプリを Add to Home Screen(A2HS、ホーム画面に追加)してもらう必要があることです。通常の Web ページの一つとして Safari で閲覧しているときは、プッシュ API は利用できないようになっています。A2HS までの導線をうまくデザインできるかも重要になってきますね。また、ウェブアプリマニフェストが設置されていることで、A2HS したときにネイティブアプリっぽく振る舞うことも重要です。それにはウェブアプリマニフェストを正しく記述する必要があります。

3 つ目は通知権限の要求は必ずユーザーのインタラクション後に行う必要があることです。Android Chrome などはこの制約がないため、ランディングしてすぐ「通知を許可しますか」と表示してくるマナーの悪いサイトが大量発生しました。iOS がインタラクション後にのみ通知権限を要求できる設計にしたことで、本当にそのサイト・Web アプリに関心があるユーザーが自発的に通知を許可してくれるでしょう。サイトデザインもそれを考慮して「新着通知はこちらをクリック」のようなボタンを設置する必要があります。

## サービス構成

サービス構成を図にすると次のようなイメージになります。(お粗末な絵ですみません)

![microCMS から Firebase Cloud Functions に webhook というテキスト付きで矢印が伸びている。microCMS から Next.js に Contents というテキスト付きで矢印が伸びている。Next.js から Firebase Cloud Firestore に Device Token というテキスト付きで矢印が伸びている。Firebase Cloud Firestore から Firebase Cloud Functions に Device Tokens というテキスト付きで矢印が伸びている。Firebase Cloud Functions から Firebase Cloud Messaging に message request というテキスト付きで矢印が伸びている。Firebase Cloud Messaging から Next.js に notification というテキスト付きで矢印が伸びている。](/images/web-push-with-microcms-and-fcm/service-matrics.png)

microCMS からコンテンツを取得して Next.js でページをビルドします。

Next.js でビルドされたサイトではプッシュ通知の許可をユーザーに要求し、FCM SDK で発行されるデバイストークンを Firestore に格納しておきます。これは通知を送信するのに使用するデバイスの識別子です。

一方 microCMS の Webhook でコンテンツの追加と同時に Functions を実行します。Functions では FCM に対して通知送信を実行します。図では表現できていませんが、Webhook を受けるたびにデバイストークンを都度全件取得して FCM に渡すわけではありません。FCM ではデバイストークンに対してトピックを紐付けることができます。Firestore にデバイストークンが保存されたイベントによって発火する Function を実装しておき、そこでトピック紐付けを行います。FCM で通知を送信する時は、トピックを指定するだけで紐づいているデバイスに通知が送信されるので効率的です。

## 実装

Firebase にすでにプロジェクトがあり、PC に `firebase` コマンド等がインストールしてあることを前提に話を進めます。Firebase Functions を使うので有料プランにアップグレードしてください。お試し実装なら無料枠で済むので大丈夫です。

microCMS にもプロジェクトがある前提で進めます。簡単かつ無料で作成できるので適当にプロジェクトを作成してください。今回のデモも無料枠の範囲で試すことができます。

### 記事表示まで(ほぼおまかせ)

ソースコードを格納するディレクトリを作成します。

```bash
mkdir microcms-notification-media
cd microcms-notification-media
```

git 管理する人は `git init` もここでしておきます。

続いて Firebase プロジェクトとしてセットアップします。

```bash
firebase init
```

CLI に使用する機能を伝えるときに Firestore と Functions を選択してください。Firebase Cloud Messaging は選択肢にないので探さなくても大丈夫ですよ。その他は CLI に沿ってセットアップを完了させてください。(Firestore を事前に Firebase Console でインスタンス生成しておく必要があることに注意。してなければ CLI が準備のための URL を表示してくれるので迷うことはないでしょう。)

同じディレクトリで Next.js のセットアップも行います。

```bash
npx create-next-app website --ts --use-npm
```

`website` の箇所は好きな名前で(ただしこの記事では Next.js のディレクトリを差して `website` と呼ぶ場合があります)、 `--ts` は必須で(？)、 yarn が入ってるのに npm を使いたい~~不思議な~~人は `--use-npm` を指定しましょう。Next.js には app ディレクトリという機能がありますが、僕は pages ディレクトリで実装しているので、以降の説明はそれが前提であることに注意してください。あとは CLI の言われるがままに。

```bash
.
├── firebase.json
├── firestore.indexes.json
├── firestore.rules
├── functions
│   ├── node_modules
│   ├── package-lock.json
│   ├── package.json
│   ├── src
│   ├── tsconfig.dev.json
│   └── tsconfig.json
└── website
    ├── README.md
    ├── next-env.d.ts
    ├── next.config.js
    ├── node_modules
    ├── package-lock.json
    ├── package.json
    ├── public
    ├── src
    └── tsconfig.json
```

ここまででディレクトリ構造(2 階層目まで)はこんな感じになっています。

そうしましたら、 Next.js アプリである `website` のほうを適当に実装して通知機能以外の記事一覧と記事詳細を表示できるようにしてください(本筋ではないので解説はいたしません)。僕は `src/pages/index.tsx` と `src/pages/articles/[contentId].tsx` を Chakra UI で作りました。 microCMS のデータ取得は公式 SDK の microcms-js-sdk や弊社のテックリードが作った microcms-ts-sdk が使えます。

もし僕のコードをコピペしたい場合は次の URL をご覧ください。MIT ライセンスの範囲で好きなように使っていただけます。

https://github.com/y-hiraoka/microcms-notification-media/blob/main/website/src/pages/index.tsx

https://github.com/y-hiraoka/microcms-notification-media/blob/main/website/src/pages/articles/%5BcontentId%5D.tsx

### 通知機能のセットアップ

まずは iOS 対応の注意点で述べたように、ウェブアプリマニフェストが必要です。といっても iOS 対応で必須なのは、ネイティブアプリっぽく見せるための `display` プロパティのみです。 `display` には `"standalone"` か `"fullscreen"` が指定されている必要があります。ほかは適当に入れられる範囲で。ウェブアプリマニフェストについての説明は MDN を御覧ください。

https://developer.mozilla.org/ja/docs/Web/Manifest

僕は次のような JSON を `manifest.json` というファイル名で `website/public` ディレクトリに配置しました。

```json
{
  "$schema": "https://json.schemastore.org/web-manifest-combined.json",
  "name": "Web Push Media",
  "short_name": "Web Push Media",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#fff",
  "description": "Webでも通知が来るメディア"
}
```

ウェブアプリマニフェストの場所をブラウザに伝えるために、 `<head>` 要素に `<link>` 要素を挿しておきます。 `_document.tsx` を編集します。

```tsx
import { Html, Head, Main, NextScript } from "next/document";

export default function Document() {
  return (
    <Html lang="ja">
      <Head>
        <link rel="manifest" href="/manifest.json" />
      </Head>
      {/* 略 */}
    </Html>
  );
}
```

これでウェブアプリマニフェストがブラウザに読み込まれるようになりました。 A2HS するとネイティブアプリっぽく起動するようになります。PWA と呼ぶにはもう 1 つ要素が必要で、Service Worker を置く必要があります。Service Worker は FCM SDK が要求するため必須です。次のような Service Worker を同じく `website/public` ディレクトリに配置しましょう。

```jsx
importScripts("https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js");
importScripts(
  "https://www.gstatic.com/firebasejs/8.10.1/firebase-messaging.js"
);

firebase.initializeApp({
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "...",
});
const messaging = firebase.messaging();

messaging.onBackgroundMessage((payload) => {
  console.log(
    "[firebase-messaging-sw.js] Received background message ",
    payload
  );
});
```

Service Worker を配置する URL のパスは `/firebase-messaging-sw.js` である必要があります。これは FCM SDK がそのパスを指定して Service Worker の登録を試みるからです。存在しないとエラーになります。

Service Worker からも Firebase SDK を読み込みますが、`importScripts` を使用して CDN からダウンロードします。FCM の Service Worker はモジュールではないので v8 を読み込んでいます。今回は .js ファイルを直接書いて置いていますが、バンドラを駆使して Service Worker をビルドするようにすれば、npm install した v9 の SDK も使えるんですかね？やってみたことはないので不明…。

Service Worker でやることは FCM SDK を初期化するだけです。これさえすれば、FCM から送られてきた通知データをバックグラウンドにいてもプッシュ通知してくれます。追加で何かしたいときは `onBackgroundMessage` イベントを処理します。Service Worker 内なので DOM にアクセスもできずやることは多くないかと思いますが。何もすることがなければ `firebase.messaging()` だけで十分です。

これで Web サイトが PWA と呼べるようになり、Web サイトがバックグラウンドにあるときに FCM からのプッシュ通知が表示されるようになりました！

### FCM で公開鍵を発行する

Web Push 通知では、公開鍵認証を用いた送信者の証明を行います。知らないサーバーからプッシュ通知が送られてきたといった事態を防ぐためです。FCM では鍵ペアの生成は Firebase Console からワンクリックでできるようになっています。 Console の URL はこちら:

https://console.firebase.google.com/project/\_/settings/cloudmessaging/?hl=ja

Generate key pair で鍵のペアを生成してください。

![ウェブプッシュ証明書発行画面。未発行なので Generate key Pair のボタンが表示されている](/images/web-push-with-microcms-and-fcm/gen-vapid-key-console.png)

生成後に表示されているランダムな文字列が VAPID KEY と呼ばれるもので要するに公開鍵のほう。Web に埋め込むので公開情報です。漏らしても大丈夫です。対になる秘密鍵のほうは開発者にすら表示されないので安心です。

ここで発行した VAPID KEY を後から使うのでメモっておきましょう。公開情報なのでいつでもコンソールで確認できますけどね。

### 通知権限をブラウザに要求する

Next.js 側で通知権限リクエストを表示して、ユーザーに許可されたらデバイストークンを Firestore に保存する処理を用意します。関数名は `requestNotificationPermission` とします。

```tsx
import { initializeApp } from "firebase/app";
import { getFirestore, collection, addDoc } from "firebase/firestore";
import { getMessaging, getToken } from "firebase/messaging";

const app = initializeApp({
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "...",
});

export async function requestNotificationPermission() {
  const firestore = getFirestore(app);
  const messaging = getMessaging(app);

  try {
    const token = await getToken(messaging, {
      vapidKey: process.env.NEXT_PUBLIC_FIREBASE_WEBPUSH_KEY,
    });

    if (token) {
      console.log(`Notification token: ${token}`);
      await addDoc(collection(firestore, "notification"), { token: token });
    } else {
      console.log(
        "No registration token available. Request permission to generate one."
      );
    }
  } catch (error) {
    console.error("An error occurred while retrieving token. ", error);
  }
}
```

FCM SDK の `getToken` を実行することで、ユーザーに通知権限を要求します。「このサイトの通知を許可しますか？」のようなダイアログが表示されるアレです。我々が `Notification.requestPermission()` を呼ぶ必要はなく、内部で勝手にやってくれます。許可されればデバイストークンが発行されるので、それを Firestore に保存します。

`getToken` の第 2 引数で渡されている環境変数が先程 Firebase Console で発行した VAPID KEY です。実行する前に環境変数にセットされるようにしておいてください。(公開情報なのでハードコーディングで良かったかもしれない)

Firestore のコレクション名は `notification` で、 `{ token: string }` という型でデータをドキュメントに保存しておきます。この型で保存されていることは頭の片隅に置いておいてください。

これを試しに実行する前に、セキュリティルールも書きましょう(テストモードのフルアクセスで Firestore を起動している人は書く必要はないですがお気をつけて 🙏)。`notification` にだけドキュメントの追加を許可します。

```firestore-security-rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /notification/{notificationId} {
      allow create: if true;
    }
  }
}
```

これで `notification` に対してドキュメントの新規作成だけできるデータベースになります。デプロイしておきましょう。

```bash
firebase deploy --only firestore:rules
```

さて `website` のコードに戻り、ボタンの `onClick` イベントに仕込みます。モジュールトップとか `useEffect` のような場所で実行してしまうと、ユーザーインタラクションの前に実行することになるので Safari に無視されてしまいます。Safari でなくとも、ユーザーに不快感を与えないためにも必ず興味をもって能動的に通知ボタンをクリックしてくれるユーザーにだけ権限リクエストを表示しましょう。

僕の場合は Next.js で作っているので、いい感じのカスタムフックにラップします。 `useNotification` という関数名で、通知が許可されている状態、通知の権限を要求する処理、通知の権限を要求している途中かどうかを取得できると便利ですね。型で表現するなら次のような関数型です:

```tsx
declare function useNotification(): {
  permission: NotificationPermission | "not-supported";
  requestPermission: () => void;
  isRequesting: boolean;
};
```

`NotificationPermission` は TypeScript 組み込みの型定義で、次のようなリテラル型です:

```tsx
type NotificationPermission = "default" | "denied" | "granted";
```

`"default"` は通知権限リクエスト前、 `"denied"` は通知が拒否されている、 `"granted"` は通知が許可されている状態です。これがわかれば相応の UI が表示できますね。また、プッシュ通知がサポートされていないブラウザで閲覧しているときもその旨を伝えられるように、独自に `"not-supported"` をユニオンに追加しています。

`requestPermission` は先程の Firebase SDK をラップした `requestNotificationPermission` の実行に加えて、React ステートをいい感じに変更します。

`isRequesting` は、 `requestNotificationPermission` が非同期処理なのでローディング UI を見せるために用意します。

実装はこんな感じ。

```tsx
import { useCallback, useEffect, useState } from "react";
import { requestNotificationPermission } from "@/firebase";

export function useNotification() {
  const [permission, setPermission] = useState<
    NotificationPermission | "not-supported"
  >("default");

  const revalidate = useCallback(() => {
    if ("Notification" in window) {
      setPermission(Notification.permission);
    } else {
      setPermission("not-supported");
    }
  }, []);

  // SSR 時に Notification を参照するのを回避するための useEffect
  useEffect(() => {
    revalidate();
  }, [revalidate]);

  const [isRequesting, setIsRequesting] = useState(false);

  const requestPermission = useCallback(() => {
    setIsRequesting(true);
    requestNotificationPermission()
      .then(() => revalidate())
      .finally(() => setIsRequesting(false));
  }, [revalidate]);

  return { permission, requestPermission, isRequesting };
}
```

戻り値のうち `permission` は React ステートとして宣言します。

その次のフックのスコープに閉じている `revalidate` 関数では、 `Notification` がサポートされているブラウザかどうかを確認して、サポートされていれば現在のステートに権限状態を `setPermission` し、サポートされていなければ `"not-supported"` をセットする処理を行います。

続いて `useEffect` で `revalidate` を発火します。ブラウザで必ず一度発火するようにしておけば、通知権限の初期状態の表示ができます(SSR しているときはサーバーサイドでは初期値を `"default"` 固定にするしかなく、UI 表示がチラつくのはご愛嬌)。

最後に権限要求を行う `requestPermission` を作ります。処理中フラグである `isRequesting` ステートも宣言し、実行の最初に `setIsRequesting(true)` しておきます。 内部で Firebase SDK の処理を行う `requestNotificationPermission` を発火して、完了したら `revalidate` を再度実行して権限状態を更新します。 `finally` で必ず `isRequesting` のフラグを戻すこともしておきましょう。

`permission`, `requestPermission`, `isRequesting` を return してカスタムフックの処理を終了とします。

あとはこれを呼び出すだけです。ステートに対応する UI はよしなに。

```tsx
export const AppSideNav: FC = () => {
  const { permission, isRequesting, requestPermission } = useNotification();

  return (
    <Box>
      {permission === "not-supported" ? (
        <Text>プッシュ通知がサポートされていません</Text>
      ) : permission === "denied" ? (
        <Text>プッシュ通知を拒否しました</Text>
      ) : permission === "granted" ? (
        <Text>新着記事をプッシュ通知します！</Text>
      ) : (
        <Button onClick={requestPermission} isLoading={isRequesting}>
          通知を受け取る
        </Button>
      )}
    </Box>
  );
};
```

「通知を許可しますか？」のダイアログが表示されているうちは Promise がずっと pending しているので、 `isRequesting` を `isLoading` なり `disabled` に渡しておくとよいです。(上のコードは Chakra UI です。無関係な props は省略)

これでボタンがクリックされたらユーザーに権限を要求するようになりました！権限の状態によって適切な文言も表示できますね。

### デバイストークンにトピックを紐付ける

FCM では、トピックとデバイストークンを紐付けてトピックに対してプッシュ通知することで一斉送信ができる機能があります。これを利用して、全デバイストークンに同じトピックを紐付けて、そのトピックを指定して通知送信すれば全員にプッシュ通知が実現できます。もちろんデバイストークンを指定して直接送信することもできますが、全員に送信するケースだと毎回 Firestore から全員分のデバイストークンを read することになるのでコストがかかります。また、デバイストークンを複数指定して一括送信する API は一度に指定できるデバイストークンの上限があるので、通知送信処理の記述が面倒になります。

Firebase Functions のトリガー機能で Firestore にデバイストークンが登録されると同時に共通トピックを紐付ける処理を実装しましょう。

```tsx
import * as functions from "firebase-functions";
import * as admin from "firebase-admin";
import { getMessaging } from "firebase-admin/messaging";

admin.initializeApp();

export const subscribeToNewArticle = functions.firestore
  .document("notification/{notificationId}")
  .onCreate(async (snapshot) => {
    await getMessaging().subscribeToTopic(snapshot.data().token, "new-article");
  });
```

`getMessaging().subscribeToTopic()` の第 1 引数がデバイストークン、第 2 引数がトピックです。 `"new-article"` としていますがどんな文字列でも大丈夫です。これだけでデバイストークンとトピックが紐付けられます。 `onCreate` イベントなのですべてのデバイストークンが強制的に `"new-article"` トピックを購読するのもわかりますね。

### プッシュ通知を送信する

microCMS で記事が作成されると同時にプッシュ通知を送信したいですね。microCMS にはコンテンツの編集をきっかけに任意の URL に HTTP POST を送信してくれる Webhook 機能があります。これを Firebase Functions で受けて、FCM の通知送信を行うという流れになります。関係ないですが microCMS の Webhook を Next.js の API Routes で受けることで on-demand ISR も実装できます。便利。

microCMS の Webhook で送られてくるリクエストボディは次のようなデータ構造の JSON です:

```tsx
type WebhookBody = {
  service: string;
  api: string;
  id: string | null;
  type: "new" | "edit" | "delete";
  contents: {
    new: {
      id: string;
      publishValue: {
        title: string;
        thumbnail?: { url: string };
      };
    } | null;
  } | null;
};
```

実際はもういくつかプロパティが生えていますが今回使う分だけ記載しています。

`publishValue` の部分は microCMS で各自設定するスキーマがそのまま格納されています。上記は僕が設定した記事データが入っていることを想定しています。プッシュ通知に使用するのが記事データのうち `title` と `thumbnail` だけなので、その他もデータに含まれてはいますが型定義からは省略しています。

ではいよいよ記事更新の Webhook を受け取って FCM によってプッシュ通知を送信する処理です。

```tsx
export const notifyNewArticle = functions.https.onRequest(
  async (request, response) => {
    if (request.method !== "POST") {
      response.status(405).send("method not allowed.");
      return;
    }

    const requestKey = request.headers["x-notification-request-key"];

    if (!requestKey || requestKey !== process.env.NOTIFICATION_REQUEST_KEY) {
      response.status(401).send("invalid request key.");
    }

    const webhook = request.body as WebhookBody;

    if (webhook.type !== "new") {
      response.send({ notified: false });
      return;
    }

    await getMessaging().send({
      topic: "new-article",
      notification: {
        title: "新着記事のお知らせ",
        body: webhook.contents?.new?.publishValue.title,
        imageUrl: webhook.contents?.new?.publishValue.thumbnail?.url,
      },
      webpush: {
        fcmOptions: {
          link: `https://microcms-notification-media.vercel.app/articles/${webhook.id}`,
        },
      },
    });

    response.send({ notified: true });
  }
);
```

先頭 2 つの if 文はただのリクエスト検証です。2 つ目の if 文では環境変数を使用しているのでセットしておいてください。なお、[microCMS のドキュメントにはもっとセキュアなリクエスト検証の方法が紹介してあります](https://document.microcms.io/manual/webhook-setting#hb2d39bd6cc)のでこれは参考にしないほうが良いです。

3 つ目の if 文では、 Webhook の種類が記事の新規作成かどうかを確認しています。ここでは削除や更新のときはプッシュ通知したくないので、 `webhook.type !== "new"` のときはさっさとレスポンスを返してしまいます。

そして次が FCM による通知送信部分です。 `getMessaging().send({ topic, notification, webpush })` となっています。

`topic` は通知を送信したいトピックで、ここで指定した値に紐付けられているデバイスに通知が送信されることを意味します。

`notification` は通知の内容を指定します。 記事に設定したサムネイルを `imageUrl` プロパティに指定することで、プッシュ通知にも画像が表示されるようにできます(デバイス側が対応していればですが…)。

`webpush.fcmOptions.link` がメディア系サイトでは重宝されると思います。届いたプッシュ通知をクリックしたときに開く URL を指定することができます。新着記事のお知らせなので、当然タップしたときにその記事の本文が開くことを期待するでしょう。この値をセットしておけばすぐにユーザーが見たいものを表示させることが可能です。上記コードサンプルでは実際に Next.js がデプロイされている Vercel の URL を指定しています。

プッシュ通知の送信が完了したら適当にレスポンスを返して終わりです(Webhook にレスポンスしても意味ないのでなんでも大丈夫です。204 でも)。

この HTTP 関数と一つ前のセクションのトリガー関数をデプロイしておきましょう。

```sh
firebase deploy --only functions
```

デプロイが完了したら HTTP 関数のほうの URL が CLI に表示されるのでそれをメモしておきます。

### microCMS で Webhook を設定する

自分の microCMS の開き、

コンテンツ（API） > {プッシュ通知したい記事 API} > API 設定 > Webhook

のページを開きます。「+ 追加」ボタンをクリックしていい感じに入力しましょう。

![Untitled](/images/web-push-with-microcms-and-fcm/microcms-webhook-setting.png)

URL には Firebase Functions のデプロイ時に表示された URL を入力します。

microCMS がちゃんとおすすめしているセキュアなリクエスト検証をしている人は「シークレット」を入力してください。僕は手抜きなのでヘッダーにリクエストキーを入れています。Key の部分はソースコードでヘッダーの値を取り出すときに指定した Key と一致していることを確認してください。

通知タイミングの設定では「コンテンツの公開時・更新時」のチェックボックスだけ ON になっていれば十分です。

これで microCMS で記事を作成・更新すると Webhook が Firebase Functions に飛ぶようになりました。

以上で実装完了です！あとは Next.js をよしなにデプロイしたら完成です。

## コスト的な話

FCM に乗っかって通知機能を実装する場合、コストはかなり小さく(規模次第ではずっとゼロ)抑えられます。というのも、FCM は完全無料で提供されていて、どれだけ使っても課金対象になることはありません。今回の構成のように Firebase Functions や Firestore と組み合わせて使用する場合は、それらが課金対象になります。が、無料枠がそれなりにある上に従量課金なのでコストはかなり小さく見積もれるでしょう。

Functions は Next.js の API Routes に寄せることもでき、Firestore は工夫すればなしでも作れます。その場合は Firebase admin 環境構築のために秘密鍵を発行して扱うことにはなりますが。

## より現実的なプッシュ通知活用

興味のある記事カテゴリーに絞ってプッシュ通知を受け取りたいというユーザーもいるでしょう。それに応えることもできます。

記事に記事カテゴリーを設定できるようにして、記事カテゴリーの ID をトピックに設定することでカテゴリー単位で記事を購読してもらうことが可能です(というかトピック送信の本来の使い方)。

FCM は柔軟に送信先指定ができるようになっているので、色々試したいですね。

## まとめ

Firebase Cloud Messaging, microCMS, Next.js で Web プッシュ通知機能の実装方法を紹介しました。

- microCMS の Webhook で Firebase Functions を呼ぶ
- FCM のトピック機能で全員通知ができる

また iOS Safari もサポートするために必要な注意点の紹介もしました。

- ウェブアプリマニフェストに `display` をセットして配置する
- A2HS をしてもらう(そのための導線が必要)
- ユーザーのインタラクションで通知権限をリクエストする

Safari が要件に掲げているように、ユーザーの操作もなしにリクエストダイアログを表示するのはもうやめましょう。Safari の決定によって Web Push 通知の治安がよくなることを祈るばかり 🙏

それでは良い Web ライフを！
