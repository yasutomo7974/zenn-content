---
title: "Terraformのリモートステートを試してみた。tfstateをS3に置いて、DynamoDBでロックした"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

Terraformのtfstateを、自分のMacではなく、S3に置く「リモートステート」を、実際に試しました。S3にバケットを作って、ロック用のDynamoDBのテーブルも作り、applyしたらtfstateがS3に保存されるところまで確認しています。今回も、画面とコードの意味をAIに解説してもらいながら、1ステップずつ進めました。

## 結論

`backend "s3"`を書いて`terraform init`すると、applyしたときに、tfstateがS3に保存されました。S3のバージョニングを有効にしておいたので、applyのたびに、古いtfstateもバージョンとして残りました。

## tfstateとは、なぜS3に置くのか

tfstateは、Terraformが「何を作ったか」を書き留めている記録のファイルです。自分のMacに置いたままだと、チームで使うときに、困ることがあります。

- 2人が、別々のコピーを持つと、食い違う
- 同時にapplyしても、止められない
- 秘密の情報が入ることがあるので、GitHubには上げられない

そこで、S3のような、共有できる場所に、1つだけ置きます。この方法を、リモートステートと呼びます。

## 試したことの流れ

1. tfstateを置く、S3バケットを作る(バージョニングを有効にする)
2. ロック用の、DynamoDBのテーブルを作る
3. main.tfに、`backend "s3"`を書く
4. `terraform init`で、準備する
5. `terraform apply`で、SNSトピックを1つ作る
6. S3で、tfstateを確認する
7. コードを少し変えて、もう1回applyする
8. S3で、バージョンが2つになっているのを確認する
9. 片付ける

## 1・2. S3バケットとDynamoDBのテーブルを作る

この2つは、Terraformのコードではなく、先に、コンソールで手作りしました。tfstateの置き場所を、tfstateで管理することはできないためです。

- S3バケット:バージョニングを有効にする。tfstateが壊れても、古いバージョンに戻せるようにするため
- DynamoDBのテーブル:パーティションキーは`LockID`(文字列)、容量モードはオンデマンド。ロックの印を書き込む場所

## 3. main.tfにbackendを書く

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket         = "<バケット名>"
    key            = "backend-test/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "tfstate-lock-test"
    encrypt        = true
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

resource "aws_sns_topic" "test" {
  name = "backend-test-topic"
}
```

- `backend "s3"`は、「tfstateは、このS3に置く」という設定
- `bucket`と`key`は、置き場所のバケット名と、その中でのファイルの場所
- `dynamodb_table`は、ロックに使うテーブル
- `encrypt = true`は、tfstateを暗号化して保存する設定
- 一番下のSNSトピックは、tfstateに記録するものが必要なので、料金のかからない小さなリソースを1つだけ作った

今回使ったTerraformは1.5.7です。S3だけでロックできる機能は、1.10以降なので、このバージョンでは、DynamoDBでロックします。

## 4〜6. initとapplyで、S3に保存された

```bash
terraform init
terraform apply
```

initで、`Terraform has been successfully initialized!`と表示されました。applyのあと、S3のバケットを開くと、`backend-test/`のフォルダの中に、`terraform.tfstate`ができていました。自分のMacではなく、S3に保存されています。

ファイルの中身は、秘密の情報が入ることがあるので、開いていません。

## 7・8. もう1回applyして、バージョンが増えた

SNSトピックに、タグを1つ足して、もう1回applyしました。

```hcl
resource "aws_sns_topic" "test" {
  name = "backend-test-topic"

  tags = {
    Environment = "lab"
  }
}
```

結果は、`Apply complete! Resources: 0 added, 1 changed, 0 destroyed.`でした。トピックは作り直されず、タグだけが、その場で変更されました。

S3で、「バージョンの表示」をオンにすると、`terraform.tfstate`が2つに増えていました。1回目のapplyのもの(2.6 KB)と、タグを足したあとのもの(2.7 KB)です。applyのたびに、新しいtfstateが保存されて、古いものも残ることを、確認できました。

## 9. 片付け

1. `terraform destroy`で、SNSトピックを消す
2. S3バケットを「空にする」で、tfstateの古いバージョンごと消してから、バケットを削除する
3. DynamoDBのテーブルを削除する

S3バケットとDynamoDBのテーブルは、コードに書いていない、手作りのものなので、destroyでは消えません。コンソールで、別に消しました。

## S3とDynamoDBの役割(学んで整理したこと)

- S3:tfstateそのものを置く
- DynamoDB:ロックの印を置く。applyの間は「使用中」の印を付けて、もう1人が同時にapplyしようとすると、待たされる
- この2つは、お互いを知らない。つないでいるのは、Terraformで、backendに書いた名前で、それぞれを使う
- ロックが働く様子は、今回は見ていない。仕組みとして整理しただけ

## GitHubとS3の違い(学んで整理したこと)

- コードは、GitHubで共有する。各自のMacにコピーを持って作業して、pushとpullで合体させる
- tfstateは、GitHubには上げない。applyのたびに内容が変わり、同期が手動のGitでは、食い違うため
- S3なら、1つだけ置いて、applyのたびに、直接読み書きするので、常に最新になる

## まとめ

- tfstateは、Terraformが「何を作ったか」を書き留めた記録。チームで使うときは、S3などの共有の場所に置く(リモートステート)
- `backend "s3"`を書いてinitすると、applyのときに、tfstateがS3に保存された
- S3のバージョニングを有効にすると、applyのたびにバージョンが増えて、古い状態も残る
- ロックはDynamoDBで行う。S3バケットとDynamoDBのテーブルは、コードの外で、先に作っておく
- コードはGitHub、tfstateはS3、と分けて管理する

コードとは別に、「記録を、どこに置くか」を考える必要があることが、実際に試して、よく分かりました。それでは、また明日!
