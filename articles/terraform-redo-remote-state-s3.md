---
title: "Terraformを白紙からやり直してみた(リモートステート編)。tfstateをS3に移した"
emoji: "🪣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

前回、Terraformのコードを、2つのモジュールに分けました。今回は、tfstateの保存先を、Macの中からS3に移す「リモートステート」を試しました。今回も、コードとコマンドの意味をAIに解説してもらいながら、1ステップずつ進めています。

## 結論

保存先をS3に切り替えると、tfstateがS3に保存されて、Macの中のtfstateは更新されなくなりました。S3のバージョニングを有効にしたので、tfstateの過去の版も残りました。

## tfstateとは、なぜS3に置くのか

tfstateは、Terraformが「何を作ったか」を記録するファイルです。これまでは、Macの中にあり、Macが壊れると消えてしまいます。S3に置けば、Macが壊れても残ります。

## やったこと

- S3のバケット(保存先)を作り、バージョニング(上書きしても前の版が残る設定)と、パブリックアクセスのブロック(公開されない設定)を有効にした
- main.tfに、保存先の設定(`backend "s3"`)を足した
- `terraform init -migrate-state`で、保存先を切り替えた
- `terraform apply`で、リソースを作った
- `terraform destroy`で削除し、最後に、バケットも片付けた

```hcl
terraform {
  backend "s3" {
    bucket  = "バケット名"
    key     = "lab/terraform.tfstate"
    region  = "ap-northeast-1"
    encrypt = true
  }
}
```

bucketは保存先のバケット、keyはバケットの中の保存場所とファイル名です。

## 確認できたこと

- applyのあと、S3に`lab/terraform.tfstate`ができた
- Macの中の`terraform.tfstate`は、更新されなくなった
- S3のバージョニングで、tfstateの過去の版が、4つ残った

保存先の設定を変えたら、initをやり直す必要があります。initをやり直す前に`terraform state list`を実行すると、「初期化が必要です」というエラーが出ました。

## 片付け

このバケットは、Terraformの管理の外で作ったので、`terraform destroy`では消えません。保存先をMacに戻して、バケットの中身を、全部の版ごと削除してから、バケットを削除しました。

## ロックは、今回はやっていない

複数人で使う時に、同時にapplyしないための「ロック」も、リモートステートでは、よく使われます。今回は、個人の練習なので、やっていません。次の課題です。

## まとめ

- tfstateの保存先を、Macの中から、S3に移す設定が`backend "s3"`
- 保存先を変えたら、initをやり直して、切り替える
- S3のバージョニングを有効にすると、tfstateの過去の版が残る
- Terraformの外で作ったバケットは、destroyでは消えないので、自分で片付ける

記事で整理していたリモートステートの話が、実際の画面で確認できました。次は、ロックなどを試したいと思います。それでは、また明日!
