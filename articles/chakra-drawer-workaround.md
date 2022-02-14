---
title: "Chakra UI の全画面 Drawer をスマホで開くとコンポーネントの一部が隠れる問題の回避策"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "chakraui"]
published: false
---

# Drawer の size を "full" にする場合は要注意

Chakra UI の Drawer には `size` と `placement` が props として渡せます。
`size` はその名の通りドロワーの大きさを指定でき、 `size="full"` とすれば全画面ドロワーにできます。
`placement` はドロワーがどの方向からスライドインしてくるかを指定でき、 `"top" | "right" | "bottom | "left" | "start" | "end"` を選択できます。

## 問題点

下のサンプルコードのように、 Drawer に `size="full"` かつ `placement="bottom"` を渡してみます。

```tsx
const App: VFC = () => {
  const { isOpen, onOpen, onClose } = useDisclosure();

  return (
    <Box p="8">
      <Button colorScheme="teal" onClick={onOpen}>
        Open
      </Button>
      <Drawer isOpen={isOpen} size="full" placement="bottom" onClose={onClose}>
        <DrawerOverlay />
        <DrawerContent>
          <DrawerCloseButton />
          <DrawerHeader>Create your account</DrawerHeader>
          {/* 省略 */}
        </DrawerContent>
      </Drawer>
    </Box>
  );
};
```

@[codesandbox](https://codesandbox.io/embed/my-sandbox-lm4kf?fontsize=14&hidenavigation=1&theme=dark)

https://lm4kf.csb.app/

この Codesandbox のアプリをスマートフォンで開いて操作してみた動画がこちらです。
![](https://storage.googleapis.com/zenn-user-upload/6c45f1be387e-20220214.gif)
_Google Pixel 6 pro の Chrome を撮影_

ドロワーのタイトルとクローズボタンが URL バーに隠れてしまいます("Open Sandbox" のボタンに隠れるのは関係ありません)。

これはドロワーの実体である `div` 要素に対して `height: 100vh` が付与されているのが原因と考えられます。モバイルブラウザの場合、 `100vh` は URL バーの高さを含むため、ドロワーの大きさがウインドウを飛び出してしまいます。

上のサンプルでは `placement="bottom"` なのでドロワー上部が隠れていますが、 それ以外の値を `placement` に指定すると共通してドロワー下部が隠れるようになります。

## 回避策

状況が少し違いますが、下記の issues がヒントになりました。

https://github.com/chakra-ui/chakra-ui/issues/1955

ウインドウの高さと同じ値を `<DrawerContent />` の `max-height` に指定することで解決しました。

ウインドウの高さはどうやって取るんやということですが、 react-use に `useWindowSize` があるのでこれが使えます。

```tsx
import { useWindowSize } from "react-use";

const App: VFC = () => {
  const { isOpen, onOpen, onClose } = useDisclosure();
  const { height } = useWindowSize();

  return (
    <Box p="8">
      <Button colorScheme="teal" onClick={onOpen}>
        Open
      </Button>
      <Drawer isOpen={isOpen} size="full" placement="bottom" onClose={onClose}>
        <DrawerOverlay />
        <DrawerContent maxH={`${height}px`}>
          <DrawerCloseButton />
          <DrawerHeader>Create your account</DrawerHeader>
          {/* 省略 */}
        </DrawerContent>
      </Drawer>
    </Box>
  );
};
```

@[codesandbox](https://codesandbox.io/embed/zenn-chakra-drawer-workaround-solved-0xmg8?fontsize=14&hidenavigation=1&theme=dark)

https://0xmg8.csb.app/

![](https://storage.googleapis.com/zenn-user-upload/c6174e2e90ad-20220214.gif)

画面サイズぴったりのドロワーになりました。

## まとめ

Chakra UI の全画面 Drawer をスマホで開くとコンポーネントの一部が隠れる問題の回避策を紹介しました。

ドキュメントを読みつつ `size="full"` を指定してみたのにうまくできなくて困ったので書いてみました。しかも PC ブラウザのスマホサイズ再現だと 100vh は正常に表示されてしまうので、不具合をスルーしてリリースしてしまうこともありそうです…(やりました)。

この記事が誰かの手助けになれば幸いです。

それではよい Chakra UI ライフを！