---
title: "plan前にやるべき地味な確認、terraform fmt と validate"
emoji: "🧹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "githubactions", "初学者"]
published: true
---

おはようございます、YASUです。

GitHub Actionsの設定ファイルの中身を見た時、terraform initとterraform planだけを扱いましたが、実務ではその前段階に、コードの見た目や文法をチェックする工程が挟まることが多いようです。今日はterraform fmtとterraform validateという2つのコマンドを整理してみます。

## 結論

terraform fmtはコードの「見た目」を自動整形するコマンド、terraform validateはコードの「文法」に間違いがないかをチェックするコマンドです。どちらも実際にAWS上のリソースには一切触れない、安全な確認作業という点が共通しています。

## terraform fmt:コードの見た目を整える

```bash
terraform fmt
```

これは、インデント(字下げ)やスペースの空け方など、コードの見た目を、Terraformの標準的なスタイルに自動で揃えてくれるコマンドです。

```hcl
# 整形前
resource "aws_instance" "web" {
    ami           = "ami-12345"
        instance_type = "t2.micro"
}

# terraform fmt 実行後
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
}
```

複数人でコードを書いていると、人によってインデントの深さや揃え方がバラバラになりがちです。fmtを実行するだけで、誰が書いても同じ見た目に統一されるので、コードレビューの際に「見た目の違い」に気を取られず、「中身の変更点」に集中できるようになります。

## terraform validate:文法をチェックする

```bash
terraform validate
```

こちらは、コードの構文(文法)が正しいかどうかをチェックするコマンドです。例えば、括弧の閉じ忘れや、必須の項目が抜けている、といった単純なミスを検出してくれます。

```
Success! The configuration is valid.
```

と表示されれば問題なし、エラーがあれば具体的にどの行で何が間違っているかを教えてくれます。

## plan/applyとの違い

ここが一番のポイントです。fmtとvalidateは、AWS環境には一切アクセスしません。

- terraform fmt:ファイルの見た目を直すだけ(ネットワーク接続すら不要)
- terraform validate:文法上のエラーがないかだけをチェック(こちらもAWSには繋がない)
- terraform plan:実際にAWSの現状と比較して、差分を計算する(AWSへの接続が必要)
- terraform apply:実際にAWS上へ変更を反映する

つまりfmtとvalidateは、planより前の段階で行う、いわば「提出前の見直し」のような位置づけです。

## なぜCI/CDに組み込むと便利なのか

以前紹介したGitHub Actionsのワークフローに、planより前にこの2つを追加すると、こんな流れになります。

```yaml
steps:
  - name: リポジトリを取得
    uses: actions/checkout@v4

  - name: Terraformをセットアップ
    uses: hashicorp/setup-terraform@v3

  - name: フォーマットチェック
    run: terraform fmt -check

  - name: 文法チェック
    run: terraform validate

  - name: terraform init
    run: terraform init

  - name: terraform plan
    run: terraform plan
```

`fmt -check`は、実際にファイルを書き換えるのではなく「整形が必要な箇所がないか」だけを確認するオプションです。もしコードの見た目が乱れていたり、文法にミスがあったりすると、この段階でエラーになり、後続のplanやapplyに進む前に気づける、という仕組みです。

## 身近な例えで考えると

これは、以前紹介したGitのgit diffで最終チェックする話や、コミット前の確認習慣とも通じる考え方です。本番に近い工程(plan/apply)に進む前に、まず簡単で機械的にチェックできる部分(見た目・文法)から順番に確認していく。この「軽い確認から重い確認へ」という段階を踏む発想は、Terraformに限らず、ソフトウェア開発全般に共通する基本的な考え方のようです。

## まとめ

- terraform fmtはコードの見た目を自動整形するコマンド
- terraform validateは文法上のミスをチェックするコマンド
- どちらもAWS環境には一切アクセスしない、安全な確認作業
- CI/CDの中で、planより前の段階に組み込むことで、軽微なミスを早期に発見できる

plan/applyという実際にインフラに影響するコマンドの手前に、こうした地味だけど大事な確認工程がある、というのは新しい発見でした。それでは、また明日!
