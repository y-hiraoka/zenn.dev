---
title: "【Prisma】findFirst をユニーク検索に使うと危ない"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma"]
published: true
publication_name: chot
---

TypeScript の ORM である Prisma の話。

Primary Key 制約や Unique 制約のついたカラムを WHERE 句に指定してデータを取得する場合、通常は `findUnique` を使います。

```ts
await prisma.user.findUnique({ where: { id: userId } });
```

`findFirst` というメソッドも存在し、まったく同じように書けてまったく同じ値が取得できます。

```ts
await prisma.user.findFirst({ where: { id: userId } });
```

しかし、この `findFirst` が恐ろしい不具合を生みかねないので、ちゃんと `findUnique` を使いましょう、という話です。

## 起きること

`findFirst`/`findFirstOrThrow`/`findMany` の `where` 句に `undefined` を指定すると、そのカラムに対するフィルタリングを指定しないことを意味します。

次のようなクエリを書いているとします。

```ts
await prisma.user.findFirst({ where: { id: jwtPayload.userId } });
```

仕様変更で `jwtPayload` の型を変更してユーザー ID の取り方が `jwtPayload.user.id` になったら、上記のクエリ例は次のようになります。

```ts
await prisma.user.findFirst({ where: { id: undefined } });
```

これは次の SQL を生成します。

```sql
SELECT `main`.`User`.`id`, `main`.`User`.`email`, `main`.`User`.`name`, `main`.`User`.`banned`
FROM `main`.`User`
WHERE 1=1
LIMIT ? OFFSET ?
```

WHERE 句には何も条件が指定されていません。つまり、無条件に検索したユーザーの中から先頭の1人を取得することになります。サービスにログインしたら全然違う人として認証されてしまう…なんてことが起こり得ます。

そんなの型エラーで検出できるだろと思われるかもしれません。しかし、JWT 周りのコードは verify 後のペイロードの型を `as` アサーションでごまかしていたり、Express.js 向けの認証ライブラリが `any` を使っていたりと、型エラーで検出不可能な場合が多いです。

実データベースを動かすテストコードを書いていたとしても、不具合を検出できない可能性が高いです。なぜなら、テストデータとしてユーザーを1件だけ登録していることが多いので、「無条件に検索したユーザーの中の先頭」はそのテストユーザーになるからです。

## 解決策

Primary Key 制約か Unique 制約のついたカラムで一意に特定されたデータを取得するには必ず `findUnique` を使いましょう。

```ts
await prisma.user.findUnique({ where: { id: jwtPayload.userId } });
```

`findUnique` はユニークに絞り込み可能なカラムを `where` 句に指定させることを、型レベルだけでなくランタイムでもチェックしています(後述の `strictUndefinedChecks` が無効でも)。

仮に次のような `where` 句の `id` に `undefined` を指定したコードを実行したとしても、ランタイムエラーとなり不具合の早期検出が可能です。

```ts
await prisma.user.findUnique({ where: { id: undefined } });
```

```plaintext: エラーログ
Argument `where` of type UserWhereUniqueInput needs at least one of `id` or `email` arguments. Available options are marked with ?.
PrismaClientValidationError:
Invalid `prisma.user.findUnique()` invocation in
/Users/stin/prisma-find-first/src/index.ts:10:35
```

## 何故 `findFirst` が使われていたか

なぜ一意なデータを取得するときにお誂え向きな `findUnique` ではなく、`findFirst` を使っていたのでしょうか？ここには Prisma ユーザーのスキルレベルに留まらない原因があると考えます。

Prisma v4 以前では、`findUnique` の `where` 句にユニークになるカラム以外を指定できませんでした(v4.5 から feature preview `extendedWhereUnique` として追加、v5 でデフォルトになった)。

https://www.prisma.io/docs/orm/reference/prisma-client-reference#filter-on-non-unique-fields-with-userwhereuniqueinput

つまり、次のようなテーブル構造のときに、

```prisma
model User {
  id     Int     @id @default(autoincrement())
  email  String  @unique
  name   String?
  banned Boolean @default(false)
}
```

次のようなコードは v4 で型エラーもランタイムエラーも起こります。

```ts
await prisma.user.findUnique({
  where: {
    id: userId,
    banned: false,
  },
});
```

なので v4 以前の Prisma では、ユニークな検索条件プラスアルファの条件でクエリを書くには `findFirst` を使わざるを得ませんでした。

```ts
await prisma.user.findFirst({
  where: {
    id: userId,
    banned: false,
  },
});
```

なので、長いこと Prisma を使っているプロジェクトでは、ユニーク検索なのに `findFirst` を使っているコードが残っているかもしれません。リスクを孕んでいるので見直すのが良いでしょう。

## v6 の現在でもユニーク検索に `findFirst` を使わざるを得ないケース

Prisma の `findUnique` には、そのテーブル内のカラムで一意に特定できる組み合わせしか指定できません。

次のような `User` と `Organization` の多対多のリレーションを考えてみます。

```prisma
model User {
  id                   Int                    @id @default(autoincrement())
  email                String                 @unique
  name                 String?
  banned               Boolean                @default(false)
  UserOrganizationRole UserOrganizationRole[]
}

model Organization {
  id                   Int                    @id @default(autoincrement())
  name                 String
  UserOrganizationRole UserOrganizationRole[]
}

model UserOrganizationRole {
  id             Int          @id @default(autoincrement())
  userId         Int
  user           User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  organizationId Int
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  role           String

  @@unique([userId, organizationId])
}
```

`UserOrganizationRole` テーブルを `organizationId` と `userId` で検索して特定のユーザーの `role` を知るには、次のように書くことができますね。

```ts
await prisma.userOrganizationRole.findUnique({
  where: {
    userId_organizationId: {
      organizationId: organizationId,
      userId: userId,
    },
  },
});
```

ところで、`User` の `email` カラムには `@unique` が付いているので、`User` は `email` によっても一意に絞り込めます。つまり、物理的には `UserOrganizationRole` テーブルを `(organizationId, email)` の組み合わせで一意に絞り込めるはずです。

しかしそうは問屋が卸しません。`findUnique` の `where` 句にはテーブルを跨いだユニークなカラム指定ができないようになっています。なのでこのケースでは `findFirst` を使う必要があります。

```ts
await prisma.userOrganizationRole.findFirst({
  where: {
    organizationId: organizationId,
    user: { email: userEmail },
  },
});
```

または、クエリを2回に分けることを受け入れられるならば、`findUnique` を使うこともできます。

```ts
const user = await prisma.user.findUniqueOrThrow({
  where: { email: userEmail },
});

await prisma.userOrganizationRole.findUnique({
  where: {
    userId_organizationId: {
      organizationId: organizationId,
      userId: user.id,
    },
  },
});
```

### より安全にするために

次の `findFirst` の例で、周辺コードを修正していたらいつの間にか `organizationId` が `string | undefined` 型になってしまったとします。

```ts
await prisma.userOrganizationRole.findFirst({
  where: {
    organizationId: organizationId, // organizationId: string | undefined
    user: { email: userEmail },
  },
});
```

`findFirst` は元々 `where` 句の任意のカラムがオプショナルになっているわけですから、型エラーにはなりません。

こういった意図しない `undefined` の混入を防ぐために、Prisma の preview features のひとつに `strictUndefinedChecks` があります。

`strictUndefinedChecks` を有効にすると、Prisma の各種メソッドに明示的な `undefined` を渡すとランタイムエラーを起こすようになります。リテラルで指定した `undefined` だけでなく、変数がたまたま `undefined` になった場合も防げます。

```ts
let organizationId: string | undefined = undefined;

// ランタイムエラー！
await prisma.userOrganizationRole.findFirst({
  where: {
    organizationId: organizationId,
    user: { email: userEmail },
  },
});
```

```plaintext: エラーログ
PrismaClientValidationError:
Invalid `prisma.userOrganizationRole.findFirst()` invocation in
/Users/stin/prisma-find-first/src/index.ts:12:51

   9 let userEmail: string = "user@example.com";
  10
  11 async function main() {
→ 12   const found = await prisma.userOrganizationRole.findFirst({
         where: {
           organizationId: undefined,
                           ~~~~~~~~~
           user: {
             email: "user@example.com"
           }
         }
       })

Invalid value for argument `where`: explicitly `undefined` values are not allowed.
```

条件によって値を指定したり指定しなかったりするクエリの場合は、`Prisma.skip` を `undefined` の代わりに指定してやります。次は記事検索を想定したクエリ例です。

```ts
await prisma.blogPost.findMany({
  where: { categoryId: params.categoryId ?? Prisma.skip },
});
```

このように `strictUndefinedChecks` を有効にすれば、不意の `undefined` 指定による検索フィルタ漏れで情報漏洩を引き起こすような事故を回避できます。

ただし `Prisma.skip` は `strictUndefinedChecks` を有効にしないと使えないので、少しずつ移行を進めることができません。一度有効化したら、すべての Prisma クエリをチェックして `undefined` が渡る可能性のある箇所を一気に `Prisma.skip` に置き換える必要があります。

Prisma は `strictUndefinedChecks` の有効化と同時に `tsconfig.json` の `exactOptionalPropertyTypes` を有効化することを推奨しています。これによって、オプショナルプロパティを持つ型の変数に対して、`undefined` の代入を許可しなくなります。以下は `propOptional` に `undefined` を代入しようとして型エラーになる例です。

```ts
type HasOptional = {
  propUndefined?: string | undefined;
  propOptional?: string;
};

const value: HasOptional = {
  propUndefined: undefined, // OK
  propOptional: undefined, // ⚠️ Type 'undefined' is not assignable to type 'string'.
};
```

Prisma の各種メソッドの引数はオプショナルで型定義されているため、`tsconfig.json` の `exactOptionalPropertyTypes` の有効化によって型レベルで `undefined` の混入を防ぎます。

```ts
await prisma.userOrganizationRole.findFirst({
  where: {
    organizationId: organizationId, // ⚠️ Type 'undefined' is not assignable to type 'number | Skip | IntFilter<"UserOrganizationRole">'.
    user: { email: userEmail },
  },
});
```

`exactOptionalPropertyTypes` のほうは工夫次第で徐々に有効化できますが、Prisma 以外のコードベースにも影響が及ぶので、移行完了まで長い作業になるでしょう。

早く有効化したい（希望的観測）

## まとめ

Prisma の `findFirst` をユニーク検索で用いると問題になるという話をしました。

ユニークに絞り込みたいのであれば専用の `findUnique` を使用しましょう。

Prisma v4 以前から使っているプロジェクトでは、ワケあってユニーク検索に `findFirst` を使っているコードが残っているかもしれません。見直すと良いでしょう。

Prisma をこれから導入するか使い始めたばかりなら、すぐに `strictUndefinedChecks` を有効化するのをおすすめします。ずっと無効のまま Prisma 使っているならば、いっしょに頑張りましょう（？）

それでは良い Prisma ライフを！
