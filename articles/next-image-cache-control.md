---
title: "Next.jsの画像周りのキャッシュ戦略について調べる"
emoji: "🌆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: false
publication_name: chot
---

:::message
本記事は社内勉強会の資料をベースにしています。
:::

Next.jsの画像最適化機能であるnext/imageは画像の読み込み方法として２種類あります。

```tsx
import Image from "next/image";
import stinCatJpg from "./stin_cat.jpg";

// 画像をJavaScriptモジュールのように扱い、指定する形式
<Image src={stinCatJpg} alt="stin cat" />;

// img要素同様に文字列を指定する形式
<Image src="/stin_cat.jpg" alt="stin cat" width="300" height="300" />;
```

本記事では、画像をJavaScriptモジュールのように扱う形式を「モジュール読み込み」と呼び、文字列を指定する形式を「文字列読み込み」と呼ぶことにします。

モジュール読み込み形式だとビルド時に画像サイズを計測して自動で`width`, `height`を指定してくれます。

これらは単に`width`,`height`の指定を省略できるだけの違いに留まらず、実はキャッシュの挙動が異なります。

今回はその挙動をいろんな組み合わせで調べてみました。

Next.js独自のキャッシュの話ではなく、HTTPレベルのお話です。なので、HTTPのキャッシュ関連の予備知識から始めます。

## `ETag`について

レスポンスヘッダーのひとつです。レスポンス内容を識別する文字列が格納されています(ファイルのハッシュ値とか)。

ブラウザは同じURLに再度リクエストを送信する際、リクエストヘッダーの`If-None-Match`に`ETag`の値を転記して送信します。

サーバーはリクエストヘッダーに`If-None-Match`が含まれている場合、それに対応するリソースがあればステータスコード304 Not Modifiedをレスポンスします。304レスポンスにはボディが含まれないので、転送コストを削減できます。

ブラウザが`If-None-Match`をリクエストヘッダーに付けてリソースが更新されているかどうか確認するリクエストを特に「再検証リクエスト」と呼びます。

## `Cache-Control`について

リクエストヘッダー、レスポンスヘッダーともに用いられますが、主にレスポンスヘッダーで使われます。

HTTPレスポンスのキャッシュを細かく制御するためのヘッダーです。ディレクティブの組み合わせでキャッシュの挙動を指定します。ここで紹介するディレクティブがすべてではありません。

### 登場人物

`Cache-Control`を理解するため、以下の登場人物を押さえておきます。

- ブラウザ
- CDN
  キャッシュ済みのHTMLや画像をオリジンサーバーの代わりにレスポンスする。世界中にばらまかれたコンピューターのネットワークで、ブラウザは物理的に近いCDNと通信する。
  - Cloudflare
  - AWS CloudFront
  - Vercel
  - etc.
- オリジンサーバー
  - 計算したりDBやファイルシステムにアクセスしたりして、JSONやHTML、ときには画像を生成しレスポンスする。生成に時間がかかる上にブラウザから物理的に遠い位置にあることが多い。

### 共有キャッシュ、プライベートキャッシュ

- 共有キャッシュ
  CDNが行うキャッシュ。`Cache-Control`に`private`ディレクティブを**付けないことで**共有キャッシュ可能となる。複数のユーザーが再利用する可能性がある。
- プライベートキャッシュ
  ブラウザが行うキャッシュ。ユーザーごとに異なる。

#### 注意

共有キャッシュ可能かどうかは「`public`を付けるかどうか」ではありません。

通常、リクエストヘッダーに`Authorization`が含まれているリクエストに対するレスポンスを、CDNはキャッシュしません。しかし`Cache-Control`に`public`ディレクティブを付与することで、`Authorization`が含まれているリクエストに対するレスポンスもCDNがキャッシュするようになります。

ブラウザはベーシック認証がかかったHTMLにアクセスするとき、CSSなどの関連リソースの取得にも`Authorization`ヘッダーを付与します。そのとき関連リソースについてはキャッシュしてほしいので、`Cache-Control: public`を付与することになります。

### 再検証関連

- `must-revalidate`
  キャッシュが古い(`max-age`を越えている)場合、キャッシュを使う前に再検証リクエストを送るよう指示します。再検証リクエストの結果、304レスポンスを受信すれば引き続きキャッシュを利用可能と判断します。200レスポンスなら古いキャッシュを破棄して新たにレスポンスをキャッシュします。
- `immutable`
  キャッシュが新鮮(`max-age`を超えていない)な場合、そのリソースは更新されないことを保証します。その間、再検証リクエストは送信されません。
  ユーザーがページをリロードすると、ブラウザはHTMLのキャッシュが新鮮かどうかに関わらず再検証リクエストを飛ばします。そのとき、CSSやJSなどの関連リソースにも同様に再検証リクエストを飛ばそうとします。しかしCSSやJSは絶対に更新されないことが保証されていれば、関連リソースまで再検証リクエストすることは無駄です。`immutable`はそういった再検証リクエストが不要な静的リソースに指定することで効果を発揮します。

### 例

- `Cache-Control: max-age=86400, must-revalidate`
  ブラウザもCDNも1日キャッシュします。1日後は再検証リクエストが送信されます。ただし`Authorization`ヘッダーがついたリクエストに対するレスポンスの場合CDNはキャッシュしません。
- `Cache-Control: private, max-age=60, must-revalidate`
  ブラウザは1分間キャッシュします。1分後は再検証リクエストが送信されます。CDNはキャッシュしません。
- `Cache-Control: public, max-age=31536000, immutable`
  ブラウザもCDNも1年間キャッシュします。ブラウザリロードしても再検証リクエストは送信されません。

## next/imageによる画像の表示方法

next/imageの`src` propsを色々試し、キャッシュの挙動を確認していきます。厳密には、Next.jsの`_next/image`エンドポイントのレスポンスヘッダーを確認していくことになります。

Next.jsは(おそらく他のフレームワークも)開発ビルドとプロダクションビルドで異なるキャッシュの振る舞いをします。以下はすべてプロダクションビルドをローカルで起動した場合の挙動です。

挙動確認のために作成したリポジトリはこちら:

https://github.com/stinbox/next-image-cache-test

### publicディレクトリの画像を表示する

```tsx
<Image src="/stin_cat.jpg" alt="stin cat" width="300" height="300" />
```

URL: http://localhost:3000/\_next/image?url=%2Fstin_cat.jpg&w=384&q=75

```
HTTP/1.1 200 OK
Vary: Accept
Cache-Control: public, max-age=60, must-revalidate
ETag: +mPkCMnmURL37FHXdGx26+W3vunxSw4Ph4OpkhGy9ZY=
Content-Type: image/webp
Content-Disposition: inline; filename="stin_cat.webp"
Content-Security-Policy: script-src 'none'; frame-src 'none'; sandbox;
X-Nextjs-Cache: STALE
Content-Length: 23668
Date: Tue, 23 Jul 2024 05:08:51 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

参考までに、publicディレクトリから直接取得する場合

URL: http://localhost:3000/stin_cat.jpg

```
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: public, max-age=0
Last-Modified: Tue, 23 Jul 2024 02:33:09 GMT
ETag: W/"25b66-190dd6f7e2b"
Content-Type: image/jpeg
Content-Length: 154470
Date: Tue, 23 Jul 2024 05:08:51 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

#### 考察

publicディレクトリに配置したファイルは、`max-age=0`で配信されます。キャッシュはされるが必ず再検証リクエストが飛ぶので、ページを開くたびに304レスポンスを受信することになります。これは、ファイル名を変えずに中身を変更したらブラウザまで変更が届くようにするためと考えられます(`must-revalidate`が付いていないので、ネットワークエラーなどで再検証リクエストが失敗すると引き続きキャッシュが利用されます)。

publicディレクトリに配置した画像をnext/image経由で表示する場合、`max-age=60`が付けられます。next/imageは最小キャッシュタイムを必ず指定する仕様のようです([参考](https://nextjs.org/docs/app/api-reference/components/image#minimumcachettl))。とはいえキャッシュタイム1分だけなので、高確率で再検証リクエストが送信されることになります。

### モジュール形式で画像を表示する

```tsx
import stinCatJpg from "./stin_cat.jpg";

<Image src={stinCatJpg} alt="stin cat" />;
```

URL: http://localhost:3000/\_next/image?url=%2F_next%2Fstatic%2Fmedia%2Fstin_cat.2aa26832.jpg&w=1080&q=75

```
HTTP/1.1 200 OK
Vary: Accept
Cache-Control: public, max-age=315360000, immutable
ETag: iMBFlTZ-h-DRP1Rf3GoV5Q0cgPZua0Ryu7xOR+ySrqU=
Content-Type: image/webp
Content-Disposition: inline; filename="stin_cat.webp"
Content-Security-Policy: script-src 'none'; frame-src 'none'; sandbox;
X-Nextjs-Cache: HIT
Content-Length: 90952
Date: Tue, 23 Jul 2024 05:08:51 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

#### 考察

publicディレクトリ画像とは違い、`max-age`が10年かつ`immutable`になっています。画像がビルドに取り込まれてハッシュ値が付けられるようになるので、ファイル名を変更せずに画像の中身を変更してもビルドタイムで異なるハッシュ値が付与され、別ファイル扱いになります。それぞれの画像URL単位では二度と変更されないと見做すことができ、`max-age`を極端に大きくしたり`immutable`を付与して再検証リクエスト不要と宣言できます。

### 外部の画像を表示する

```tsx
<Image
  src="https://placehold.jp/150x150.png"
  alt="placeholder"
  width="300"
  height="300"
/>
```

URL: http://localhost:3000/\_next/image?url=https%3A%2F%2Fplacehold.jp%2F150x150.png&w=384&q=75

```
HTTP/1.1 200 OK
Vary: Accept
Cache-Control: public, max-age=31536000, must-revalidate
ETag: 6iKlZFr95Vu30RakGwAAznYVP3Fam6y8VlvF3Wao9vg=
Content-Type: image/webp
Content-Disposition: inline; filename="150x150.webp"
Content-Security-Policy: script-src 'none'; frame-src 'none'; sandbox;
X-Nextjs-Cache: HIT
Content-Length: 380
Date: Tue, 23 Jul 2024 05:08:51 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

#### 考察

`max-age`に1年指定されています、これは`placehold.jp`側の`max-age`をそのまま利用しています(`max-age`が異なる別の画像サービスでも確認済み)。`must-revalidate`はnext/imageが設定しており、`placehold.jp`側で画像が変更された場合に備えて、`max-age`を越えたら再検証リクエストを要求させたいと考えられます。

### next/ogで生成した画像を表示する

```tsx
// app/next-og-image/route.tsx
import { ImageResponse } from "next/og";

export const dynamic = "force-dynamic";

export const GET = (request: Request) => {
  const { searchParams } = new URL(request.url);
  const text = searchParams.get("text") || "HELLO WORLD";

  return new ImageResponse(<div>(省略)</div>);
};

// page.tsx
<Image
  src="/next-og-image?text=STIN_CAT NAMAIKI"
  alt="STIN_CAT NAMAIKI"
  width="1200"
  height="630"
/>;
```

URL: http://localhost:3000/\_next/image?url=%2Fnext-og-image%3Ftext%3DSTIN_CAT%20NAMAIKI&w=1200&q=75

```
HTTP/1.1 200 OK
Vary: Accept
Cache-Control: public, max-age=31536000, must-revalidate
ETag: iieAn3ECTPBsSm-2dc-MV1-jAno+3wzAn27vKsPQcSg=
Content-Type: image/webp
Content-Disposition: inline; filename="next-og-image.webp"
Content-Security-Policy: script-src 'none'; frame-src 'none'; sandbox;
X-Nextjs-Cache: MISS
Content-Length: 6096
Date: Tue, 23 Jul 2024 05:08:51 GMT
Connection: keep-alive
Keep-Alive: timeout=5
```

参考までに、next/ogを使った画像エンドポイントに直接リクエストしたら

URL: http://localhost:3000/next-og-image?text=STIN_CAT%20KAWAII

```
HTTP/1.1 200 OK
vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch
cache-control: public, immutable, no-transform, max-age=31536000
content-type: image/png
Date: Tue, 23 Jul 2024 05:08:51 GMT
Connection: keep-alive
Keep-Alive: timeout=5
Transfer-Encoding: chunked
```

#### 考察

next/ogの`Cache-Control`デフォルトは1年キャッシュの`immutable`のようです。ただ、URLにハッシュ値は付かないので、next/ogを使ったRoute Handlerのソースコードを変更してもブラウザまで変更が届かないことになります（特殊なRoute Handlerである`opengraph-image.tsx`で使うと`og:image`にはハッシュ値が付与されるため問題ない）。`opengraph-image.tsx`以外でnext/og使用するときは`ImageResponse`オプションの`headers`で`Cache-Control`を調整するのが安全です。

next/ogの画像をnext/imageに通すと`immutable`ではなく`must-revalidate`になります。これは外部リソースの取得と同様に、元の画像の変更にブラウザが追従してほしいからと考えられます。

## みんな大好きお金の話

publicディレクトリに画像を置いて使う場合、直接ファイル取得すると`max-age=0`だしnext/imageを通しても(デフォルトでは)`max-age=60`となります。これだと、ほぼページを開くたびに再検証リクエストが行われることになります。

画像も一つのJavaScriptモジュールとして読み込めば、バンドラーがいい感じにしてくれてキャッシュの効率を最大化でき、ページを何度開いても画像についてはリクエスト回数が増えません。

ところで、Vercelの新料金体系ではEdge Requestsという項目が追加されました。純粋にリクエストの回数だけお金を取るよという項目です。これはVercelに対するすべてのリクエストがカウントされ、CSS/JS/Imageなどの静的リソースも含みます。304レスポンスであっても1回は1回としてカウントされるのです。

つまり画像をモジュール読み込みしていれば、Vercelコストダウンにつながる！かも。

## まとめ

- publicディレクトリのファイルは`max-age=0`で配信されるので、再検証リクエストが頻繁に行われる
- next/imageはpublicディレクトリの画像を`max-age=60`で配信する
- next/imageはモジュール読み込み形式の画像を`max-age=315360000, immutable`で配信する
- next/imageは外部リソースの画像を`max-age={画像取得元の指定値}`と`must-revalidate`で配信する
- next/imageはnext/ogで生成した画像を`max-age=31536000, must-revalidate`で配信する
- next/ogはデフォルトで`max-age=31536000, immutable`を付与するので、`opengraph-image.tsx`以外で使う場合は要注意
- モジュール形式に寄せることで画像の再検証リクエストを削減でき、Vercelのコストダウンにつながる

それでは良いNext.jsライフを！
