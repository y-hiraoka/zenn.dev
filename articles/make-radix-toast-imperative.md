---
title: "Radix Toast をもっと使いやすくしたい！命令型インターフェイスを目指して実装する"
emoji: "🍞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "radix"]
published: false
publication_name: chot
---

Toast は、ユーザーのアクションの結果、成功したり失敗したことをフィードバックするために一時的にぴょこっと表示される UI です。たくさんのヘッドレスコンポーネントを提供する Radix には、Toast の機能とアクセシビリティを提供するパッケージがあります。

```bash
npm install @radix-ui/react-toast
```

この記事では Radix Toast をより使いやすくするための実装案を紹介します。

## モチベーション

React は宣言的 UI ライブラリです。ステートが先にあり、そのステートから計算された結果として完成形のビューを宣言します。Radix も React コンポーネントを提供する以上、このスタイルで扱うことになります。次のコードブロックは Radix Toast のドキュメントのトップで提供されているサンプルコードの抜粋です。

```tsx
import * as React from "react";
import * as Toast from "@radix-ui/react-toast";

const ToastDemo = () => {
  const [open, setOpen] = React.useState(false);
  const timerRef = React.useRef(0);

  React.useEffect(() => {
    return () => clearTimeout(timerRef.current);
  }, []);

  return (
    <Toast.Provider swipeDirection="right">
      <button
        className="略"
        onClick={() => {
          setOpen(false);
          window.clearTimeout(timerRef.current);
          timerRef.current = window.setTimeout(() => {
            setOpen(true);
          }, 100);
        }}
      >
        Add to calendar
      </button>

      <Toast.Root className="略" open={open} onOpenChange={setOpen}>
        {/* 略 */}
      </Toast.Root>
      <Toast.Viewport className="略" />
    </Toast.Provider>
  );
};
```

`open` というトーストの開閉状態を指すステートがあり、それを `Toast.Root` に渡すことで表示状態を制御します。
ボタンをクリックしたら `open` ステートを一旦 `false` にしています。そして `setTimeout` で 100ms 後に `open` ステートを `true` にしています。こうすることでアニメーションを発火することを保証しているのでしょう。タイマー制御のために `useRef` も使っていて、コードを追いにくくなっています。

実際にはトーストは処理の結果をフィードバックするために使うので、どちらかといえば命令的な使い方をしたいはず。つまり次のように書けると嬉しいのではないでしょうか。

```tsx
const handleClick = async () => {
  try {
    await createTweet(text);
    openToast({
      type: "success",
      message: "ツイートしました",
      duration: 3000,
    });
  } catch (e) {
    openToast({
      type: "error",
      message: "ツイートに失敗しました。APIされてます。",
    });
  }
};
```

処理が成功したら `type: "success"` で `openToast` 関数を呼び出し、失敗したら `type: "error"` で呼び出しています。このような命令的に使えると、トーストを使う側のコードがシンプルになります。ブラウザ JavaScript 標準の `alert` 関数と同じような使い勝手になりますね。

## 実装

それでは、命令的にトーストを表示できるようなインターフェイスになるように実装していきましょう。

### 最終的な形

どんなコードを目指すかを最初に示しておきます。

```tsx :index.tsx
import ReactDOM from "react-dom/client";
import App from "./App.tsx";
import { ToastProvider } from "./components/Toast.tsx";

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <ToastProvider>
    <App />
  </ToastProvider>
);
```

```tsx :SomeComponent.tsx
import { useToast } from "../components/Toast";

export const SomeComponent = () => {
  const openToast = useToast();

  const handleClick = async () => {
    await createTweet("Hello World!");
    openToast({
      type: "success",
      title: "ツイートしました",
      duration: 3000,
    });
  };

  return <button onClick={handleClick}>ツイートする</button>;
};
```

よくあるライブラリのように `ToastProvider` コンポーネントをツリーのトップに配置しておきます。この `ToastProvider` が内部でステートをよしなに管理します。
そして `ToastProvider` の子孫コンポーネントが使えるようになる `useToast` カスタムフックを提供します。その戻り値はトーストを表示するための関数になっているので、各コンポーネントがそれを好きなタイミングで呼び出すだけです。

この記事では、 `openToast` を複数回実行すればその回数だけトーストも積み上げられていくように実装します。

### ToastProvider で管理するステートの型

`ToastProvider` では次のような型のステートを配列で管理します。

```tsx :components/Toast.tsx
type ToastItem = {
  id: string;
  type: "success" | "error";
  title: ReactNode;
  description?: ReactNode;
  duration?: number;
  isOpen: boolean;
};
```

`id` はトーストを一意に識別する値ですが、これは `useToast` には公開しません。 `ToastProvider` の中で削除対象を特定したり `key` に渡すために内部でのみ使います。

`isOpen` も `useToast` を使う側からは操作することはありません。 `ToastProvider` の中で時間を測って勝手に `false` になるようにします。

`duration` はトーストが表示されてから消え始めるまでのミリ秒数です。

`title` と `description` は、 Radix Toast のコンポーネントが `Toast.Title`, `Toast.Description` のように分かれているため、それに合わせています。

`type` は独自の値なので、 `"warn"` とか `"info"` のような値を自由に増やすこともできます(対応する UI はその数だけ用意する必要があります)。

上記のオブジェクト 1 つで 1 つのトーストを描画することになります。

### Context で トーストを開くための関数を配信する

子孫コンポーネントが必要なのは、トーストを表示するための関数だけです。それを配信するための React Context を用意しましょう。

`ToastItem` 型のうち `id` と `isOpen` は `ToastProvider` の内部で制御される値です。なので、トーストを表示する関数の引数は `ToastItem` 型からそれらを除いたものになります。

```tsx :components/Toast.tsx
type OpenToastParams = Omit<ToastItem, "id" | "isOpen">;

const OpenToastContext = createContext<(params: OpenToastParams) => void>(
  () => null
);

export function useToast() {
  return useContext(OpenToastContext);
}
```

`OpenToastContext` を export するのではなく、 `useToast` という名前を付けてから export しましょう。react-refresh の eslint が怒る？さぁ…興味ないから…。

実際に `Context.Provider` で値を配信する部分は後述。

### Toast コンポーネントを実装する

`ToastItem` 型がトースト 1 つ分のステートオブジェクトでした。それを受け取って実際にビューを表示するコンポーネントを実装しましょう。尚、本記事ではロジックに焦点を当てるため、スタイリングはすべて省略させていただきます。コピーして使う場合はお好みの CSS と `className` を追記してください。スタイリングのための `div` なども自由に追加できます。また、icon 類もお好きなものに読み替えてください。

```tsx :components/Toast.tsx
import * as RadixToast from "@radix-ui/react-toast";

const Toast: FC<{
  value: ToastItem;
  onClose: (id: string) => void;
}> = ({ value, onClose }) => {
  return (
    <RadixToast.Root
      open={value.isOpen}
      onOpenChange={(isOpen) => !isOpen && onClose(value.id)}
      duration={value.duration}
    >
      {value.type === "success" ? <SuccessIcon /> : <ErrorIcon />}
      <RadixToast.Title>{value.title}</RadixToast.Title>
      {value.description && (
        <RadixToast.Description>{value.description}</RadixToast.Description>
      )}
      <RadixToast.Close>
        <CloseIcon />
      </RadixToast.Close>
    </RadixToast.Root>
  );
};
```

`RadixToast.Root` はトーストのロジックを引き受けるコンポーネントです(それ自体が `li` 要素でもあります)。 `open` に `true` を渡すとトーストが表示され、表示状態が変わると `onOpenChange` イベントが発火します。 `onOpenChange` といっても、`open={true}` の状態でマウントされて閉じるのを待つだけなので、 `onOpenChange` の引数には `false` だけが渡ってきます。なので、 `onOpenChange` の中で `isOpen` が `false` になったときだけ `onClose` を呼び出すようにしています。

その `onClose` ですが、ステート自体はこの親コンポーネントで管理しているため、イベントハンドラーだけを受け取る形にしています。実行時には閉じようとしているトーストの `id` を指定し、あとは親側に処理を任せます。

`ToastItem` の `duration` もここで渡します。 `duration` で指定したミリ秒数が経過すると `onOpenChange` を引数 `false` で発火してくれます。`duration` が未指定の場合の挙動は後述。

`value.type` の値によってアイコンを変えています。もし `value.type` の値を変更する場合は、この `Toast` コンポーネントでバリエーションを増やしていきます。または、タイプ別に `Toast` コンポーネントを用意するのも手かもしれません。

`RadixToast.Close` は `button` 要素をレンダリングします。スタイリング以外は特に渡す必要はなく、クリックするとやはり `onOpenChange` を引数 `false` で発火してくれます。

### ToastProvider を実装する

いよいよトーストの本丸である `ToastProvider` を実装します。 `ToastProvider` は `ToastItem` の配列をステートとして持ち、 `useToast` で配信する関数を実装します。

まず、内部で ID をランダムに生成する関数を用意しておきます。トーストが識別できる程度のランダム性があれば十分なので、 `Math.random` でサクッと作ります。

```tsx :components/Toast.tsx
const genRandomId = () => Math.random().toString(32).substring(2);
```

`ToastProvider` は `ToastItem` の配列をステートとして持つことになっていましたね。持ちましょう。これを素材として `Toast` コンポーネントが複数レンダリングされます。

```tsx :components/Toast.tsx
export const ToastProvider: FC<{ children: ReactNode }> = ({ children }) => {
  const [toasts, setToasts] = useState<ToastItem[]>([]);
  // ...
};
```

続いて、子孫コンポーネントに渡すための、トーストを開く関数を実装します。これは、`toasts` ステートにアイテムを追加するようにステート更新する関数として実装します。引数は事前に定義した `OpenToastParams` です。

```tsx :components/Toast.tsx
const openToast = useCallback((params: OpenToastParams) => {
  const id = genRandomId();
  setToasts((prev) => [...prev, { id, isOpen: true, ...params }]);
}, []);
```

関数の実行によって `id` をランダムに生成します。また、トーストを開くための関数なので、 `isOpen` は固定で `true` にします。それらと引数で受け取った値を 1 つにまとめたオブジェクトを新たな配列の要素として追加し、ステートの更新を行います。

次はトーストを閉じる関数です。これは先に紹介した `Toast` コンポーネントに渡すためのもので、 `id` を受け取ってクローズする対象を特定します。

```tsx :components/Toast.tsx
const closeToast = useCallback((id: string) => {
  setToasts((prev) =>
    prev.map((value) => (value.id === id ? { ...value, isOpen: false } : value))
  );

  setTimeout(() => {
    setToasts((prev) => prev.filter((value) => value.id !== id));
  }, 200);
}, []);
```

注意点としてはいきなり配列ステートから消してしまわずに、一旦削除対象の `isOpen` を `false` にします。これはトーストのフェードアウトアニメーションを待つためです。いきなり配列から削除してしまうと DOM からも消えることになるのでアニメーションが見れません。

`setTimeout` でしばらく待った後に改めてステートから対象を削除します。この待ち時間にあたる `setTimeout` の第 2 引数は、フェードアウトアニメーションの時間以上にしておくと良いです(`duration` とは関係ない値であることに注意。`duration` はトーストを開いてから閉じるまでの待機時間です)。

最後に `ToastProvider` から return する JSX です。トーストを開く関数を配信する `Context.Provider` と、 Radix Toast を使う上で必要な `RadixToast.Provider` で子要素をラップします。

```tsx :components/Toast.tsx
<OpenToastContext.Provider value={openToast}>
  <RadixToast.Provider duration={5000}>
    {children}
    {toasts.map((value) => (
      <Toast key={value.id} value={value} onClose={closeToast} />
    ))}
    <RadixToast.Viewport />
  </RadixToast.Provider>
</OpenToastContext.Provider>
```

`openToast` 関数は `OpenToastContext.Provider` で配信します。これで、子孫コンポーネントは `useToast` によって `openToast` を取得することになります。

`toasts` 配列ステートから `Toast` コンポーネントのレンダリングも行います。その時、`closeToast` 関数を `onClose` に渡しています。

`RadixToast.Viewport` は実際にトーストを画面に表示するエリアになります。ちらっと `RadixToast.Root` が `li` 要素であると書きましたが、 `RadixToast.Viewport` は `ul` 要素になっていて DOM として正しい構造になります。画面右下に fixed するようなスタイリングをするとよいでしょう。

`RadixToast.Provider` には `duration` を渡すことができます。これは `RadixToast.Root` に `duration` を渡さなかった時のデフォルト値になります。`RadixToast.Provider` すら `duration` が渡されなかった場合は 5000 ミリ秒がデフォルト値になります。

ここまでをまとめると、`ToastProvider` は以下のようになります。

```tsx :components/Toast.tsx
import * as RadixToast from "@radix-ui/react-toast";

const genRandomId = () => Math.random().toString(32).substring(2);

export const ToastProvider: FC<{ children: ReactNode }> = ({ children }) => {
  const [toasts, setToasts] = useState<ToastItem[]>([]);

  const openToast = useCallback((params: OpenToastParams) => {
    const id = genRandomId();
    setToasts((prev) => [...prev, { id, isOpen: true, ...params }]);
  }, []);

  const closeToast = useCallback((id: string) => {
    setToasts((prev) =>
      prev.map((value) =>
        value.id === id ? { ...value, isOpen: false } : value
      )
    );

    setTimeout(() => {
      setToasts((prev) => prev.filter((value) => value.id !== id));
    }, 200);
  }, []);

  return (
    <OpenToastContext.Provider value={openToast}>
      <RadixToast.Provider duration={5000}>
        {children}
        {toasts.map((value) => (
          <Toast key={value.id} value={value} onClose={closeToast} />
        ))}
        <RadixToast.Viewport />
      </RadixToast.Provider>
    </OpenToastContext.Provider>
  );
};
```

これで目標としていた最終的な形になりました！あとは使ってみるだけです。

### 使ってみる

Codesandbox を用意しました。`onClick` で `openToast` を呼び出すだけの簡単な例ですが、読みやすくなっていると思います。

@[codesandbox](https://codesandbox.io/embed/laughing-breeze-nmyfx8?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.tsx&theme=dark)

## まとめ

Radix Toast を使いやすくする方法を紹介しました。

React の宣言的な思想とは相反して、トーストは命令的に使えるようになっていると嬉しいです。また、今回の実装によって Radix Toast が一箇所のモジュールに隠蔽されるので、トーストを使うコンポーネントは何も意識する必要がなくなることも嬉しいですね。Radix Toast ではないヘッドレスコンポーネントに乗り換えることになっても、変更が一箇所で済みます。

これでトーストを使うときは

```ts
const toast = useToast();
//...
toast({...});
```

と書くだけでよくなります。簡単！

### 最後に

🍞 ← これはトーストではない

それでは良い React ライフを！
