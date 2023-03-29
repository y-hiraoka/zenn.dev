---
title: "【Next.js】getStaticPropsでのnotFoundとrevalidateの組み合わせはどんな挙動をするのか。検証します"
emoji: "🫡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
publication_name: chot
---

## 動機

`getStaticProps` から `notFound` と `revaidate` を組み合わせて返す時、「あれ、これどんな挙動するんだっけ」となった。

例えば、あるタイミングで `notFound` だけを return したが、その後コンテンツを追加して再度ページアクセスした場合は 404 にならずにコンテンツは表示されるのか。

また、`notFound` と `revalidate` をセットで return したが、その後コンテンツを追加して {revalidate}秒以内に再度ページアクセスした場合は 404 になるのか。

## 検証準備

ルーティングとして次の 2 つを用意する。どちらも `getStaticProps` で同じコンテンツを取得する。

- `pages/with-revalidate/[contentId].tsx`
  revalidate とセットで notFound を返す

  ```tsx
  export const getStaticProps: GetStaticProps<PageProps, PageParams> = async ({
    params,
  }) => {
    if (!params) return { notFound: true };

    const content = await getSomeContent(params.contentId).catch(() => null);

    if (!content) {
      return {
        notFound: true,
        revalidate: 60, // revalidate とセットで notFound を返す
      };
    }

    return { props: { content } };
  };
  ```

- `pages/without-revalidate/[contentId].tsx`
  notFound だけで返す

  ```tsx
  export const getStaticProps: GetStaticProps<PageProps, PageParams> = async ({
    params,
  }) => {
    if (!params) return { notFound: true };

    const content = await getSomeContent(params.contentId).catch(() => null);

    if (!content) {
      return {
        notFound: true, // notFound だけで返す
      };
    }

    return { props: { content } };
  };
  ```

アクセス時にページ生成されるようにどちらのページも次の関数を export しておく。

```tsx
export const getStaticPaths = async () => {
  return {
    paths: [],
    fallback: "blocking",
  };
};
```

`getStaticProps` の検証はプロダクションビルドされたサーバーで行う必要があるため、 `npm run build && npm run start` で起動した Next.js サーバーで動作を確認する。

## 検証

- `pages/with-revalidate/[contentId].tsx`

  1. 存在しない `contentId: "test1"` でアクセスする
  1. 404 が返ってくる
  1. `contentId: "test1"` でコンテンツを作成する
  1. {revalidate}秒以内に再びアクセスする
  1. **結果その 1:** **{revalidate}秒以内はずっと 404 が返る**
  1. {revalidate}秒以降に再びアクセスする
  1. **結果その 2:** **{revalidate}秒以降はコンテンツが返る**

- `pages/without-revalidate/[contentId].tsx`

  1. 存在しない `contentId: "test2"` でアクセスする
  1. 404 が返ってくる
  1. `contentId: "test2"` でコンテンツを作成する
  1. 再びアクセスする
  1. **結果その 3:** **そのページは永久に 404 を返す**

## おまけ検証

`without-revalidate` のパスから `notFound` が返されたことをどこで覚えているのか気になった。

`.next/server` ディレクトリにはそれらしきファイルはなかった。

一度サーバーを落として `npm run start` で再起動した後、 404 だったページに再びアクセスしてみた。

アクセスできた。

`revalidate` を設定しなかった `notFound` はインメモリで覚えているっぽい、多分。

以上！
