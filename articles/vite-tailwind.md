---
title: "Vite + Tailwind CSSでペライチHTMLをマークアップする"
emoji: "️🖊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vite", "tailwind", "tailwindcss"]
published: true
---

## はじめに

つい最近、ReactもVueも使わずに静的なペライチHTMLのマークアップを行う必要に迫られました。その際ViteのVanilla[^1]テンプレートとTailwindを使った環境でマークアップをしたところ非常に快適だったので、環境構築の手順を記事にしておこうと思いました。

[^1]: ReactやVueといったライブラリやフレームワークを使わない状態のことをVanillaと呼んでいます

### Viteを使う理由

ViteというとVueやReactの開発ツールと思われがちですが、UIライブラリに依存しないVanillaテンプレートが用意されています。
ファイルの更新を検知してブラウザへの反映といった基本的な機能に加えて、PostCSSのサポートもあるのでTailwindでの開発と相性が良いです。

## セットアップ

Viteでプロジェクトを立ち上げる普段の手順に加えて、Tailwindのセットアップを行います。

```sh
npm create vite@latest
# project名を入力後、フレームワークでは「vanilla」を選択
# vanillaでもvanilla-tsのどちらを選んでも問題ありません
cd project-name
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init
# 実行後、tailwind.config.jsが生成されます。
```

`postcss.config.js`を作成し、以下の内容に変更します。[^2]

[^2]: PostCSSを使ってTailwindの変換を行う一般的な構成です。

```js:postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

生成された`tailwind.config.js`のcontentに`index.html`を指定します。

```js:tailwind.config.js
module.exports = {
  content: ["index.html"],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

`main.css`を作成しTailwindのディレクティブを追加します。

```css:main.css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

`index.html`で`main.css`を読み込みます。

```diff:index.html
  <!DOCTYPE html>
  <html lang="en">
    <head>
      <meta charset="UTF-8" />
      <link rel="icon" type="image/svg+xml" href="favicon.svg" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      <title>Vite App</title>
+     <link rel="stylesheet" href="/main.css" />
    </head>
    <body>
      <div id="app"></div>
      <script type="module" src="/main.js"></script>
    </body>
  </html>
```

ここまでの手順でセットアップは完了です。もともとのテンプレートにある`main.js`は記述を消すなり、都合のいい感じに書き換えるなりしておくといいでしょう。

以降`npm run dev`で開発環境を立てて`index.html`でTailwind CSSが使えます。

:::message
複数のHTMLを扱う際は[ViteドキュメントのMulti-Page App](https://vitejs.dev/guide/build.html#multi-page-app)を参考にしてください。
`tailwind.config.js`に追加したファイルのパスを加えることを忘れないでください。
:::

## ビルド

`npm run build`するとデフォルトで`dist`ディレクトリにHTML、CSS、JSが生成されます。
HTMLは静的ファイルですし、CSSもTailwind CSSで使っているスタイルのみ含まれたファイルになっています。

## おわりに

令和になってペライチでHTMLを書くという行為をするケースはほとんどない[^3]と思いますが、今風の開発ツールでも快適に書けるようになっています。
自分はこの環境の上に[markuplint](https://markuplint.dev/)を使って令和時代に求められる水準でマークアップを進めました。

DX(Developer Experience）の高い環境でやっていきたいところです。

[^3]: ノーコードツールを使えるなら使ったほうがいいです。

