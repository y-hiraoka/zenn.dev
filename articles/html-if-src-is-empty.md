---
title: "空世界 〜HTMLの永遠仕様探訪記、或いは、文字なきsrcにまつわる寥々たる考察について〜"
emoji: "🌦️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["html"]
published: false
publication_name: chot
---

## 問題

`<img src="">` をブラウザで表示した時、どうなるか知っていますか？わざわざimg要素の`src`属性を空文字列にする機会がないので意外と知らないかもしれません。

もちろん画像は表示されず、(指定していれば)altが表示されます。

![src属性が空文字列のimg要素を表示しようとした画面のスクリーンショット。壊れた画像のアイコンとaltテキスト「srcが空だよ」が代わりに表示されている](/images/html-if-src-is-empty/empty-src-image-representation.png)

img要素の`src`属性を空文字列にすると、リンク切れになることがわかりました！いかがでしたか？(？)

### そのときHTMLImageElementは

JavaScriptで`src`が空文字列のimg要素のDOMインスタンスを確認してみましょう。https://zenn.dev/stin を開き、Chrome開発者ツールを使って`src`属性に空文字列を指定したimg要素を埋め込んでおきます。

そして次のJavaScriptを実行します。

```jsx
const element = document.querySelector('img[src=""]');
console.log(element.src);
```

![直前のコードブロックを実行した結果、「https://zenn.dev/stin」と表示されていることを示したスクリーンショット](/images/html-if-src-is-empty/element-src-on-console.png)

なんとそのHTMLを取得するURLが表示されました。

空文字列だったはずなのになぜこのような挙動になるのでしょうか。これに疑問を感じ、どのように仕様が定義されているのか調べるため、HTML Living Standardを探ったお話です。

## HTML仕様書を開く

HTML仕様書のimg要素について記載されているページは次です。

https://html.spec.whatwg.org/multipage/embedded-content.html#the-img-element

MDNで探している項目の説明ページを開くと仕様書へのリンクが貼ってあるのでたどり着きやすいです。

## `src`属性とは

先程のリンクを開いて少し下にスクロールした箇所に求めていそうな記述がありました。

> The [`src`](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-img-src) attribute must be present, and must contain a [valid non-empty URL potentially surrounded by spaces](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#valid-non-empty-url-potentially-surrounded-by-spaces) referencing a non-interactive, optionally animated, image resource that is neither paged nor scripted.

> (DeepL訳、一部筆者により修正。以下翻訳は同様)
> src属性は存在しなければならず、ページングもスクリプトもない、非インタラクティブでアニメーションも可能な画像リソースを参照する、valid non-empty URL potentially surrounded by spaces を含む必要があります。

URLが指すコンテンツの性質はさておき、srcはvalidでnon-emptyなURLである必要があるとのこと。空文字列である時点でnon-emptyを満たしていません。オワタ…。

### 空文字列はURLと言えるのか

non-emptyは一旦見なかったことにして、空文字列はURLと言えるのかを追っていくことにしました。先程の引用の [valid non-empty URL potentially surrounded by spaces](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#valid-non-empty-url-potentially-surrounded-by-spaces) から順番に用語定義をたどります。

すると**valid non-empty URL potentially surrounded by spaces**とは、[**valid URL string**](https://url.spec.whatwg.org/#valid-url-string)であることがわかります。

さらにリンクをたどるとHTML仕様書からURL仕様書(URL Standard)に移ります。それによれば、valid URL stringとは[relative-URL-with-fragment string](https://url.spec.whatwg.org/#relative-url-with-fragment-string) か [absolute-URL-with-fragment string](https://url.spec.whatwg.org/#absolute-url-with-fragment-string) のどちらかであるとのこと。

**absolute-URL-with-fragment string**はおそらくURLと聞いて思い浮かぶ形式のことですね。定義的には少なくとも[**URL-scheme string**](https://url.spec.whatwg.org/#url-scheme-string)とコロン(`:`)を持つ必要があるようです。`http:`とか`wss:`から始まっている文字列をabsolute-URL stringとみなすということですね。つまり空文字列はabsolute-URL stringではないことがわかりました。

では**relative-URL-with-fragment string**はどうでしょうか。relativeというからにはベースになるURL(base URL)があり、そのscheme次第で満たすべき内容が変わるようです。img要素の`src`のbase URLが、それを埋め込んでいるHTMLドキュメントのURLであることは一般的に知られているので、http or httpsのケース(file以外の[**special scheme**](https://url.spec.whatwg.org/#special-scheme)のケース)だけを考えてよいでしょう。base URLのschemeがhttp or httpsのとき、relative-URL stringが取り得る形態は次の3つです。

- a [scheme-relative-special-URL string](https://url.spec.whatwg.org/#scheme-relative-special-url-string)
- a [path-absolute-URL string](https://url.spec.whatwg.org/#path-absolute-url-string)
- a [path-relative-scheme-less-URL string](https://url.spec.whatwg.org/#path-relative-scheme-less-url-string)

[scheme-relative-special-URL string](https://url.spec.whatwg.org/#scheme-relative-special-url-string) は`//`が先頭に付く必要があるので空文字列は満たしません。[path-absolute-URL string](https://url.spec.whatwg.org/#path-absolute-url-string) は`/`が先頭に付く必要があるのでこれも空文字列は満たしません。最後の [path-relative-scheme-less-URL string](https://url.spec.whatwg.org/#path-relative-scheme-less-url-string) は**ゼロ個以上の** [URL-path-segment string](https://url.spec.whatwg.org/#url-path-segment-string) で先頭スラッシュを含まないということで、[URL-path-segment string](https://url.spec.whatwg.org/#url-path-segment-string) がゼロでも良い、つまり空文字列でもpath-relative-scheme-less-URL stringを満たします。

ということで、base URLが指定されていれば空文字列はURLと言えますね。

### base URLは本当にHTMLのURLなのか

先程は「img要素の`src`のbase URLが、それを埋め込んでいるHTMLドキュメントのURLであることは一般的に知られている」と書きましたが、それは本当なのでしょうか。それが記載されている箇所を探します。

`src`は [valid non-empty URL potentially surrounded by spaces](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#valid-non-empty-url-potentially-surrounded-by-spaces) であると定義されていました。このリンクをたどった先に、HTML標準における[**Parsing URLs**](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#resolving-urls)について記載がありました(URLのパースはURL標準の範疇ですが、HTMLではさらにbase URLとエンコーディングが絡むため追加で定義しているとのこと)。そこでは、base URL について

> Let baseURL be environment's [base URL](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#document-base-url), if environment is a [`Document`](https://html.spec.whatwg.org/multipage/dom.html#document) object; otherwise environment's [API base URL](https://html.spec.whatwg.org/multipage/webappapis.html#api-base-url).

> (訳)
>
> 環境が Document オブジェクトの場合は baseURL を環境のベース URL とし、そうでない場合は環境の API ベース URL とする。

と書かれています。HTMLページはDocumentでしょう。[Documentの base URL](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#document-base-url) には、`href`属性を持つ`base`要素がなければ(かつ普通のHTMLページなら)ドキュメントのURLが採用され、そうでなければ最初の`base`要素の`href`が採用されるとのことでした。

ということで、`base`要素が存在しなければ、`src`属性のbase URLはHTMLドキュメントのURLであることは本当だとわかりました。

また、「空文字列はURLである」と「`src`のbase URLはHTMLドキュメントのURLである」の2点から、`<img src="">` をブラウザで表示したときHTMLImageElementの`src`プロパティがHTMLドキュメントのURLを指していることがわかり、最初に提示した挙動の説明が付きましたね。

### non-emptyのこと忘れてないか？

`src`属性は [valid non-empty URL potentially surrounded by spaces](https://html.spec.whatwg.org/multipage/urls-and-fetching.html#valid-non-empty-url-potentially-surrounded-by-spaces) でなければなりません。non-emptyとあるように、そもそも空文字列は仕様違反です。しかし違反と言えどその状態にできてしまうのがHTMLです。`src`属性を空文字列にした場合どうなるかを調べます。

HTMLImageElementで画像がダウンロードされたかどうかを確認できる [**complete**](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-complete) プロパティの挙動について記載がありました。

> the [`srcset`](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-img-srcset) attribute is omitted and the [`src`](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-img-src) attribute's value is the empty string;
> then return true.

> (訳)
> `srcset`属性が省略され、`src`属性の値が空文字列の場合、`true`を返す

`complete`プロパティは`src`属性が空文字列のときは強制的に`true`になるようですね。

試すとわかりますが、`<img src="">`をHTMLに埋め込んでも、`element.src`はHTMLのURLを指すものの、実際にリクエストが飛ぶわけではありません。[**_Updating the image data_**](https://html.spec.whatwg.org/multipage/images.html#updating-the-image-data) という節を見ると、実際に画像を取得する流れが書いてありました。そこには

> 11. If selected source is null, then:
>     (略)
>
>     Return.
>
>     (略)
>
> 12. Fetch the Image.

とあり、seleceted sourceがnullならば、26. Fetch the Image に到達することなく処理を終了することがわかります。

seleceted sourceを決めるための [**_Selecting an image source_**](https://html.spec.whatwg.org/multipage/images.html#selecting-an-image-source) というフローもあり、source setがemptyのときにnullを返すそう。[**source set**](https://html.spec.whatwg.org/multipage/images.html#source-set) は画像要素が内部で持っている画像ソースのOrdered Set(コレクション)のようです。source setがemptyとなるかを確認するために [**_Updating the source set_**](https://html.spec.whatwg.org/multipage/images.html#update-the-source-set) を見てみると、

> If el is an [`img`](https://html.spec.whatwg.org/multipage/embedded-content.html#the-img-element) element that has a [`src`](https://html.spec.whatwg.org/multipage/embedded-content.html#attr-img-src) attribute, then set *default source* to that attribute's value.
> (略)
> Let el's [source set](https://html.spec.whatwg.org/multipage/images.html#source-set) be the result of [creating a source set](https://html.spec.whatwg.org/multipage/images.html#create-a-source-set) given *default source*, *srcset*, and *sizes*, and img.

> (訳)
> el が src 属性を持つ img 要素の場合、_default source_ をその属性の値に設定します。
> (略)
> el のsource set を、_default source_、_srcset_、_sizes_、_img_ が与えられた creating a source set の結果とします。

とのことで、`src`属性はdefault sourceとして扱われます。なので、default sourceに空文字列が入っている前提で [**_Creating a source set from attributes_**](https://html.spec.whatwg.org/multipage/images.html#create-a-source-set) を見てみます(`srcset`, `sizes` は考えません)。

> If default source is not the empty string and source set does not contain an [image source](https://html.spec.whatwg.org/multipage/images.html#image-source) with a [pixel density descriptor](https://html.spec.whatwg.org/multipage/images.html#pixel-density-descriptor) value of 1, and no [image source](https://html.spec.whatwg.org/multipage/images.html#image-source) with a [width descriptor](https://html.spec.whatwg.org/multipage/images.html#width-descriptor), append default source to source set.

> (訳)
> default source が空文字列ではなく、source set にピクセル密度記述子値が 1 の画像ソースがなく、幅記述子を持つ画像ソースもない場合、source set に default source を追加します。

「default source が空文字列ではなく」という条件から、生成されたsource setには空文字列のdefault sourceは渡されないことがわかりました。

つまり遡ると、img要素の`src`属性が空文字列の場合、source setはemptyになり、selected sourceはnullとなり、画像取得リクエストが送信されないことがわかりました。

## なぜこんなことを調べたのか

html-to-imageというライブラリを使ってちょっとした開発をしていました。このライブラリはブラウザ上でDOMを画像として保存する処理を提供しています。

https://github.com/bubkoo/html-to-image

このライブラリは対象のDOM内部にimg要素を見つけると、そのHTMLImageElementの`src`プロパティを取り出して、そのURLに対して`fetch`を実行します。下のGitHubコードブロックはその該当箇所です。

https://github.com/bubkoo/html-to-image/blob/128dc3edfde95d6ac636f2756630f5cbd6f7c8df/src/dataurl.ts#L15-L38

コードを見ると、エラー扱いするのは`fetch`結果が404 Not Foundのときだけです。

しかし僕のアプリケーション都合で、画像化したいDOMに`<img src="">`が含まれていました。仕様書を読み漁ってわかったように、`src`属性が空文字列のときはHTMLImageElementの`src`プロパティはそのHTMLのURLになります。つまり`fetch`が200 OKになってしまうのです。

その結果、画像データが含まれているはずのblobにhtml文字列が混入し、後続の処理の途中に予期せぬエラーで失敗していました。

エラー内容もよくわからず、調査に時間を費やしました。その調査の過程で、`src`が空文字列の場合はHTML URLを指していることに気づき、仕様を調べることにしたのです。

## まとめ

- img要素の`src`属性を空文字列にすると、そのHTMLImageElementの`src`プロパティは空文字列になる
- HTML仕様では`src`属性を空文字列にするのは仕様違反
- 空文字列自体はbase URLがあればvalid URL stringとみなせる
- html-to-image使うときは仕様違反のimg要素に気をつけて

仕様違反のコードは書かないように気をつけよう。仕様書読むのめちゃ疲れた。

それでは良いHTMLライフを！この両の手が死に届くその日まで。
