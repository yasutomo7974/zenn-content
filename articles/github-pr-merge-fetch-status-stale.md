---
title: "GitHubでPRを作ってマージした。ポートフォリオを更新しようとしたら手元が古かった"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git", "github", "初学者"]
published: true
---

おはようございます、YASUです。

GitHubで、プルリクエスト(PR)を作ってマージする練習をしました。そのあと、自分のポートフォリオサイトを更新しようとしたら、手元のファイルがGitHubより古いことに気づきました。今回は、この2つを1本にまとめます。コマンドの意味は、AIに解説してもらいながら、1ステップずつ進めました。

## 結論

変更は、枝(ブランチ)を作ってPRでmainに入れるのが基本です。そして、作業を始める前と公開する前には、`git fetch`と`git status`で、手元がGitHubの最新かどうかを確認するのが大事だと分かりました。

## 1. PRでREADMEを追加した

PRは、「この変更をmainに入れてもいいですか?」と、GitHub上で確認をお願いする仕組みです。実務では、mainに直接書かず、PRを通すのが基本です。

流れは次のとおりです。

1. `git checkout -b add-readme`で、作業用の枝を作る
2. READMEを作って、`git add`と`git commit`でコミットする
3. `git push -u origin add-readme`で、枝をGitHubに送る
4. GitHubの画面で「Compare & pull request」を押して、PRを作る
5. 画面に`No conflicts with base branch`(ぶつかる部分なし)と出ているのを確認して、「Merge pull request」を押す
6. 役目を終えた枝を、「Delete branch」で消す
7. `git checkout main`と`git pull`で、手元を最新にする

マージのあとは、画面にMergedと出て、手元にもREADME.mdが増えました。

## 2. ポートフォリオを更新しようとしたら、手元が古かった

私のポートフォリオサイトは、GitHub ActionsとOIDCを使って、mainにpushすると、S3とCloudFrontに自動でデプロイされる仕組みにしてあります。資格(AIFとDVA)を追加するために、手元のファイルを直しました。

ところが、手元でページを開くと、実績のカードが1つ足りませんでした。そこで、次を打ちました。

```bash
git fetch
git status
```

- `git fetch`は、GitHubの最新情報を取ってくるコマンド。手元のファイルは変わらない
- `git status`は、今の状態を見るコマンド

すると、次のように出ました。

```
Your branch is behind 'origin/main' by 9 commits
```

手元のフォルダが、GitHubより9コミット分、古いままでした。この状態でpushすると、古いファイルで公開サイトが上書きされてしまいます。

## 最新に合わせて、直しを入れ直した

念のため、最初にフォルダ全体をコピーして、バックアップを取りました。そのあと、次の順番で進めました。

1. `git stash`で、手元の変更を、いったんしまう
2. `git pull`で、GitHubの最新を取り込む
3. 今回直したい2か所(資格のカードとAbout Meの文章)を、新しいファイルに入れ直す
4. ブラウザで見た目を確認する
5. `git add`、`git commit`、`git push`で公開する

`git stash`は、変更中のファイルを「しまっておく」コマンドです。しまうと、ファイルは最後のコミットの状態に戻りますが、変更は消えません。

## pushしたら、自動で公開された

`git push`をすると、GitHubのActionsタブに、デプロイの実行が1つ出ました。緑のチェックが付いて、約15秒で終わりました。サイトを再読み込みすると、資格の表示が更新されていました。

## まとめ

- PRは、mainに入れる前の確認依頼。枝を作る → push → PR → マージ → pullの流れ
- 作業の前や公開の前に、`git fetch`と`git status`で、手元が最新か確認する
- 手元が古いときは、stashで変更をしまって、pullで最新にしてから、直しを入れ直す
- 自動デプロイがあるサイトでは、pushがそのまま公開になる。だから、push前の確認が大事

コマンドを覚えるより、「今、手元はGitHubの最新か」を確かめる習慣がつきました。それでは、また明日!
