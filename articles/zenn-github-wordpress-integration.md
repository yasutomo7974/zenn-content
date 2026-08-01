---
title: "ZennとGitHubを連携させて、WordPressにも自動反映させてみた"
emoji: "🔗"
type: "tech"
topics: ["zenn", "github", "wordpress", "初学者"]
published: true
---

## はじめに

TypeScriptのHello World記事を書き終えたところで、「これ、毎回ブラウザからコピペで投稿するの面倒じゃないか」と思い立った。調べてみると、ZennにはGitHubリポジトリと連携して、`git push`するだけで記事が自動公開される仕組みがあるらしい。せっかくなので、GitHub連携の設定と、その記事一覧をWordPressブログにも自動表示する設定まで、一気にやってみた。

## Zennのアカウント作成

GoogleアカウントでログインしてすぐにZennは使い始められた。ユーザー名は後から変更できないので、GitHubのユーザー名と揃えておいた。表示名や自己紹介は後からいつでも変更できるので、ここは気軽に決めて大丈夫。

## GitHubリポジトリと連携する

Zennの「GitHub連携」ページから、「リポジトリを連携する」ボタンを押すと、GitHub側の認可画面に飛ぶ。ここで選べるのは以下の2択。

- **All repositories**: 保有する全リポジトリへのアクセスを許可
- **Only select repositories**: 指定したリポジトリだけ許可

無関係なリポジトリにまでアクセス権を広げる必要はないので、`Only select repositories`を選び、Zenn専用に新規作成した`zenn-content`というリポジトリだけを指定した。

連携が完了すると、「1つのリポジトリと連携中」という表示になり、対象ブランチ(`main`)と、対象リポジトリが確認できる。

## Zenn CLIをセットアップする

ローカル側の準備は次の通り。

```bash
npm init -y
npm install zenn-cli --save-dev
npx zenn init
npx zenn new:article --slug typescript-hello-world-experience
```

`npx zenn new:article`を実行すると、`articles/`配下にMarkdownファイルの雛形が生成される。フロントマター(先頭の`---`で囲まれた部分)にタイトル・絵文字・トピック・公開設定を書き、本文をMarkdownで書いていくだけ。

プレビューもローカルで確認できる。

```bash
npx zenn preview
```

`http://localhost:8000`にアクセスすると、実際にZenn上でどう表示されるかがそのまま確認できて便利。

## 公開する

記事の中身ができたら、フロントマターの`published`を`true`に変更してコミット・プッシュするだけ。

```yaml
published: true
```

```bash
git add .
git commit -m "Add first article"
git push origin main
```

プッシュ後、Zennのダッシュボード(`/dashboard/deploys`)を見ると、デプロイ履歴に「デプロイ成功」のバッジが表示され、数分後には実際に記事が公開されていた。

## ハマったポイント: HTTPS認証

初回のプッシュで、GitHubへのHTTPS認証情報が設定されておらずエラーになった。以前ポートフォリオサイト用に作っていたSSH鍵がそのまま使えたので、remoteのURLをSSH形式に変更して解決した。

```bash
git remote set-url origin git@github.com:yasutomo7974/zenn-content.git
```

一度SSH鍵を作っておけば、別のリポジトリでもそのまま使い回せる、というのは地味に便利な発見だった。

## WordPressにZenn記事一覧を表示する

最後に、WordPress側にも新しい固定ページを作り、標準の「RSS」ブロックを設置した。ZennはユーザーごとにRSSフィードのURLが用意されている。

```
https://zenn.dev/yasutomo7974/feed
```

このURLをRSSブロックに貼るだけで、Zennに投稿した記事の一覧が自動的にWordPress側にも表示されるようになった。表示件数や、日付・抜粋の表示有無、リンクを新しいタブで開くかどうかも、ブロックの設定パネルから調整できる。

これで、今後は記事をZennに1本書けば、GitHub経由でZennに公開され、WordPress側にも自動的に反映される、という流れが完成した。

## まとめ

- ZennはGitHubリポジトリと連携すると、`git push`だけで記事の公開・更新ができる
- ローカルでのプレビュー(`npx zenn preview`)で、公開前に見た目を確認できる
- WordPressにRSSブロックを設置すれば、Zennの記事一覧を自動的に表示できる
- 一度作ったSSH鍵は、別のGitHubリポジトリでもそのまま使い回せる

技術記事を書く場所を1本化できたので、今後の運用がだいぶ楽になりそうだ。
