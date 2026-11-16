---
title: "TerraformのコードをGitHubにpushした。大事なtfstateは上げないようにできた"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "github", "git", "初学者"]
published: true
---

おはようございます、YASUです。

これまで練習してきたTerraformのコードを、GitHubに上げてみました。リポジトリを作って、pushして、画面で確認するところまでやっています。途中でログインのエラーが出たので、その話も書きます。今回も、コマンドの意味をAIに解説してもらいながら、1ステップずつ進めました。

## 結論

TerraformのコードをGitHubに上げることができました。.gitignoreの設定どおり、terraform.tfstateや.terraformフォルダは上がっていないことも、GitHubの画面で確認できました。pushの最初のログインは、パスワードでは通らず、ghコマンドでログインし直して解決しました。

## やったこと

1. GitHubで、空のPrivateリポジトリを作る
2. ローカルで、コミット漏れがないか確認する
3. 上げ先(GitHub)を登録する
4. pushする
5. GitHubの画面で、中身を確認する

## 1. 空のリポジトリを作る

GitHubで「New repository」を押して、名前を付け、公開設定はPrivateにしました。READMEなどのチェックは全部オフにして、空のまま作ります。チェックを入れると、ローカルの履歴とぶつかって、pushでつまずくためです。

## 2. ローカルの状態を確認する

```bash
git status
```

`nothing to commit, working tree clean`と出れば、コミット漏れはありません。

## 3. 上げ先を登録する

```bash
git remote add origin https://github.com/ユーザー名/terraform-lab.git
git remote -v
```

`git remote add`は、手元のパソコン(ローカルリポジトリ)と、GitHubなどのインターネット上の保管場所(リモートリポジトリ)を結びつける(接続する)ためのコマンドです。

originは、リモートリポジトリにつける「ニックネーム(短縮名)」です。長いURLを毎回打つ代わりに、この名前でデータを送受信できます。

`git remote -v`で、登録できたかを確認します。

## 4. pushする

```bash
git push -u origin main
```

- `git push`は、ローカルのコミットをGitHubに送るコマンド
- `-u`を付けると、次からは`git push`だけで送れる

## ログインのエラーと、その解決

最初のpushは、次のエラーで失敗しました。

```
remote: Invalid username or token. Password authentication is not supported for Git operations.
```

GitHubは、パスワードでのpushを受け付けていません。そこで、GitHub公式のコマンド`gh`を入れて、ブラウザでログインし直しました。

```bash
brew install gh
gh auth login
```

- `brew install gh`で、ghをMacに入れる
- `gh auth login`で、ブラウザに表示される1回限りのコードを入力して許可する

ログイン後にもう一度pushすると、成功しました。

## 5. GitHubの画面で確認する

ブラウザでリポジトリを開いて、次の点を確認しました。

- main.tf、modulesフォルダ、.gitignore、.terraform.lock.hclが上がっている
- terraform.tfstateと.terraformフォルダは、上がっていない

tfstateには、リソースの情報など、外に出したくない内容が入ることがあります。.gitignoreで除外しておけば、GitHubに上がらないことを、実際に画面で確認できました。

## まとめ

- 流れは、空のリポジトリ作成 → `git status` → `git remote add` → `git push` → 画面で確認
- pushはパスワードでは通らない。`gh auth login`のブラウザ認証で解決した
- .gitignoreのおかげで、tfstateは上がらない。上げる前後に、この確認をする

次は、GitHub上でコードを直して合流させる、プルリクエスト(PR)を試してみます。それでは、また明日!
