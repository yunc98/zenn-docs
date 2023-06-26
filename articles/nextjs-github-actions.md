---
title: '【Next.js × Docker × GitHub Actions × GitHub Pages】ポートフォリオサイトを公開してみた'
emoji: '⛳'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['nextjs', 'docker', 'githubpages', 'githubactions']
published: true
---

## 1. はじめに

今回、GitHub Actions を用いて、Docker で構築した Next.js のポートフォリオサイトを、GitHub Pages へデプロイしました。

その際の手順を記事に残します。

なお、今回の技術選定理由は、フロントエンドでは Next.js が一番使い慣れていたから、GitHub Pages のドメインで公開したかったからです。

## 2. ディレクトリ構成

```
portfolio/
├── .github/
│   └── workflows/
│       └── nextjs-deploy.yml
├── app/
│   ├── next.config.js
│   └── package.json
├── Dockerfile
└── compose.yml
```

## 3. Docker

以下のファイルを作成します。

```Dockerfile:Dockerfile
FROM node:lts-bullseye-slim

WORKDIR /app
```

```yml:compose.yml
version: '3'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ./app:/app
    command: sh -c "npm run dev"
    ports:
      - '3000:3000'
```

## 4. Next.js

`package.json`の`scripts`を以下の通り変更します。

```diff json:package.json
"scripts": {
    "dev": "next dev",
-   "build": "next build",
+   "build": "next build && next export",
    "start": "next start",
    "lint": "next lint"
  },
```

`next.config.js`を以下の通り変更します。
今回、`https://xxx.github.io/リポジトリ名`という URL で公開したかったので、`basePath`と`assetPrefix`にリポジトリ名を指定しています。

```js:next.config.js
/** @type {import('next').NextConfig} */
const urlPrefix = '/portfolio'; // ご自身のリポジトリ名に変更してください

const nextConfig = {
  output: 'export',
  assetPrefix: urlPrefix,
  basePath: urlPrefix,
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
};

module.exports = nextConfig;
```

:::message
`images`で`unoptimized: true`を指定しないと、うまく画像を表示させることができませんでした。Next.js の`<Image>`タグを使っていると発生する事象のようですが、詳しい理由がわかったら追記します。
:::

## 5. GitHub Actions

今回はリポジトリ内でワークフローを定義するため、以下のファイルを作成します。

```yml:.github/workflows/nextjs-deploy.yml
name: nextjs-deploy

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: 'pages'
  cancel-in-progress: true

jobs:
  build:
    name: Build
    runs-on: ubuntu-20.04

    steps:
      - uses: actions/checkout@v3

      - name: Install packages
        run: docker compose run --rm app sh -c "npm install"

      - name: Build
        run: docker compose run --rm app sh -c "npm run build"

      - name: Add nojekyll
        run: |
          cd app
          mkdir -p app/out
          touch app/out/.nojekyll

      - uses: actions/upload-pages-artifact@v1
        with:
          path: 'app/out'

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1
```

## 6. GitHub Pages

リポジトリの設定を変更します。

リポジトリの`Settings`→`Pages`→`Build and deployment`の`Source`を`GitHub Actions`に設定します。
![github pages](/images/articles/nextjs-github-actions/github-pages.png)

これで、`main`ブランチへの push 時に自動でビルド、デプロイされるようになりました。

## 7. 最後に

初めて GitHub Pages へデプロイしてみましたが、無料で手軽に利用できてよかったです。

実際のコードはこちらです。
https://github.com/yunc98/portfolio

## 8. 参考

以下の記事を参考にさせていただきました。
[https://tech-lab.sios.jp/archives/33691](https://tech-lab.sios.jp/archives/33691)
[https://maku.blog/p/5q3eq2c/](https://maku.blog/p/5q3eq2c/)
[https://r7kamura.com/articles/2023-05-27-github-pages-deploy-direct](https://r7kamura.com/articles/2023-05-27-github-pages-deploy-direct)
[https://qiita.com/SKYSK/items/d647032a3e941d2da325](https://qiita.com/SKYSK/items/d647032a3e941d2da325)
