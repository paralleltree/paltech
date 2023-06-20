---
title: "Hugoのブログを立てました"
date: 2023-06-18T14:19:43+09:00
draft: true
summary: ''
tags: ['Hugo', 'Netlify']
---


今まで静的サイトジェネレーターの類を触ったことがなく、[さくらのナレッジ](https://knowledge.sakura.ad.jp/22908/)を参考にリポジトリを作成。
テーマはぱっと見つけたPaperModを入れてみました。

Netlifyでデプロイするまでの過程で直したりしたところの記録です。

## スタイル修正

試しに記事を書き始めてみると、文字が大きめに感じたので全体的に小さくなるようにスタイルを修正してみました。

次にコードスニペットを書いてみたところ、Windows上で見る限りはmonospaceフォントがMSゴシックにフォールバックしていました……。

[Issueも立っていた](https://github.com/adityatelange/hugo-PaperMod/issues/634)のですが、どうも再現していないみたいです。

コード書くのに残念な感じになるのは避けたいので、適切なフォントを当てるようにスタイルをオーバーライドしました。

https://github.com/paralleltree/blog.paltee.net/commit/301c7d14303801a8906491b9f01371464f1a61c2

## Analyticsの導入

テーマのテンプレートをlayoutsディレクトリにコピーして計測タグを挿入するよう対応しました。
Google Analyticsの測定IDは設定か環境変数から渡せるようにしています。
ここではNetlifyの環境変数から渡すように設定しました。

https://github.com/paralleltree/blog.paltee.net/commit/64d2c9e233b83c53b8c758101bfa12b2a4e54931


## ページ構成とPage Bundles

ページ管理はある程度のまとまりで分けたいと考え、年ごとにディレクトリを切ってみることにしました。

```
content/
└── posts
    └── 2023
        └── setup-hugo-blog
            └── index.md
```

ただURLはタイトルがあれば良いかなとも思ったので、permalinksではfilenameのみになるように設定しました。

```
permalinks:
  posts: "/posts/:filename/"
```

今のHugoは[Page Bundles](https://gohugo.io/content-management/page-bundles/)のおかげで画像が管理しやすくなっているみたいですね。

permalinksにfilenameを指定していたので、URLはどうなるんだろうと思ったらちゃんと`index.md`のディレクトリ名になってました。

https://github.com/gohugoio/hugo/blob/0e7944658660b5658b7640dce3cb346d7198d8c9/resources/page/permalinks.go#L251-L255

## デプロイ

まずはVercelを使ってデプロイを試してみましたが、submoduleの解決がうまくいかないようでビルドできなかったのでNetlifyを使うことにしました。

リポジトリを設定してデプロイしたところ、なぜかhugoのサイトとして構成されていないと怒られました。

```plain
8:04:24 PM: Failed during stage 'building site': Build script returned non-zero exit code: 2 (https://ntl.fyi/exit-code-2)
8:04:22 PM: Netlify Build
8:04:22 PM: ────────────────────────────────────────────────────────────────
8:04:22 PM: ​
8:04:22 PM: ❯ Version
8:04:22 PM:   @netlify/build 29.12.1
8:04:22 PM: ​
8:04:22 PM: ❯ Current directory
8:04:22 PM:   /opt/build/repo
8:04:22 PM: ​
...
8:04:22 PM: Build command from Netlify app
8:04:22 PM: ────────────────────────────────────────────────────────────────
8:04:22 PM: ​
8:04:22 PM: $ hugo
8:04:22 PM: Error: Unable to locate config file or config directory. Perhaps you need to create a new site.
8:04:22 PM:        Run `hugo help new` for details.
8:04:22 PM: Total in 0 ms
```

確認したところhugoのバージョンが古かったようで、環境変数から`HUGO_VERSION=0.113.0`を指定して解決しました。

その後はドメインの設定もすぐにでき、バージョン指定を除いて確かに良い体験だなあと感じました。

## おわりに

やろうやろうと思ってやっていなかった静的サイトジェネレーターによるブログを作りました。
過去にやったことも備忘として記録していこうと思います。
