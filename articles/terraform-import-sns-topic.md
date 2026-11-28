---
title: "コンソールで作ったSNSトピックを、terraform importで、Terraformの管理に入れてみた"
emoji: "📥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

今日は、Terraformの import を、実際に試しました。画面で手作りしたSNSトピックを、Terraformの管理に取り込む、という練習です。AIに1つずつ手順を教えてもらいながら、進めました。

## 結論

terraform import を使うと、画面で手作りしたリソースを、Terraformの管理下に取り込めることがわかりました。

取り込む前と後で、terraform plan の結果が、はっきり変わりました。

- 取り込む前: `Plan: 1 to add`(Terraformは「まだ存在しない」と思っている)
- 取り込んだ後: `No changes.`(コードと実際が、一致している)

## そもそも、importとは

Terraformは、自分で作ったものは、記録(tfstate)に残して、管理します。でも、画面で手作りしたものは、その記録に載っていません。つまり、Terraformは、その存在を知りません。

そこで使うのが import です。手作りしたリソースを、Terraformの記録に、取り込みます。

## 手順1: 画面で、SNSトピックを作る

AWSコンソールで、SNSのトピックを1つ作りました。タイプはスタンダードで、名前は `import-test-topic` にしました。他は、初期設定のままです。

## 手順2: Terraformのコードを書く

新しいフォルダを作って、main.tf に、次のコードを書きました。

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

resource "aws_sns_topic" "imported" {
  name = "import-test-topic"
}
```

最後の resource の部分が、「SNSトピックが1つある」という注文書です。名前は、画面で作ったものと、同じにしました。

そのあと、初期化しました。

```bash
terraform init
```

## 手順3: 取り込む前に、planを見る

importの前に、planを実行しました。

```bash
terraform plan
```

結果は、`Plan: 1 to add, 0 to change, 0 to destroy.` でした。

画面では、すでに作ってあります。それでも、Terraformは「新しく作る」と言っています。これが、「手作りしたものを、Terraformが知らない」状態です。

## 手順4: importで取り込む

AWSコンソールで、トピックのARN(リソースの住所のようなもの)をコピーしました。そして、次のコマンドを実行しました。

```bash
terraform import aws_sns_topic.imported <トピックのARN>
```

- `aws_sns_topic.imported`: コードの中の、どのリソースに取り込むか
- 最後のARN: AWSの、どのリソースを取り込むか

意味は、「このARNのトピックを、コードの imported に、紐づけてね」です。

結果は、`Import successful!` でした。画面には、次の意味の説明が出ました。

```
このリソースは、Terraformの記録(state)に入りました。これからは、Terraformが管理します
```

## 手順5: もう一度、planを見る

```bash
terraform plan
```

今度は、`No changes. Your infrastructure matches the configuration.` と出ました。コードと実際のAWSが、一致している、という意味です。

## 手順6: 後片付け

取り込んだトピックは、Terraformから削除できるようになっています。最後に、次のコマンドで削除しました。

```bash
terraform destroy
```

`Destroy complete! Resources: 1 destroyed.` と出て、AWS側のトピックも、消えました。

## 学んで整理したこと(実際には試していません)

今回は、terraform import のコマンドで試しました。調べた範囲では、コードの中に import ブロックを書いて取り込む方法も、あると理解しています。こちらは、まだ試していません。

また、今回は、画面で作ったトピックに、タグなどの追加設定をしていません。もし、実際のリソースに、コードに書いていない設定があれば、取り込んだあとのplanで、差分として表示されると思います。この点も、まだ試していません。

## まとめ

- 手作りしたリソースは、Terraformの記録に載っていない
- 取り込む前のplanは、`1 to add` だった
- `terraform import` で取り込むと、Terraformが管理するようになる
- 取り込んだ後のplanは、`No changes.` だった
- 取り込んだあとは、Terraformから、削除もできた

「手作りしたものを、あとからコードで管理する」場面は、実務でも、ありそうです。コマンドの意味を理解して、実際に試せたのは、大きな一歩でした。

それでは、また明日!
