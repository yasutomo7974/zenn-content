---
title: "GitHub Actions、難しそうで実はシンプルだった"
emoji: "⚙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

先日、TerraformとCI/CDを組み合わせる「考え方」について書きましたが、今日はもう一歩踏み込んで、実際にその仕組みを実現しているGitHub Actionsの設定ファイルの中身を見てみます。

## 結論

GitHub Actionsは、リポジトリの中に置いた1つのYAMLファイルに「いつ」「何を」実行するかを書くだけで、PR作成時に自動でterraform planを実行する、という仕組みを実現できます。難しそうに見えて、構造自体はシンプルでした。

## 設定ファイルの置き場所

GitHub Actionsの設定は、リポジトリの中の決まった場所に置きます。

```
.github/
  workflows/
    terraform-plan.yml
```

この`.github/workflows/`フォルダの中にYAMLファイルを置くだけで、GitHubが自動的にその内容を認識し、指定したタイミングで処理を実行してくれます。

## 設定ファイルの中身を見てみる

```yaml
name: Terraform Plan

on:
  pull_request:
    branches:
      - main

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - name: リポジトリを取得
        uses: actions/checkout@v4

      - name: Terraformをセットアップ
        uses: hashicorp/setup-terraform@v3

      - name: terraform init
        run: terraform init

      - name: terraform plan
        run: terraform plan
```

## 1行ずつ、意味を分解する

- `name: Terraform Plan`:このワークフロー全体の名前。GitHub上の画面で分かりやすく表示するためのラベル
- `on: pull_request` / `branches: main`:「いつ実行するか」の条件。今回は「mainブランチに対してPRが作成された時」に実行される
- `jobs: plan`:実行する作業のまとまりに、`plan`という名前を付けている
- `runs-on: ubuntu-latest`:どんな環境(仮想マシン)で実行するか。GitHubが用意しているLinux環境を借りて実行する
- `steps`:実際に実行する手順を、上から順番に並べたもの

## stepsの中身、1つずつ見ていく

**actions/checkout@v4**:まず、リポジトリの中身(Terraformのコード)を、この実行環境にダウンロードしてくる手順です。これがないと、そもそも手元にコードがない状態になってしまいます。

**hashicorp/setup-terraform@v3**:Terraform本体を、この実行環境にインストールする手順です。個人のパソコンで`terraform`コマンドが使えるように、この仮想環境の中にも一時的にTerraformをセットアップしています。

**terraform init / terraform plan**:ここでようやく、いつも自分のパソコンで打っていたコマンドが登場します。仕組みとしては、自分のパソコンで手動でやっていたことを、そのままGitHub側の仮想環境で自動的にやってもらっている、というだけのことでした。

## 「難しそう」に感じていたものの正体

CI/CDやGitHub Actionsと聞くと、何か特別な魔法のような仕組みを想像してしまいますが、中身を見てみると、やっていることは驚くほどシンプルです。

```
1. コードを持ってくる(checkout)
2. 必要なツールをインストールする(setup-terraform)
3. いつものコマンドを実行する(terraform init / plan)
```

普段手元でやっている作業の手順を、そのままYAMLという設定ファイルに書き写して、それをGitHub側で自動的に実行してもらっている、というだけの話でした。

## 実際の運用で、これがどう活きるか

このワークフローが設定されていると、誰かがTerraformのコードを変更してPRを作成した瞬間、自動的にterraform planが走ります。その結果はPRの画面上に表示されるので、レビュアーはコードを読むだけでなく、実際の差分(何が追加・変更・削除されるか)を確認しながらレビューできます。以前紹介した通り、これが「個人の注意力ではなく、仕組みで事故を防ぐ」という考え方の具体的な実現方法でした。

## まとめ

- GitHub Actionsの設定は`.github/workflows/`に置いたYAMLファイル1つで完結する
- 「いつ実行するか(on)」「何を実行するか(steps)」を順番に書くだけのシンプルな構造
- 中身は、普段手元でやっているコマンドを、そのままGitHub側の仮想環境で自動実行しているだけ
- PR作成をきっかけに自動でplanが走ることで、レビューの質と安全性が高まる

「CI/CD」という言葉の響きに身構えていましたが、実際の中身を見てみると、これまで学んできたTerraformコマンドの延長線上にあるだけだと分かり、一気に親近感が湧きました。それでは、また明日!
