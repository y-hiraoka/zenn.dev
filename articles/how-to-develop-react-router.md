---
title: "react-router 作り方"
emoji: "🪵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "reactrouter", "history"]
published: true
---

# `history` で自分だけの最強のルーターライブラリを作ろう！

react-router は history というライブラリを内部で使用しています。ブラウザの history API の抽象化を提供するのが目的のライブラリです。
react-router v5 の API に `useHistory` というカスタムフックがあります(v6 から名前が変更されています)が、それはまさに history ライブラリの実体を取得するための関数になっています。

ルーティングに関する以下の処理は history ライブラリ内部で行われています。

- 現在の URL から Location オブジェクトを作成する
- 履歴の操作を行う
- 履歴の変更を検知してコールバックを実行する

react-router は React 用のインターフェイスを提供しているに過ぎません。

つまり、 history をベースに React コンポーネントやカスタムフックを作成すれば、比較的簡単に React 用ルーターライブラリを作成することができるのです。

この記事では history を使って簡易的な react-router を作成する手順を紹介します。

## history ライブラリについて

https://github.com/remix-run/history

history ライブラリのソースコードはかなり少なく、すぐに全体を把握することができます。

### `History` インターフェイス

https://github.com/remix-run/history/blob/8bef6f4d50548f46ab7c97e171b3d8634093e7a7/packages/history/index.ts#L183-L280

history ライブラリはインターフェイスと実装の分離がされています。まずは TypeScript の interface で `History` 型を定義し、history オブジェクトが　プロパティやメソッドを持つことを義務付けます。

後述の `createBrowserHistory()`, `createHashHistory()`, `createMemoryHistory()` はこの `History` インターフェイスを実装したオブジェクトを生成して返します。どの create 関数で生成しても共通のインターフェイスを持つことが約束されているため、 create 関数を差し替えるだけで振る舞いを変更できます。理想的な設計になっていますね。

#### プロパティ

`History.action` と `History.location` が生えています。

`History.action` は最後に行われた履歴操作がなにかを `enum Action` で持っており、 `Pop`, `Push`, `Replace` のいずれかになります。 `History` オブジェクトが生成された直後の値は `Action.Pop` になるようです。

`History.location` は履歴スタックの現在位置(ブラウザの URL バーに表示されているもの)のパスから生成された `Location` オブジェクトが格納されています。

#### メソッド

履歴スタックを移動したり更新したりするためのメソッド(`.push()`, `.go()` 等)と、履歴スタックの変化を検知してコールバック関数を実行する `listen()`, `block()` があります。

前者はパスを指定して遷移したりブラウザの戻るボタンをクリックしたときの挙動を再現したりするのに使用します。
後者は、前者のメソッドやユーザーのブラウザ前後ボタンの操作等を検知したときに発火したい関数を登録しておくことができ、その関数は引数で Location オブジェクトを受け取ることができます。

### `createXXXHistory` 関数

`History` インターフェイスを実装した history オブジェクトを生成する関数が 3 つ用意されています。環境によって使い分けますが、先述の通りすべてが同じインターフェイスを実装しているため、生成する関数を差し替えるだけで振る舞いを変更することが可能です。

#### `createBrowserHistory()`

ブラウザの URL から Location オブジェクトを生成して履歴スタックに保存するような history オブジェクトを生成します。通常の Single Page Application の開発を行う場合はこれを使用します。内部実装では当然ブラウザの history API にがっつり依存しています。

#### `createHashHistory()`

URL のハッシュ部分 (`http://example.com/#hoge` の `#hoge`) のみから Location オブジェクトを生成して履歴スタックに保存するような history オブジェクトを生成します。ブラウザの URL バーに表示される実際の URL は以下のようになります。

```
https://example.com/#/foo/123/bar/456?hoge=fuga#hash
```

ドメインのすぐ後に `#` が置かれることで、ハッシュにパスとクエリパラメーターと論理的(?)なハッシュを詰め込んでいます。
つまりブラウザから見ると URL のパスの変更は起こっていないが、URL と Web アプリの対応付けができているという状態を実現できます。

これの使い所は、例えば GitHub Pages にデプロイするときなどが挙げられます。GitHub Pages の URL は、ユーザーサイトの場合 `https://user-name.github.io` で、プロジェクトサイトの場合は `https://user-name.github.io/repository-name` が割り当てられます。ユーザーサイト側でルーティングをする際にパス名がプロジェクトサイトのリポジトリ名とかぶってしまうと、URL の衝突が起こってどちらかが表示されないという事態になります。そんなときに `createHashHistory()` でルーティングを実装していれば、パスの変更が発生しないため URL の衝突を考慮する必要がなくなります。

#### `createMemoryHistory()`

ブラウザの URL や history API には依存せず、文字通りメモリ上だけで履歴スタックを管理する history オブジェクトを生成します。ブラウザ API に依存しない、 React Native やテスト環境で使用されることが想定されています。

## react-router の簡易的な作り方

ここから本題に入って、 history ライブラリを使用して react-router を作る手順を説明します。なお、完全な API の再現ではなくむしろ足りない機能があったり TypeScript friendly ではなかったりしますが、そこは大目に見てください。また、 react-router のソースコードをそのまま載せるのではなく、(参考程度に読んではいますが)我流のものを紹介しています。

模倣する react-router のバージョンは v5 です(v6 はまだちゃんと使ったことがない)。

コード例で `import` 文は省略することが多いですが、適宜モジュール上部に記載してあるとして読んでください。 `History` や `Location` の型定義は TypeScript 標準ライブラリに含まれるものと history ライブラリが export しているものがありますが、前者は一切使用していないので、これらの使用が見られる箇所は `import {History, Location} from "history";` が前提です。

### Context を用意する

React Context を 3 つ用意します。

```tsx
const HistoryContext = createContext<History | undefined>(undefined);
const LocationContext = createContext<Location | undefined>(undefined);
const PathContext = createContext<string | undefined>(undefined);
```

それぞれ、 history オブジェクトを配信する用、 location オブジェクトを配信する用、 path 文字列を配信する用です。
この段階で、 `useHistory` や `useLocation` の実装もすぐイメージできるかもしれません。
path 文字列とは、ここでは `"/foo/:fooId/bar/:barId"` のようなものを想定しています。 react-router で `Route` に props として渡すアレですね。なぜ Context に乗せるかはあとでわかります。

`undefined` 許容型にして初期値を `undefined` としているのは、 `Provider` で囲われなかったときにエラーにしたいからです(後述します)。

### Router コンポーネント

`Router` コンポーネントを定義します。(今回は `createBrowserHistory` 用だけ作ります。)

```tsx
export const Router: VFC<{ children: ReactNode }> = ({ children }) => {
  return (...);
};
```

`Router` コンポーネントの役割は以下のとおりです。

- history オブジェクトの生成と保持
- location ステートの保持
- 3 つの Context の Provider を配置

history オブジェクトは一度生成されたら参照が変更されないミュータブルなインスタンスなので `useRef` で保持しておきます。

```tsx
const historyRef = useRef<History>();
if (historyRef.current === undefined) {
  historyRef.current = createBrowserHistory();
}
```

初期値を渡さずに ref オブジェクトを宣言し、 `Router` コンポーネントの初回評価時のみ `createBrowserHistory` が実行されるように条件分岐します。

location オブジェクトを保持しておくステートを宣言します。 `useState` に渡す初期値は history オブジェクトの location プロパティから取り出します。

```tsx
const [location, setLocation] = useState(historyRef.current.location);
```

そして `history.listen` に `setLocation` を仕込んでおくことでロケーションが変化したら最新の location オブジェクトでステート更新できるようにしておきます。

```tsx
useLayoutEffect(() => {
  if (historyRef.current) {
    return historyRef.current.listen(({ location }) => setLocation(location));
  }
}, []);
```

`history.listen` はリスナーの登録を解除する関数を return してくれるので、 `useLayoutEffect` の中からさらに return することでアンマウント時にリスナーが解除されるようにします(`Router` コンポーネントはおそらくアプリのトップに配置されるのでアンマウントされることはないと思いますが一応)。

そして `Context.Provider` に値を渡して JSX を return します。

```tsx
return (
  <HistoryContext.Provider value={historyRef.current}>
    <LocationContext.Provider value={location}>
      <PathContext.Provider value="/">{children}</PathContext.Provider>
    </LocationContext.Provider>
  </HistoryContext.Provider>
);
```

`location` は都度更新されていくステートなので、 `useContext(LocationContext)` を実行している子コンポーネントは URL の変更によって再レンダリングがされることがわかりますね。

`PathContext.Provider` の `value` には固定で `"/"` を渡しています。 `PathContext` の値は後述する `Route` が設定してくれるので、 `Router` はフォールバックだけセットしておきます。

:::details Router コンポーネント全文

```tsx
export const Router: VFC<{ children: ReactNode }> = ({ children }) => {
  const historyRef = useRef<History>();
  if (historyRef.current === undefined) {
    historyRef.current = createBrowserHistory();
  }

  const [location, setLocation] = useState(historyRef.current.location);

  useLayoutEffect(() => {
    if (historyRef.current) {
      const unlisten = historyRef.current.listen(({ location }) => setLocation(location));
      return unlisten;
    }
  }, []);

  return (
    <HistoryContext.Provider value={historyRef.current}>
      <LocationContext.Provider value={location}>
        <PathContext.Provider value="/">{children}</PathContext.Provider>
      </LocationContext.Provider>
    </HistoryContext.Provider>
  );
};
```

:::

### `useHistory` フック

history オブジェクトを使用したいコンポーネントで実行するカスタムフックを作成します。と言っても難しいことはしなくて、 `useContext` をラップするだけです。

```tsx
export function useHistory() {
  const history = useContext(HistoryContext);
  if (history === undefined) {
    throw new Error("Component must be wrapped with Router.");
  }

  return history;
}
```

`HistoryContext` には `Router` の配下であれば必ずインスタンスが格納されています。逆に言えば `Router` で囲っていなければ `undefined` しか取り出せません。 `Router` で囲っていない位置で `useHistory` が実行された場合は例外とみなしてエラーを throw してしまいます。TypeScript にとっては型の絞り込みの意味もあります。 `useHistory` の戻り値の型が `undefined` を含まなくなるため、 `undefined` チェックが不要になり扱いやすくなります。

### `useLocation` フック

location オブジェクトを使用したいコンポーネントで実行するカスタムフックを作成します。こちらも `useContext` をラップするだけですが。

```tsx
export function useLocation() {
  const location = useContext(LocationContext);
  if (location === undefined) {
    throw new Error("Component must be wrapped with Router.");
  }

  return location;
}
```

`useHistory` 同様に `undefined` の場合は `Router` でラップされていないということなので例外とみなしてエラーを throw します。

### `useRouteMatch` フック

`path` 文字列を渡すと現在の location と比較してマッチしているかどうか判定してくれるカスタムフックを作成します。判定しつつパスパラメーターの抽出もできるようにします。といってもこれを自力で実装するのは普通に難しいので、 本家 react-router v5 も依存していた(v6 から依存から外れた) path-to-regexp を使用します。

```
npm install path-to-regexp
```

フックとしてではなくマッチ判定ロジックだけを別の場所で使いたいので、カスタムフックを作成する前に判定ロジックを純粋関数として定義しておきます。

```tsx
import { match } from "path-to-regexp";

function matchPath<T extends Record<string, string>>(
  path: string,
  currentPath: string,
  exact: boolean
) {
  const matcher = match<T>(path, { end: exact });
  const result = matcher(currentPath);

  return result ? result : null;
}
```
次のような使い方を想定しています。

```ts
const matched = matchPath<{ fooId: string }>("/foo/:fooId", "/foo/123", true);
console.log(matched !== null ? matched.params.fooId : "not matched");
```

第 1 引数のパステンプレートに第 2 引数のパスがマッチしたら、パスパラメータを内包したオブジェクトを返します。マッチしなかったら null を返します。boolean 型の第 3 引数 `exact` は、パスの判定時に前方一致か完全一致かを指定します。 `true` の場合は完全一致です(`"/foo/:fooId"` に対して `"/foo/:fooId/bar"` がマッチしない)。
パスパラメータの型は template literal types で推論可能ですが、実装をサボっています。まじめに作る場合は推論できるようにしたいところですね。

この `matchPath` を使って `useRouteMatch` を実装していきます。

```tsx
export type Matched<P extends Record<string, string>> = {
  params: P;
};
export function useRouteMatch<P extends Record<string, string> = {}>(option: {
  path: string;
  exact: boolean;
}): Matched<P> | null {
  const location = useLocation();

  const matched = useMemo(
    () => matchPath<P>(option.path, location.pathname, option.exact),
    [option.path, option.exact, location.pathname]
  );

  return matched;
}
```

確認したいパステンプレートと完全一致かどうかのフラグは引数で受け取り、現在 URL のパス情報は `useLocation` から取得します。

`matchPath` はオブジェクトを返却するため、 `useMemo` で囲っておくのがマナーです。 `useRouteMatch` を使う側が戻り値をそのまま `useEffect` 等の依存配列に突っ込む可能性もあるため、不必要な参照の変化は避けます。

### `useParams` フック

現在のパスからパスパラメータを抽出するカスタムフックを定義します。といっても `useRouteMatch` がパスパラメータの抽出までやってくれるため、その処理はそちらに委譲します。

パステンプレートは `PathContext` から取り出します。その値は `Route` がセットしてくれるはずなので、 `useParams` を使用するコンポーネントの祖先に `Route` がいないと効果を発揮しないということですね。

```tsx
export function useParams<T extends Record<string, string>>(): T {
  const path = useContext(PathContext);
  if (path === undefined) {
    throw new Error("Component must be wrapped with Router.");
  }

  const matched = useRouteMatch<T>({ path, exact: false });
  if (matched === null) throw new Error();
  return matched.params;
}
```

同様に `path` の `undefined` チェックを行います。 `undefined` になるのは `Router` の外で実行された場合に限られます。

`path` を `useRouteMatch` に渡してマッチしているかを判定します。と言っても `path` は `Route` がセットし、かつ `Route` はマッチしていない場合は子コンポーネントをレンダリングしません。 `Route` に囲われていなかったとしても `Router` が `"/"` をセットしていて、かつ前方一致の判定(`exact: false`)のため 100％ マッチします。つまりここで `useRouteMatch` が null を返すことはありえません。型の絞り込みのためだけにエラーを throw して、 `.params` を return します。

### `Route` コンポーネント

`Route` は指定したパステンプレートにマッチしたときだけ子コンポーネントをレンダリングするコンポーネントです。
まずは Props を考えましょう。必要なものはパステンプレート、完全一致かどうか、子コンポーネントですね。

```tsx
export type RouteProps = {
  path: string;
  exact?: boolean;
  children: ReactNode;
};
```

`exact` は optional にしています。渡されなかった場合は `false` にしておきます。`path` と `children` は必須です。 `Route` の役割を考えたらこれは必然的ですね。

`Route` 内部では現在のロケーションが `path` にマッチしているかを判定しますが、これは `useRouteMatch` で判定可能なのでこれを使います。また props に渡された `path` を `PathContext.Provider` に渡すことによって子孫コンポーネントが `useParams` を実行したときにパスパラメータを取得できるようにしておきます。

```tsx
export const Route: VFC<RouteProps> = ({ path, exact = false, children }) => {
  const matched = useRouteMatch({ path, exact });

  if (matched === null) return null;

  return <PathContext.Provider value={path}>{children}</PathContext.Provider>;
};
```

### `Link` コンポーネント

続いては `a` 要素の代替として使用する `Link` コンポーネントを作成します。 `LinK` は `a` 要素をレンダリングしますが、クリックイベントを奪って代わりに history オブジェクトの `.push()` で遷移できるようにします。

まずは Props を検討します。必要なものは `href` に相当するものですが、 `a` に渡す `href` 文字列は `history.createHref` で生成したいです。なので、 `Link` コンポーネントが受け取るものは `history.createHref` の引数に相当する `To` 型の値になります。また `history.push` による遷移時には location state を渡すことができます。それも props で受け取っておきます。そして `href` を除く `a` 要素のすべての属性を受け取れるようにしておくことで、 `href` 以外は `a` 要素と同じように扱えるようにします。

```tsx
export type LinkProps = {
  to: To;
  state?: unknown;
} & Omit<ComponentProps<"a">, "href">;
```

`ComponentProps<"a">` は `a` 要素が受け取る props の型定義を指しています。 `Omit` を使って `a` 要素の props のうち `href` 以外のものと、 `Link` 専用の型定義をマージしています。

`Link` コンポーネントは `forwardRef` で定義する必要があります。使う側が `a` 要素の実 DOM を握りたいことがあったり、 Chakra などの UI コンポーネントと組み合わせる場合は ref にアクセスできることが前提になっている場合があります。それらのユースケースに対応できるようにしておくために `forwardRef` を使います。

```tsx
export const Link = forwardRef<HTMLAnchorElement, LinkProps>(
  ({ to, state, children, target, onClick, ...props }, forwardedRef) => {

    return (...);
  }
);
```

まずは `useHistory` を使って history オブジェクトを取得します。

```tsx
const history = useHistory();
```

`a` 要素に渡すクリックイベントを実装します。 `preventDefault` で標準の動作を停止して、代わりに history で URL を書き換えます。

```tsx
const handleClick: React.MouseEventHandler<HTMLAnchorElement> = (event) => {
  if (onClick) {
    onClick(event);
    if (event.defaultPrevented) return;
  }

  if (event.button === 0 && (!target || target === "_self")) {
    event.preventDefault();
    history.push(to, state);
  }
};
```

`Link` に `onClick` が渡されている場合はこの中で実行してあげます。 `onClick` の中ですでに `preventDefault()` が実行された場合は、 `Link` を使う側が URL の遷移を停止したいものとみなしてそこで関数の実行を終了します。

続いて `event.button === 0`(左クリックのとき) かつ `target` が未指定または `"_self"` のときに標準動作を停止した上で `history.push` を実行します。 props で受け取っている `to` と `state` を渡します。

最後に `a` 要素を return します。 分割代入しておいた変数も漏れなく props に渡しておきます。

```tsx
return (
  <a
    {...props}
    ref={forwardedRef}
    href={history.createHref(to)}
    target={target}
    onClick={handleClick}
  >
    {children}
  </a>
);
```

先述の通り `href` は `history.createHref` によって生成します。 `to` は同じインターフェイスなのでそのまま引数に渡すことが可能です。

:::details Link コンポーネント全文

```tsx
export type LinkProps = {
  to: To;
  state?: unknown;
} & Omit<ComponentProps<"a">, "href">;

export const Link = forwardRef<HTMLAnchorElement, LinkProps>(
  ({ to, state, children, target, onClick, ...props }, forwardedRef) => {
    const history = useHistory();

    const handleClick: React.MouseEventHandler<HTMLAnchorElement> = (event) => {
      if (onClick) {
        onClick(event);
        if (event.defaultPrevented) return;
      }

      if (event.button === 0 && (!target || target === "_self")) {
        event.preventDefault();
        history.push(to, state);
      }
    };

    return (
      <a
        {...props}
        ref={forwardedRef}
        href={history.createHref(to)}
        target={target}
        onClick={handleClick}
      >
        {children}
      </a>
    );
  }
);
```

:::

### `Switch` コンポーネント

`Switch` コンポーネントは、子コンポーネントとして並列に配置された複数の `Route` コンポーネントから、現在のパスにマッチしている最初のものだけをレンダリングする機能を持ちます。

現在のパスにマッチしているかどうかで分岐を行うため、まずは `useLocation` を使いましょう。

```tsx
export const Switch: VFC<{ children: ReactNode }> = ({ children }) => {
  const location = useLocation();

  return ...;
};
```

`children` に渡されてくる ReactElement を解析する API が React には公開されています。ここでは `Children` と `isValidElement` を使って `children` のうち `Route` かつマッチしているものだけを選択します。

```tsx
let matchedRoute: ReactElement | null = null;

Children.forEach(children, (child) => {
  if (matchedRoute !== null) return;

  if (!isValidElement(child) || child.type !== Route) {
    console.error("Switch can have only Route components.");
    return;
  }

  const matched = matchPath(child.props.path, location.pathname, child.props.exact);

  if (matched !== null) {
    matchedRoute = child;
  }
});

return matchedRoute;
```

`isValidElement` によって `child` が ReactElement かどうかをチェックすることができます。 TypeScript 的には `isValidElement` が Type Guard として型定義してあるので `child` の型が絞り込まれて `.props` や `.type` プロパティにアクセスできるようになります。

`child.type` は例えば `div` 要素の場合は `"div"` 文字列が、 `Route` コンポーネントの場合は `Route` 関数のインスタンスが格納されています。

もし渡された `children` の一つが `Route` ではなくても無視するだけで Error を throw したりはしませんが、開発者に伝わるように `console.error` を残しておきます(本当は development ビルドのときだけ `console.error` するみたいな処理が理想です)。

`child.type === Route` を満たし、かつ `useRouteMatch` のために作った `matchPath` を流用して判定した結果マッチしたものを return します。これで `Switch` コンポーネントの要件を満たすことが可能です。

:::details Switch コンポーネント全文

```tsx
export const Switch: VFC<{ children: ReactNode }> = ({ children }) => {
  const location = useLocation();

  let matchedRoute: ReactElement | null = null;

  Children.forEach(children, (child) => {
    if (!isValidElement(child) || child.type !== Route) {
      console.error("Switch can have only Route components.");
      return;
    }

    const matched = matchPath(child.props.path, location.pathname, child.props.exact);

    if (matched !== null) {
      matchedRoute = child;
    }
  });

  return matchedRoute;
};
```

:::

### 全体のソースコード

この記事で作成するものは以上です。マッチしているときだけ適用されるクラス名 `activeClassName` を渡せる `NavLink` や、マウントと同時に画面遷移を行う `Redirect`など作成していない API がいくつかありますが、応用すれば簡単に作成することができます。

@[codesandbox](https://codesandbox.io/embed/zenn-original-react-router-with-history-mt6fz?autoresize=1&fontsize=14&hidenavigation=1&module=%2Fsrc%2Frouter.tsx&theme=dark&view=editor)

## 動作確認

実際に作ったものが react-router のように使えるか確認しましょう。

まずは `Router` コンポーネントをアプリのトップに配置します。

```tsx
render(
  <Router>
    <App />
  </Router>,
  rootElement
);
```

`App` コンポーネントで `Link` を配置していきます。 location state も渡っていくか確認するために props に適当なオブジェクトを渡しておきます。
また、 `Switch` の中に `Route` を配置していきます。パスはなんかそれっぽく(適当)。

```tsx
const App: VFC = () => {
  return (
    <div>
      <div style={{ display: "flex", gap: 16 }}>
        <Link to="/">Home</Link>
        <Link to="/stin">Profile</Link>
        <Link to="/stin/settings" state={{ foo: "bar" }}>
          Settings
        </Link>
      </div>
      <Switch>
        <Route path="/:userId/settings">
          <UserSettings />
        </Route>
        <Route path="/:userId">
          <UserProfile />
        </Route>
        <Route path="/">
          <Home />
        </Route>
      </Switch>
    </div>
  );
};
```

`UserSettings` や `UserProfile` ではパスパラメータの `userId` が取れるはずなので `useParams` を内部で使いましょう。 location state は `useLocation` から取得できます。

```tsx
const UserSettings: VFC = () => {
  const { userId } = useParams<{ userId: string }>();
  const { state } = useLocation();

  return (
    <div>
      <div>{userId}</div>
      <div>location state: {JSON.stringify(state)}</div>
    </div>
  );
};
```

では動かしてみましょう。

@[codesandbox](https://codesandbox.io/embed/zenn-original-react-router-with-history-mt6fz?fontsize=14&hidenavigation=1&theme=dark&view=preview)

リンクをクリックすると `Route` と `Switch` で出し分けている部分が切り替わります。 Settings のリンクでは、 location state もちゃんと表示されますね。
最低限の機能を持ったルーティングライブラリが完成しました。

## まとめ

この記事では history ライブラリを使用して react-router を自作する方法を紹介しました。 history ライブラリは react-router だけでなく、新興で react-query と同じ開発者が公開している [react-location](https://github.com/tannerlinsley/react-location/blob/b4e39dba8080914221707a06be910b62eb7fd164/packages/react-location/package.json#L98) や、 TypeScript の型安全性を第一に設計されたルーターライブラリの [Rocon](https://github.com/uhyo/rocon/blob/53cfce14dd4654686720663645519a8e600a0822/package.json#L86) が依存しており、非常に使い勝手がよいのがわかります。

react-router の API を模倣をしてきましたが、自由にインターフェイスを組めば自分好みの使いやすさを持ったルーターライブラリを実装可能ということです。ぜひ最強のルーターライブラリを自作してみてください(自己責任で)。

僕自身もルーターライブラリを作ってみたいと思い、APIを考えている途中です。完成したらまた記事にしたいと思います。

それではよい React ライフを！