---
title: "Terraformを白紙からやり直してみた(続き)。EC2でnginxを動かして、destroyまで"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

前回、Terraformを白紙からやり直して、VPC、パブリックサブネット、インターネットゲートウェイ、ルートテーブルまでを作りました。今回はその続きで、セキュリティグループとEC2を足して、ブラウザでnginxの画面を確認し、最後にdestroyで片付けるところまで、実際にやってみました。今回も、コードとコマンドの意味をAIに解説してもらいながら、1ステップずつ進めています。

## 結論

前回までのコードに、セキュリティグループとEC2を足すだけで、「Welcome to nginx!」の画面が表示されるところまで、コードだけで作れました。確認のあとは、`terraform destroy`の1コマンドで、7つのリソースをまとめて削除できました。EC2は課金されるので、確認したらすぐに片付けています。

## 再開時の確認

作業を再開する時は、まず`terraform state list`で、Terraformが管理しているリソースを確認しました。前回作った5つ(VPC、サブネット、インターネットゲートウェイ、ルートテーブル、関連付け)が、そのまま残っていました。

## 1. セキュリティグループ

セキュリティグループは、EC2に出入りする通信を許可するルール表です。今回は、ブラウザからのアクセスのために、HTTP(80番ポート)だけを許可しました。SSH(22番ポート)は、今回は開けていません。

```hcl
resource "aws_security_group" "web" {
  name        = "lab-web-sg"
  description = "Allow HTTP inbound"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "lab-web-sg"
  }
}
```

- ingressは入ってくる通信、egressは出ていく通信のルール
- descriptionは、人が読むためのメモ書きで、通信のルールには関係しない
- Terraformで作る場合、egressを書かないと、出ていく通信がすべて許可されない。今回のEC2は、起動時にnginxをダウンロードするため、出ていく通信が必要

## 2. AMIを探す(dataブロック)

EC2を作るには、OSの元になるAMIのIDが必要です。AMIのIDは、リージョンごとに違って、更新もされるので、IDを直接書かずに、dataブロックで、最新のUbuntuを探すようにしました。

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}
```

resourceが「作る」ものなのに対して、dataは「すでにあるものを調べて、読み取る」ものです。ownersのIDは、Ubuntuを提供しているCanonical社のものです。planの時にエラーは出ず、AMIが見つかりました。

## 3. EC2

```hcl
resource "aws_instance" "web" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl enable --now nginx
  EOF

  tags = {
    Name = "lab-web-ec2"
  }
}

output "web_public_ip" {
  value = aws_instance.web.public_ip
}
```

- user_dataは、EC2の起動時に自動で実行される命令。ここで、nginx(Webサーバー)をインストールして、起動している
- outputは、作成後に、パブリックIPを画面に表示するためのもの
- dataで探したAMIは、`data.`を付けて参照する

## 実行の流れ

どのリソースも、fmt、validate、plan、applyの順で進めました。planは毎回、既存のリソースが画面に出ず、今回足した分だけが「作られる」と表示されました。

- セキュリティグループ:Plan: 1 to add。作成は3秒
- EC2:Plan: 1 to addに加えて、Changes to OutputsとしてパブリックIPの予告が出た。user_dataは、planの画面では、ハッシュ値で表示された。作成は13秒

applyが終わると、outputで書いたパブリックIPが表示されました。

## ブラウザで確認

パブリックIPを、`http://`から始めて、ブラウザで開くと、「Welcome to nginx!」の画面が表示されました。EC2が起動してから、user_dataのnginxのインストールが終わるまで、少し時間がかかるので、すぐに開くと表示されないことがあります。また、セキュリティグループで開けたのはHTTPだけなので、`https://`ではなく、`http://`で開きます。

この画面が見えたことで、VPC、サブネット、インターネットゲートウェイ、ルートテーブルがつながっていて、セキュリティグループの80番が通り、user_dataが実行された、ということを確認できました。

## destroyで片付ける

EC2は動いている間、課金されるので、確認したらすぐにdestroyしました。

```bash
terraform destroy
```

確認画面でyesと入力すると、`Destroy complete! Resources: 7 destroyed.`と表示されました。削除の順番は、作った時と逆で、Terraformが参照関係から、自動で決めます。片付けたあとは、次の2つで確認しました。

- `terraform state list`が、空になった
- AWSコンソールの東京リージョンで、lab-web-ec2が「終了済み」になっていた

## Gitにコミット

最後に、コードをローカルのGitにコミットしました。`git status`で、コミット前のファイルとして表示されたのは、`.gitignore`、`.terraform.lock.hcl`、`main.tf`の3つだけで、`.terraform/`や`terraform.tfstate`は表示されませんでした。`.gitignore`で除外しているためです。コミットしたのは、この3つです。

## まとめ

- セキュリティグループとEC2を足して、user_dataでnginxを自動インストールすると、ブラウザで「Welcome to nginx!」が表示された
- AMIのIDは、直接書かず、dataブロックで探した。resourceは作るもの、dataは読み取るもの
- Terraformでは、egressを書かないと、出ていく通信がすべて許可されない。descriptionはメモ書き
- 確認後は、`terraform destroy`で7つのリソースをまとめて削除し、state listと、コンソールで確認した
- コミットしたのは、コードと、`.terraform.lock.hcl`、`.gitignore`。tfstateは、`.gitignore`で除外している

コードを書いて、確認して、片付けるまでを一通り通すと、前に記事で整理した内容が、実際の画面とつながりました。次は、モジュール化と、tfstateをS3に置くリモートステートを試したいと思います。それでは、また明日!
