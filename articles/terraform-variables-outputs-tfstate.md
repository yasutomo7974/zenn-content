---
title: "Terraformのファイル、なんで分かれてるの?variables・outputs・tfstateを整理してみた"
emoji: "🗃️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "tfstate", "初学者"]
published: true
---

おはようございます、YASUです。

先日Terraformの細かい疑問(egressの-1やパブリックIPの話)をまとめましたが、今日はもう一段基本に戻って、Terraformを触り始めた時に誰もが一度は「これ何のためのファイル?」と思うであろう、variables・outputs・tfstateの3つを整理してみます。

## 結論

Terraformのプロジェクトを開くと、main.tf以外にも`variables.tf`、`outputs.tf`、`terraform.tfstate`といったファイルが出てきますが、これらは「値を外に出す」か「値を中に入れる」か「今の状態を記録する」かで役割がきっちり分かれています。

## variables.tf:外から値を入れる窓口

コードの中に直接値を書き込む(ハードコーディングする)のではなく、後から変更しやすいように「変数」として外に出しておくためのファイルです。

```hcl
variable "instance_type" {
  description = "EC2のインスタンスタイプ"
  type        = string
  default     = "t2.micro"
}
```

これを定義しておくと、他のファイルの中で`var.instance_type`という形で呼び出せます。

```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

なぜこれが必要かというと、例えば検証環境では`t2.micro`、本番環境では`t3.medium`を使いたい、というように環境ごとに値を変えたい場面が必ず出てくるからです。コードの中身を毎回書き換えるのではなく、変数の値だけ差し替えれば済むようにしておく、というのが狙いです。

## terraform.tfvars:変数に実際の値を渡すファイル

variables.tfが「箱を用意する」ファイルだとすると、実際にその箱に値を入れるのが`terraform.tfvars`です。

```hcl
instance_type = "t3.medium"
```

このファイルを用意しておくと、defaultで設定した値より優先してこちらの値が使われます。環境ごとに`dev.tfvars`、`prod.tfvars`のようにファイルを分けて、適用時に指定する、という使い方もよくされます。

## outputs.tf:作った結果を表示する窓口

`terraform apply`を実行した後、作成したリソースの情報(パブリックIPやリソースIDなど)を画面に表示させたい場合に使うのがoutputsです。

```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip
}
```

これを設定しておくと、applyが終わった時にターミナル上にそのIPアドレスが表示されます。いちいちAWSコンソールを開いて確認しに行かなくても、Terraformを実行した画面のまま必要な情報が手に入る、という地味に便利な仕組みです。

## terraform.tfstate:Terraformの記憶そのもの

これが一番最初、存在意義がピンとこなかったファイルです。tfstateファイルは、Terraformが「今、実際のAWS上に何がどういう設定で存在しているか」を記録している、いわば台帳のようなファイルです。

なぜこれが必要かというと、Terraformは実行するたびに、このtfstateファイルの内容と、コードに書かれている内容を見比べて、「差分があれば、その分だけ変更を加える」という動き方をするからです。もしこのファイルがなければ、Terraformは毎回「今何が存在しているか」を把握できず、正しく差分を計算できません。

実務では、このtfstateファイルを個人のPCに置いたままにせず、S3などの共有ストレージに置いて、チームメンバー全員が同じ状態を参照できるようにする(リモートステート)のが基本です。ここは今後もう少し深掘りしたいテーマです。

## 4つの関係を図にすると

```
terraform.tfvars ──(値を渡す)──> variables.tf ──(呼び出す)──> main.tf
                                                                    │
                                                              (作成した結果)
                                                                    │
                                                                    ▼
                                                              outputs.tf(画面に表示)
                                                                    │
                                                              terraform.tfstate(記録される)
```

## まとめ

- variables.tf:値を外から入れ替えられるようにする「箱」の定義
- terraform.tfvars:その箱に実際の値を入れるファイル
- outputs.tf:作成した結果を画面に表示する窓口
- terraform.tfstate:今の実際の状態を記録している台帳。差分計算のために必須

最初は「なんでファイルがこんなに分かれているんだろう」と思っていましたが、それぞれ役割がはっきり分かれていると分かると、逆に迷わず読み書きできるようになってきました。それでは、また明日!
