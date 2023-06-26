---
title: "YouTube のチャンネルページ URL から ChannelID を取得する TypeScript コード"
emoji: "📺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["youtube", "typescript"]
published: true
publication_name: chot
---

ChannelID は YouTube の中でチャンネルを一意に識別する ID です(そのまま)。

YouTube のチャンネルページの URL を与えられたとき、そのチャンネルの ChannelID が欲しいことがありました。というのも、最近の YouTube Web アプリではチャンネルページに ID が載っていません。代わりに一意なユーザーネームを使う方式に変わっています。
しかし YouTube Data API などを使うときはやはり ChannelID があったほうが都合が良かったりします。

## 結論

チャンネルページの HTML に記載されている canonical URL に ChannelID を含む旧式 URL が指定されているので、それをスクレイピングする。

## コード

HTML の解析には cheerio を使っています。正規表現一発で取り出せるかもしれませんが考えるのを放棄しました。
エラーのスローではなく Result 型でエラー提示をしています。この辺のインターフェイスは好みで調整してください。

```typescript
import * as cheerio from "cheerio";

type Result =
  | {
      type: "success";
      channelId: string;
    }
  | {
      type: "error";
      reason:
        | "COULD_NOT_FETCH"
        | "NOT_YOUTUBE_URL"
        | "COULD_NOT_PARSE"
        | "NOT_CHANNEL_URL";
    };

export async function getChannelIdFromChannelUrl(url: string): Promise<Result> {
  try {
    const urlObject = new URL(url);
    if (urlObject.hostname !== "www.youtube.com") {
      return { type: "error", reason: "NOT_YOUTUBE_URL" };
    }
  } catch (error) {
    // URL パースエラー
    return { type: "error", reason: "NOT_YOUTUBE_URL" };
  }

  const response = await fetch(url);
  if (!response.ok) {
    return { type: "error", reason: "COULD_NOT_FETCH" };
  }

  const html = await response.text();
  const $ = cheerio.load(html);

  const canonicalUrl = $('link[rel="canonical"]').attr("href");
  if (!canonicalUrl) {
    return { type: "error", reason: "COULD_NOT_PARSE" };
  }

  const channelId = canonicalUrl.match(/channel\/(.*)/)?.[1];
  if (!channelId) {
    return { type: "error", reason: "NOT_CHANNEL_URL" };
  }

  return { type: "success", channelId };
}
```

:::details Vitest によるテストコード

次のテストが全てパスしていることを確認済みです。

```typescript
import { describe, expect, it } from "vitest";
import { getChannelIdFromChannelUrl } from "./getChannelIdFromChannelUrl";

describe("getChannelIdFromChannelUrl", () => {
  it("username 形式 URL から ChannelID を取得する", async () => {
    const result = await getChannelIdFromChannelUrl(
      "https://www.youtube.com/@helloprojectstation"
    );

    expect(result).toEqual({
      type: "success",
      channelId: "UCnoYhOtV0IXZ6lv2R-ZnB_Q",
    });
  });

  it("旧式 URL から ChannelID を取得する", async () => {
    const result = await getChannelIdFromChannelUrl(
      "https://www.youtube.com/channel/UCnoYhOtV0IXZ6lv2R-ZnB_Q"
    );

    expect(result).toEqual({
      type: "success",
      channelId: "UCnoYhOtV0IXZ6lv2R-ZnB_Q",
    });
  });

  it("存在しないチャンネルの URL を渡すとエラーになる", async () => {
    const result = await getChannelIdFromChannelUrl(
      "https://www.youtube.com/@helloprojectstationxxx"
    );

    expect(result).toEqual({
      type: "error",
      reason: "COULD_NOT_FETCH",
    });
  });

  it("チャンネルページではない YouTube URL を渡すとパースに失敗する", async () => {
    const result = await getChannelIdFromChannelUrl(
      "https://www.youtube.com/watch?v=1woGBSxV-L0"
    );

    expect(result).toEqual({
      type: "error",
      reason: "NOT_CHANNEL_URL",
    });
  });

  it("YouTube 以外の URL を渡すとエラーになる", async () => {
    const result = await getChannelIdFromChannelUrl("https://example.com");

    expect(result).toEqual({
      type: "error",
      reason: "NOT_YOUTUBE_URL",
    });
  });

  it("不正な URL を渡すとエラーになる", async () => {
    const result = await getChannelIdFromChannelUrl("www.youtube.com");

    expect(result).toEqual({
      type: "error",
      reason: "NOT_YOUTUBE_URL",
    });
  });
});
```

:::

## FAQ

### YouTube Data API を使えばいいんじゃないの？

確かに YouTube Data API のチャンネル取得 API にはユーザーネームによるクエリが用意されています。次のリファレンスの `forUsername` がそれです。

https://developers.google.com/youtube/v3/docs/channels/list?hl=ja

しかし、原因は不明なのですが実在するユーザーネームを `forUsername` クエリに指定してリクエストを送信したのに、そのチャンネルがレスポンスに含まれていないことがありました。なので今回の方法を考えた次第です。

## まとめ

YouTube のチャンネルページの URL から ChannelID を取得する方法を紹介しました。これでチャンネルページからコピペされた URL を受け取っても YouTube Data API に投げやすくなりますね。

それでは良い開発ライフを！
