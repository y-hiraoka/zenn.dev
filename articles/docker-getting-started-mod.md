---
title: "Docker getting-started で使用しているコンテナイメージを別のものに変更して理解を深める"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker"]
published: false
---

# 対象読者

Dockerのgetting-started

# Docker の getting-started について

Docker を学び始める人は以下のページを参考にすると良いと思います。
僕も最近 Docker の学習を始めまして、 getting-started で完全に理解しました(一通り目を通した、の意)。

## 参考ページ

- Docker 公式ページ https://github.com/docker/getting-started
- 日本語訳 https://qiita.com/Michinosuke/items/5778e0d9e9c04038903c

## Docker getting-started の内容

内容はよくある TODO アプリを下記の組み合わせで作成するチュートリアルになっています。

### バックエンド

- MySQL
- Node.js
- Express.js

JSON を受け取ったり返却する WebAPI を Express で構築しています。
UI は SPA(Single Page Application)になっていて、静的配信されています。

### フロントエンド

- React.js
- Bootstrap(react-bootstrap)

ただし React は予めプロダクションビルドされた`react.production.js`を読み込むようになっています。

# この記事の目標

Docker を用いたアプリケーション開発の理解を深めるために、getting-started と全く同じ TODO アプリを別の方法で作り直していきます。

具体的には

- JavaScript をすべて TypeScript に変更
- MySQL を MongoDB に変更
- `react.production.js`を直接入れるのではなく`create-react-app`でSPA構築

あたりを行い、より実際のアプリケーション開発に近い形で進めていきます。
