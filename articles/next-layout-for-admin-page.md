---
title: " 【Next.js】管理者用ページを Route Groups で実現する"
emoji: "🈲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
---

みなさんこんにちは。chot Inc. の Web エンジニアです。

Next.js で Web アプリを開発している時に、ログインしているユーザーが管理者かどうかで出しわけをしたいケースがありました。

次のようなイメージです。

```
/posts ← 一般ユーザーも閲覧可
/posts/:postId ← 一般ユーザーも閲覧可
/mypage ← 一般ユーザーも閲覧可
/dashboard ← 管理者のみ閲覧可
/settings ← 管理者のみ閲覧可
```

これを Route Groups で対応してみたら、コードがスッキリして良いなと思ったので紹介します。

## Route Groups とは

Next.js は ディレクトリの構造と URL のパス構造が一致しますが、時々それらが一致してほしくない場合があります。そんな時に Route Groups を使えば、ディレクトリ構造と URL のパス構造を一致させないようにできます。Route Groups はディレクトリ名を `()` で括ることで実現できます。例えば次の 2 つの page.tsx は同じ `/users/:userId` にマッピングされます(実際に URL が衝突するファイルを複数用意するとビルドエラーになるので注意)。

```
app/users/[userId]/page.tsx
app/users/(foo)/[userId]/page.tsx
```

App Router では layout.tsx を設置するとネストした page.tsx すべてを包むように描画されます。しかし、URL のパス構造と React ツリーのネスト構造は必ずしも一致するとは限りません。Group 直下にも layout.tsx を配置することができるため、URL のパスは並列だが layout.tsx が異なるページを用意したいケースで使えます。

```
app
├── (group1)
│   ├── users
│   │   └── page.tsx
│   └── layout.tsx
├── (group2)
│   ├── posts
│   │   └── page.tsx
│   └── layout.tsx
├── layout.tsx
└── page.tsx
```

上記例では、 `/users` と `/posts` で異なる layout.tsx が適用されます。Route Groups 機能がない場合は `app/layout.tsx` 一つでなんとか分岐するしかありませんが、Route Groups を使うことでコードの分岐ではなくディレクトリレベルでレイアウトを区別することが可能です。

## 管理者用ページの実装方法

管理者かどうかでページを出し分ける要件の例を再掲します。

```
/posts ← 一般ユーザーも閲覧可
/posts/:postId ← 一般ユーザーも閲覧可
/mypage ← 一般ユーザーも閲覧可
/dashboard ← 管理者のみ閲覧可
/settings ← 管理者のみ閲覧可
```

これを実現するには、管理者かどうかチェックする layout.tsx を Route Groups 直下に設置して、管理者ページに対応する page.tsx をその Group に配置するだけです。

```
app
├── (admin-only)
│   ├── dashboard
│   │   └── page.tsx
│   ├── settings
│   │   └── page.tsx
│   └── layout.tsx
├── mypage
│   └── page.tsx
├── posts
│   ├── [postId]
│   │   └── page.tsx
│   └── page.tsx
├── layout.tsx
└── page.tsx
```

`app/(admin-only)/layout.tsx` の実装は次のようなイメージです。

```tsx:app/(admin-only)/layout.tsx
import { notFound } from "next/navigation";
import { FC, ReactNode } from "react";

const Layout: FC<{ children: ReactNode }> = async ({ children }) => {
  const user = await getLoginUser();

  if (!isAdminUser(user)) {
    return notFound();
  }

  return <>{children}</>;
};

export default Layout;
```

`getLoginUser` や `isAdminUser` は適当な関数が用意されているものとします。

ポイントは props で受け取った `children` をそのまま返却している点です。これで見た目上は特に変更せず、ユーザーの権限チェックだけを行ったページを描画します(もちろん UI 上のレイアウトを加えた上で return しても良いです)。

こうすることで、この layout.tsx でネストされている page.tsx は、管理者ユーザーのみ閲覧できることが保証されます。逆に管理者ではないユーザーには 404 ページが返却されます。場合によっては `notFound()` の代わりに `redirect()` でも良いかもしれません。

## ログインチェックについて

ログイン必須ページへのリクエストでユーザーがログインしているかどうかをチェックしたいケースでは、今回の Route Groups の layout.tsx 実装案は向いていません。というのも、Next.js では React Server Components でリクエスト URL が取れません。ログインしていないユーザーをログインページに遷移させ、ユーザーがログインに成功したら遷移前の URL に戻ってきてほしいですが、どこにリクエストを受けたか分からない以上戻すことができません。

これについては Next.js の Middleware を使うと良いです。Middleware ならリクエスト URL を見ることができます。Next.js 用認証ライブラリである next-auth も、リクエストが未ログインの場合はリクエスト元を記憶した上でログインページにリダイレクトさせる Middleware ヘルパーを提供しています。

## まとめ

Next.js の Route Groups を使った管理者ページの実装案を紹介しました。layout.tsx に UI 上のレイアウトだけではなく共通のチェック処理を実装することで実現できます。管理者ページだけでなく色々応用することができそうですね。

それでは良い Next.js ライフを！
