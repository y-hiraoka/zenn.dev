---
title: "オレオレ useFetch function for React"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

# React 用 fetch

`useSWR` のキャッチアップをしたくない人向けの簡易版 fetch for React のご紹介。

## コード

```tsx: useFetch.ts
import { useCallback, useEffect, useReducer, useState } from "react";

type FetchedState<T> =
  | { isFetching: true }
  | { isFetching: false; isError: false; data: T }
  | { isFetching: false; isError: true; error: any };

export function useFetch<T>(fetch: () => Promise<T>, deps: React.DependencyList) {
  const [fetchedState, setFetchedState] = useState<FetchedState<T>>({
    isFetching: true,
  });

  // eslint-disable-next-line react-hooks/exhaustive-deps
  const memoizedFetch = useCallback(fetch, deps);

  const [count, forceReFetch] = useReducer((s: number) => (s + 1) % 100_000_000, 0);

  useEffect(() => {
    const effect = async () => {
      try {
        setFetchedState({ isFetching: true });
        const fetched = await memoizedFetch();
        setFetchedState({ isFetching: false, isError: false, data: fetched });
      } catch (error) {
        setFetchedState({ isFetching: false, isError: true, error });
      }
    };

    effect();
  }, [memoizedFetch, count]);

  return [fetchedState, forceReFetch as () => void] as const;
}
```

## 使い方

```tsx
import { useFetch } from "./useFetch";

type Props = { tweetId: string };
type Tweet = { tweetId: string; content: string };

const App: React.VFC<Props> = ({ tweetId }) => {
  const [fetchedTweet, refetch] = useFetch(async () => {
    const response: Tweet = await fetch(`/api/tweet/${tweetId}`).then((r) => r.json());
    return response;
  }, [tweetId]);

  if (fetchedTweet.isFetching) {
    return <div>Loading...</div>;
  }

  if (fetchedTweet.isError) {
    return <div>error!!!</div>;
  }

  return (
    <div>
      <div>{fetchedTweet.data.content}</div>
      <button onClick={refetch}>refetch</button>
    </div>
  );
};
```

## 解説

コードの解説をしていきます。

### 関数インターフェイスについて

この `useFetch` はカスタムフックとして作成されており、 `useCallback` や `useEffect` 等と同じように `DependencyList` を第 2 引数として取ります。
第 2 引数の `DependencyList` の各要素が変化した場合は自動でデータを再取得するようになっています。そのため、 第 1 引数の `fetch` が依存している値を漏れなく入れてください。

`useFetch` の戻り値は固定長配列になっており、その第 1 要素 `fetchedState<T>` は以下の通り変化します:

- データ取得中は `isFetching: true` のみ含む
- データ取得後はエラー有無に関わらず `isFetching: false` を含む
- エラーが発生した場合は `isError: true` と、 `error` プロパティにエラーオブジェクトを含む
- 正常終了した場合は `data` プロパティに取得されたデータを含む

`DependencyList` に変化はないが明示的に再取得をしたい場合は、戻り値第 2 要素の `refetch` 関数を実行します。

### 内部実装について

`useState` で戻り値のステート `fetchedState` を宣言します。このステートは `useEffect` 内部で `fetch` の進行状況に応じて適切な値がセットされます。

`useCallback` で引数の `fetch` をメモ化します。これは引数の `deps` が変化した場合にのみ `memoizedFetch` を再生成することで、 `deps` が変化した場合だけ `useEffect` が実行されるようになります。

`useReducer` で forceReFetch 機能を作成します。インクリメントされるだけの `count` ステートを宣言し、それを `useEffect` の `DependencyList` に渡します。こうすることで、 `count` のインクリメントを forceReFetch とみなすことができます。そのため、 `useReducer` の `dispatch` を戻り値の第 2 要素として返却します。

`useEffect` で `memoizedFetch` を実行し、その結果を `setFetchedState` に渡します。

## いいところ

- 型安全性

  `FetchedState<T>` の型定義によって、フェッチ中ではないこととエラーが発生していないことを確認してからでないと、データを扱うことができなくなっています。 え、 JavaScript をそのまま書いてる？ちょっと何言ってるかわかりません…。

- 再取得 API がある

  任意のタイミングで再取得ができます。

- Promise を返せば実はなんでもいい

  いいところなのかはわかりませんが、 `useFetch` と言いつつ Promise を返せば何でも受け取れるので `axios` or `fetch` の選択はもちろん、 `fileReader` とかでも使えそう(やったことはないけど)。

## わるいところ

- Mutation API がない

  まるごと再取得しないとデータを最新にできません。作れよって話なんですが。 `useSWR` 使ってください。

- キャッシュ機能がない

  `useSWR` はデータキャッシュと自動 refetch でいい感じにやってくれますが、そういう難しいのはないです。 `useSWR` 使ってください。

- 僕が考えた

  `useSWR` 使ってください。

## まとめ

僕が考案したオレオレ `useFetch` フックの紹介をしました。普通に業務でも使っていますが、今のところこれで困っていません。
fetch 用の外部ライブラリを入れたくない人は utils ディレクトリとかにでも突っ込んでおくといいと思います。

Concurrent Mode 時代が到来するまではこれで行こうと思います(決意)。いや、決して `useSWR` をキャッチアップしたくないとかそんなんじゃなｋ(ry
