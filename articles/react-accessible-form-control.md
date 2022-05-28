---
title: "【React】Formの入力要素を簡単にアクセシブルにするヘッドレスコンポーネント"
emoji: "🙈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

## 入力要素のアクセシビリティ問題

フォームの入力要素(`input`, `select`, `textarea`)周りの HTML 構造は、おおよそ次のようになることが多いと思います。

```tsx
<div>
  <label>おなまえ</label>
  <input />
  <div>一般には公開されません</div>
  {isInvalid && <div>100文字以内で入力してください</div>}
</div>
```

もちろんこのままではアクセシビリティの観点から次のような問題点が挙げられます。

- label と input が紐付いていない
- input と補足説明が紐付いていない
- input とエラーメッセージが紐付いていない

これらの解決策に共通するのが `id` 属性によって要素同士に紐付けを与える方法です。

label と input の紐付けについては input を label の子要素とすることで達成することができますが、 label に対するスタイルが input にカスケードするので個人的にはあまり好きではありません。

上記のサンプルコードに適切に `id` 属性と紐付けを適用すると次のようなコードになります。

```tsx
<div>
  <label htmlFor="name-input">おなまえ</label>
  <input
    id="name-input"
    aria-describedby="name-input-helper-text"
    aria-errormessage="name-input-error-message"
  />
  <div id="name-input-helper-text">一般には公開されません</div>
  {isInvalid && <div id="name-input-error-message">100文字以内で入力してください</div>}
</div>
```

`id` を付与する箇所と、紐付けのための `htmlFor` や `aria-*` が渡されました。しかしこれを手動で設定するのは取り違いがありそうです。例えば、 `"name-input-helper-text"` を `aria-errormessage` に渡してしまうかもしれません。

また、 `id` のような一意性が求められる値を人間が決めることも気になります。 `id` を指定する時に必ず値が重複しないかどうかを考慮する必要が出てくるからです。これについては React18 で登場した `useId` で解決できますが([参考記事](https://zenn.dev/takepepe/articles/useid-for-a11y))。

## ChakraUI による解決策

これを React パワーを使って解決しているのが ChakraUI の `FormControl` コンポーネントです。

```tsx
import {
  FormControl,
  FormLabel,
  FormErrorMessage,
  FormHelperText,
  Input,
} from '@chakra-ui/react'

...

<FormControl isInvalid={isInvalid}>
  <FormLabel>おなまえ</FormLabel>
  <Input />
  <FormHelperText>一般には公開されません</FormHelperText>
  <FormErrorMessage>100文字以内で入力してください</FormErrorMessage>
</FormControl>
```

`FormControl` コンポーネントは内部では `Context.Provider` になっていて、子コンポーネントになった `FormLabel`, `Input`, `FormHelperText`, `FormErrorMessage` に対して、機械的に決定された `id` や `htmlFor`, `aria-describedby` などを付与してくれます。また `FormControl` の `isInvalid` props の値によって `FormErrorMessage` の表示可否が勝手に決まるので、条件分岐が散らばることもありません。

自分で `id` を振って回る必要がないので、コードがとてもスッキリして要素の構造がわかりやすくなっています。 要素間の関連を `id` による紐付けの代わりに(補足説明やエラーメッセージも含めて)入れ子構造によって表現しているとも言えます。

`FormHelperText` や `FormErrorMessage` は時には不要です。 それらを使わなかったとしても `Input` コンポーネントの `aria-describedby` 属性には必要十分な `id` だけが渡されます。

ChakraUI の設計思想のひとつにアクセシビリティの担保が掲げられており、それを果たすためのコンポーネント機能であることが窺えます。

https://chakra-ui.com/guides/principles

## ヘッドレスなコンポーネントによる解決策

しかし上記は ChakraUI のコンポーネントです。 `ChakraProvider` の中でしか使えないし、 ChakraUI のスタイリングがべっとりと塗られています。 [HeadlessUI](https://headlessui.dev/) のように、機能だけ提供されて色付けは自分たちで完全に決定できると嬉しいですよね。

ということで ChakraUI のソースコードを参考に書いてみたのが次の TypeScript ファイルです(npm には公開していません。コピペで使うつもり)。

https://github.com/y-hiraoka/react-accessible-form-control/blob/main/src/form-control.tsx

ChakraUI とほとんど同じ使い勝手で機能だけを享受できるように実装してあります。

```tsx
import {
  FormControl,
  FormErrorMessage,
  FormHelperText,
  FormInputControl,
  FormSelectControl,
  FormLabel,
  FormTextAreaControl,
} from "./form-control";

...

<FormControl isInvalid={isInvalid}>
  <FormLabel>おなまえ</FormLabel>
  <FormInputControl />
  <FormHelperText>一般には公開されません</FormHelperText>
  <FormErrorMessage>100文字以内で入力してください</FormErrorMessage>
</FormControl>
```

これがレンダリングされると(`isInvalid` が `false` の場合)次のような HTML 構造になります。

```html
<div role="group">
  <label for=":r1:-control">おなまえ</label>
  <input
    id=":r1:-control"
    aria-describedby=":r1:-control-helper-text"
    aria-invalid="true"
    aria-errormessage=":r1:-control-error-message"
  />
  <div id=":r1:-control-helper-text">一般には公開されません</div>
  <div aria-live="polite" id=":r1:-control-error-message">
    100文字以内で入力してください
  </div>
</div>
```

正しく `id` や `aria-describedby` などが設定されていますね。 `id` も自動で決定してくれるため、重複の考慮をする必要はありません。

各コンポーネントのインターフェイスはプレーンな JSX 要素そのままにしてあり `forwardRef` もしてあるので、スタイリングも素直にできるし、 react-hook-form との相性も良いです。

```tsx
const { register, formState } = useForm<{ name: string }>();

<FormControl
  isInvalid={formState.errors.name !== undefined}
  className={styles.formControl}
>
  <FormLabel className={styles.formLabel}>おなまえ</FormLabel>
  <FormInputControl
    className={styles.formInputControl}
    {...register("name", { required: true, maxLength: 100 })}
  />
  <FormHelperText className={styles.formHelperText}>
    一般には公開されません
  </FormHelperText>
  <FormErrorMessage className={styles.formErrorMessage}>
    100文字以内で入力してください
  </FormErrorMessage>
</FormControl>;
```

## 実装のポイント

### `FormControl` について

`FormControl` の props 型定義は次のようになっています。

```tsx
export type FormControlProps = {
  id?: string;
  isRequired?: boolean;
  isReadOnly?: boolean;
  isDisabled?: boolean;
  isInvalid?: boolean;
  children: ReactNode;
} & ComponentProps<"div">;
```

入力要素の必須とかバリデーション違反とかの状態を `FormControl` で受け取るようにしています。入力要素はもちろん `FormErrorMessage` も `isInvalid` を参照する必要があるためこの構成になっています。これらの値は React Context に格納されて子コンポーネントに配信されます。子コンポーネントは `useContext` によって受け取り、適切な属性に変換して使用します。

その他に、子要素に割り振るための `id` をすべて `FormControl` が生成して Context で配信します。

```tsx
const FormControl = ({ id }) => {
  const alterId = useId(); // props.id が指定されない場合は内部生成

  const controlId = (id || alterId) + "-control";
  const helperTextId = controlId + "-helper-text";
  const errorMessageId = controlId + "-error-message";

  /*...*/
};
```

### `id` の生成

`id` の採番には React18 の `useId` を使用しています。SSR 環境でもサーバーとクライアントで同じ文字列が得られることを保証してくれるので便利です。SSR なしの SPA 環境であれば、下のようなカスタムフックで簡単に代用できます(ランダム文字列生成方法の参考は[こちら](https://gist.github.com/sadnessOjisan/f76c8d0c920ee9b373ec9c79668168a6))。

```ts
export function useId(): string {
  const ref = useRef(Math.random().toString(36).slice(-8));
  return ref.current;
}
```

また、 `id` は `FormControl` の props から注入できるようにもしています。ユースケースとしてランダム文字列ではなく人間が予測可能な値をセットしたいという要望はありそうだし、何よりテストを書くのが楽になります。

### HelperText や ErrorMessage を持っているかの判定

`FormControl` の子コンポーネントとして `FormHelperText` や `FormErrorMessage` が存在するかの確認はかなりハック的な書き方をしています。

`FormControl` に `useState` で `hasHelperText` と `hasErrorMessage` を boolean 型で宣言しておき、 `FormHelperText` と `FormErrorMessage` の DOM 要素である `div` の `ref` props に setter 関数を仕込んでいます。

```tsx: FormControl
export const FormControl = forwardRef<HTMLDivElement, FormControlProps>(
  function FormControl({...}, forwardedRef) {
    const [hasHelperText, setHasHelperText] = useState(false);
    const [hasErrorMessage, setHasErrorMessage] = useState(false);

    const getHelperTextProps: getFeedbackProps = useCallback(
      (forwardedRef) => {
        return {
          id: helperTextId,
          ref: (node) => { // ref callback に setHasHelperText(true) を仕込む
            assignRef(forwardedRef, node);
            setHasHelperText(true);
          },
        };
      },
      [helperTextId]
    );

    /*...*/
  }
);
```

```tsx: FormHelperText
export const FormHelperText = forwardRef<HTMLDivElement, HelperTextProps>(
  function FormHelperText(props, forwardedRef) {
    const { getHelperTextProps } = useContext(FormControlContext) ?? {};

    return <div {...props} {...getHelperTextProps?.(forwardedRef)} />;
  },
);
```

`ref` は `useRef` で宣言した `RefObject` 以外にも `RefCallback` を渡す事が可能です。 `RefCallback` は対応する実 DOM が描画されると実行されるため、この中で `setHasHelperText` を実行しておけば `FormHelperText` の存在を `FormControl` に伝えることができます。

それって `useEffect` 外の副作用じゃんって思われるかもしれません。僕も ChakraUI のコードを読みながら思いました。 `setHasHelperText` を ref に仕込む代わりに Context に載せて、 `FormHelperText` 側の `useEffect` で `setHasHelperText(true)` を実行しても `FormControl` に自己の存在を伝えられます。ただ `useEffect` は SSR 環境では実行されないので SSR で生成した HTML 文字列上では要素の紐付けはできません。SSR 結果の HTML 文字列上でも紐付けさせるために `ref` に setter を仕込んでるのかなと思い、試しに Next.js でビルドしてみたけど `useEffect` 方法と同じく要素間の紐付けはされませんでした…。この点に関しては「調べたけどよくわかりませんでした。いかがでしたか！」です。

### 入力要素の props を作るためのカスタムフック

入力要素に渡すアクセシビリティのための props を生成するためのカスタムフックを単独で用意しました。 `useContext` で `FormControl` が配信する値を取得して、入力要素用の props に変換する役割を持ちます。

https://github.com/y-hiraoka/react-accessible-form-control/blob/main/src/form-control.tsx#L195-L257

これによって `input` でも `select` でも `textarea` でも、このカスタムフックで取得した props をスプレッド構文で埋め込むだけでよくなります。

https://github.com/y-hiraoka/react-accessible-form-control/blob/main/src/form-control.tsx#L259-L292

また、このカスタムフックを export しておけば、入力コンポーネントを別で用意することも可能です。

## 微妙な点

ChakraUI の `FormControl` のヘッドレス版があると便利だと思って作ったこのモジュールですが、微妙な点もあります。

### 実装を知っている状態に妥協する必要がある

あるコンポーネントを使う時にそれがどんな DOM 要素になってどんな属性を持つかを知る必要があるのは React のアンチパターンとして有名です。しかし今回のモジュールでは、入力要素である `FormInputControl` を使う時、どうしても「それが input 要素で、 `readonly` や `aria-invalid` を持つことを知っている」状態でスタイリングをしなければなりません。

HeadlessUI に倣って `className` を関数型で受け付けるという方法もあります。次のようにすれば、内部状態の `isInvalid` がインターフェイスに表出して、それに依存したスタイリングが可能です。

```tsx
<FormInputControl
  className={({ isInvalid }) => (isInvalid ? "invalid-input" : "valid-input")}
/>
```

ただ、 input 要素に適用する CSS は、 `invalid-input` のようなクラス名ではなくて `.input[aria-invalid="true"]` のような状態依存なセレクターを書きたいですよね。これを考慮すると、 `FormInputControl` の「実装を知っている状態」に妥協する必要があるなぁと感じています。

### HTML のネイティブなバリデーション機能を利用不可

HTML の入力要素には JavaScript を使わずとも入力を検証する機能を持つ属性があります。

```html
<!-- 100文字以内の英数字 -->
<input maxlength="100" pattern="^[\w]*$" class="custom-input" />
```

入力値がこれらの検証に違反する場合、CSS の `:invalid` 擬似クラスが有効になり、それを利用したスタイリングが可能です。

```css
.custom-input:invalid {
  border-color: red;
}
```

しかし今回作成したモジュールでは、 JavaScript で行った入力検証結果の boolean 値を `FormControl` の `isInvalid` props に渡し、 `FormInputControl` はそれを `aria-invalid`に渡すだけです。input 要素は自分自身がどんな入力検証を行うべきかを指示されていないので、`:invalid` 擬似クラスが有効になることはありません。

例えば有名な CSS フレームワークである Bootstrap は `:invalid` 擬似クラスによるセレクターを採用しているため、 `invalid` なスタイリングを適用できず相性が悪いと言えます。

https://getbootstrap.jp/docs/5.0/forms/validation/#how-it-works

## まとめ

ChakraUI の `FormControl` を参考にしたアクセシブルでヘッドレスな入力コンポーネントの実装を紹介しました。

プロジェクトごとに実装を調整する可能性があったりスタイリング手法別の相性の良し悪しがあり npm に公開することはしていませんが、便利なものが用意できているのではないかと感じています。この記事が誰かの参考になれば幸いです。

こちらのリポジトリはそのまま vite 製の react-hook-form と組み合わせたサンプルアプリになっているのでよかったら見てみてください。

https://github.com/y-hiraoka/react-accessible-form-control

それではよい React ライフを！
