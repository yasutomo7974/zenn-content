---
title: "同じコード使い回しで環境を分ける、Terraformのtfvars活用法"
emoji: "🌓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

これまでvariables・outputs・tfstate、plan/applyと基本ファイルを一通り見てきましたが、今日は実務でほぼ確実にぶつかる話、「開発環境と本番環境で、どうやって設定を切り替えるのか」を整理します。

## 結論

同じTerraformコードを使い回しつつ、環境ごとに違う値(インスタンスサイズやリージョンなど)を適用したい場合は、環境ごとに分けた`.tfvars`ファイルを用意して、applyする時に指定してあげるのが基本のやり方です。

## そもそもなぜ環境を分ける必要があるのか

開発中は安いインスタンス(t2.micro)で試して、本番では性能の高いインスタンス(t3.medium)を使いたい、といったように、環境によって欲しい設定は変わってきます。かといって、開発用と本番用でコードを丸ごと別々に書いてしまうと、修正のたびに2箇所直す羽目になり、メンテナンスが大変になります。

そこで使うのが、以前紹介した`variables.tf`と`terraform.tfvars`の仕組みです。コード(main.tf)自体は1つのまま、値だけを環境ごとに差し替える、という考え方です。

## 環境ごとにtfvarsファイルを分ける

```hcl
# dev.tfvars
instance_type = "t2.micro"
environment   = "dev"

# prod.tfvars
instance_type = "t3.medium"
environment   = "prod"
```

variables.tf側では、それぞれの変数を受け取れるように箱だけ用意しておきます。

```hcl
variable "instance_type" {
  type = string
}

variable "environment" {
  type = string
}
```

そしてmain.tf側では、この変数をそのまま使います。

```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
  tags = {
    Environment = var.environment
  }
}
```

## applyする時にファイルを指定する

実際に適用する時は、`-var-file`というオプションで、どちらのtfvarsを使うか指定します。

```bash
# 開発環境に適用したい場合
terraform apply -var-file="dev.tfvars"

# 本番環境に適用したい場合
terraform apply -var-file="prod.tfvars"
```

同じコードのまま、コマンド1つで「どちらの設定値を使うか」を切り替えられる、というのがポイントです。

## もう一段進んだやり方:ディレクトリを分ける

tfvarsを分けるだけでも十分機能しますが、実務ではさらに一歩進んで、環境ごとにディレクトリ自体を分ける構成もよく見かけます。

```
environments/
  dev/
    main.tf
    dev.tfvars
    backend.tf   (tfstateの置き場所もdev用に分ける)
  prod/
    main.tf
    prod.tfvars
    backend.tf   (tfstateの置き場所もprod用に分ける)
```

こうしておくと、以前話したtfstate(今の状態を記録する台帳)も環境ごとに完全に分離されるので、「開発環境で試した変更が、うっかり本番のtfstateに影響してしまう」という事故を根本から防げます。コードの共通化はモジュール(共通部品化の仕組み)を使うことで両立できますが、ここはまた別の機会に扱いたいと思います。

## 環境ごとに分けるべき代表的な値

- インスタンスタイプ(サイズ・性能)
- インスタンスの台数(本番は冗長化のため複数台、開発は1台など)
- ドメイン名やDNS設定
- ログレベルやモニタリングの有効/無効
- tfstateの保存先(S3バケットやキーのパス)

## まとめ

- コードは1つのまま、tfvarsファイルを環境ごとに分けて値だけ切り替えるのが基本
- `-var-file`オプションで、applyする時にどの環境向けか指定する
- 実務ではディレクトリごと環境を分離し、tfstateも環境ごとに独立させることが多い

「同じコードを使い回しつつ、環境ごとに柔軟に値を変える」という発想は、Terraformに限らず設定管理全般で出てくる考え方なので、ここで感覚を掴んでおくと今後も応用が効きそうです。それでは、また明日!
