---
title: "Next.jsのサイトをAzure Static Web Apps にデプロイする"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

公式ガイドは以下です。

[静的にレンダリングされた Next.js の Web サイトを Azure Static Web Apps にデプロイする](https://docs.microsoft.com/ja-jp/azure/static-web-apps/deploy-nextjs)

## ローカルの適当なディレクトリに Next.js のプロジェクトを作る

```console
$ npx create-next-app@latest --ts
✔ What is your project named? … next-blog-example

# 略
  npm run dev
    Starts the development server.

  npm run build
    Builds the app for production.

  npm start
    Runs the built app in production mode.

We suggest that you begin by typing:

  cd next-blog-example
  npm run dev
```

## GitHub に Next.js のプロジェクトを登録する

GitHub で空のリポジトリを作成し、ローカルの Git リポジトリを登録します。

```console
$ git remote add origin git@github.com:username/repository-name.git
$ git push -u origin main
```

## Azure Portal で Static Web Apps を作成する

Azure Portal の検索窓に `static` と入力すると「静的 Web アプリ」が選択できます。

左上の「作成」をクリックします。

「静的 Web アプリの作成」が表示されるので、必要な情報を入力します。

以下は自分の入力例です。

- サブスクリプション - 個人開発
- リソースグループ - ブログサイト
- 静的 Web アプリの詳細 - next-azure-sample
- ホスティングプラン - Free
- Azure Functions とステージングの詳細 - East Asia
- デプロイの詳細 - GitHub

「GitHub アカウントでサインイン」というボタンがあるので、クリックします。
Azure Static Web Apps に GitHub へのアクセス権限を与えます。

GitHub から情報を取得すると、

- 組織
- リポジトリ
- 分岐（ブランチ）

が選択できます。ここで Next プロジェクトの GitHub リポジトリを選択します。

「ビルドの詳細」では以下を選択します。

- ビルドのプリセット - Custom
- アプリの場所 - `/`
- API の場所 - 空のまま
- 出力先 - `out`

次に「確認および作成」をクリックします。

詳細確認画面が開きますので、そのまま「作成」をクリックします。

すると、「デプロイが完了しました」と表示されます。

GitHub を開き、 Actions を見てみると、「Build and Deploy Job」が走っていることがわかります。

## `The app build failed to produce artifact folder: 'out'. Please ensure this property is configured correctly in your workflow file`

Azure Static Web Apps で自動的に生成される GitHub Actions を実行してもエラーが出ます。

```
The app build failed to produce artifact folder: 'out'. Please ensure this property is configured correctly in your workflow file
```

`package.json` に `export` コマンドを追加します。

```json:package.json
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "export": "next export",
    "lint": "next lint"
  },
```

`.github/workflows/azure-xxxx.yml` に以下のようなコマンドを追加します。

```yaml
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true
      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: 16.x

      - name: Install NPM packages
        run: npm i

      - name: Build Next.js app
        run: npm run build && npm run export

      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
      // 略
```

これで `git push origin main` したらデプロイできるようになりました。

[Next.js アプリを GitHub Actions でビルドして GitHub Pages で公開する](https://maku.blog/p/au8ju6g/)

`next/image` を使っていると `next export` でエラーが発生するようなので、適宜 `<img>` に置き換えるなどの修正が必要です。

[Next.js11 で静的 HTML を生成しようとすると next/image や <img> タグに関するエラーが発生した](https://qiita.com/toshikisugiyama/items/9d9ada2de0cedb03a21e)
