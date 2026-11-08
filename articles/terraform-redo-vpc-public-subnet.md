---
title: "Terraformを白紙からやり直してみた。VPCからパブリックサブネットまで"
emoji: "🧱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

4月にTerraformでVPCとEC2を作りましたが、時間がたって、書き方も手順も忘れていました。そこで、新しいフォルダで、白紙からやり直してみました。コードとコマンドの意味を、AIに1つずつ解説してもらいながら、1ステップずつ進めた記録です。今回は、VPC、パブリックサブネット、インターネットゲートウェイ、ルートテーブルまでです。

## 結論

Terraformは、「コードを足して、fmt、validate、plan、applyを実行する」という流れを、リソースごとに繰り返すだけでした。実際にやってみて、特に分かったことは3つです。planは「差があるものだけ」表示される、リソースを書く順番は関係ない、AWSの認証情報はコードに書かない、です。

## 準備

- 新しいフォルダ(terraform-lab)を作って、`git init`でGitの管理を始めた
- `.gitignore`に、`.terraform/`、`*.tfstate`、`*.tfstate.*`、`crash.log`、`*.tfvars`、`*.tfvars.json`を書いた。tfstateには秘密の値が入ることがあるので、先にGitに「無視して」と登録しておく
- VS Codeでフォルダを開いて、main.tfを作った

確認したバージョンは、Terraform v1.5.7、AWS CLI 2.34.30、Git 2.51.0です。Terraformは、最新より古いというメッセージが出ましたが、今回の範囲では、問題なく動きました。

## 今回のコード

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "lab-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "lab-public-subnet"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "lab-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "lab-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

## 実行の流れ

1. `terraform init`:プロバイダー(AWS用のプラグイン)をダウンロードする。`~> 5.0`の範囲で、v5.100.0が自動で選ばれた。使ったバージョンが、`.terraform.lock.hcl`に記録された。この時点では、AWSには何もしていない。

2. VPC:コードを足して、fmt(何も表示されなければ、整っている)、validate(Success!)、plan(Plan: 1 to add)、applyの順に実行した。作成は2秒で終わった。

3. サブネット:同じ流れで追加した。planにVPCは出ず、サブネットだけが「作られる」と表示された。作成には11秒かかった。

4. インターネットゲートウェイ:同じ流れで追加して、1秒で作成された。

5. ルートテーブルと関連付け:2つ同時に追加した。Plan: 2 to addと出て、Terraformが、ルートテーブルを先に、関連付けを後に、自動で作った。

## 実際にやって分かったこと

- planは、差があるものだけ表示される:コードに書いてあるが、記録(tfstate)にないものが「作られる」と表示される。すでに作ったVPCは、planに出てこなかった
- 書く順番は関係ない:`aws_vpc.main.id`のような参照を見て、Terraformが作る順番を決める。ルートテーブルが先に作られたのが、その例
- 認証情報はコードに書かない:providerブロックには、リージョンしか書いていない。`aws configure`で保存した設定を、Terraformが自動で読む
- パブリックサブネットの条件:パブリックIPが付くことと、インターネットへの道(ルートテーブルとインターネットゲートウェイ)があることは、別のこと。サブネットだけでは、インターネットに繋がらない

## 作ったもの

- VPC(10.0.0.0/16)
- パブリックサブネット(10.0.1.0/24、AZ:ap-northeast-1a)
- インターネットゲートウェイ
- ルートテーブル(0.0.0.0/0をインターネットゲートウェイへ)と、サブネットへの関連付け

どれも、料金がかからないリソースです。EC2など、課金されるものは、まだ作っていません。

## 次にやること

- セキュリティグループとEC2を足して、user_dataでnginxを入れる
- 確認したら、`terraform destroy`で、すべて片付ける
- モジュール化と、tfstateをS3に置くリモートステートを試す

## まとめ

- Terraformは、コードを足して、fmt、validate、plan、applyを繰り返す流れだった
- planは、コードと記録の差だけを表示し、作る順番は、参照関係から自動で決まる
- 認証情報はコードに書かず、tfstateなどはGitに入れない
- 忘れていても、手を動かすと、記事に書いた内容が、実際の画面と結びついた

以前、記事で整理していた内容が、実際の画面と結びついて、理解が深まりました。続きは、EC2とdestroyまで終えてから、また書きます。それでは、また明日!
