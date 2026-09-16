---
title: "tfstateの中身を直接覗く、terraform state list/show"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

これまでtfstateについて「今の状態を記録する台帳」と何度も説明してきましたが、実はその中身を直接コマンドで覗く方法をまだ紹介していませんでした。今日はterraform state listとterraform state showを整理します。

## 結論

terraform state listは、今tfstateが管理しているリソースの一覧を表示するコマンド、terraform state showは、その中の1つを選んで詳細を表示するコマンドです。どちらも中身を「見る」だけで、何かを変更するわけではない、安全なコマンドでした。

## terraform state list:管理下のリソース一覧を見る

```bash
terraform state list
```

これを実行すると、今のtfstateが把握しているリソースが、こんな形で一覧表示されます。

```
aws_vpc.main
aws_subnet.public
aws_internet_gateway.main
aws_route_table.public
aws_instance.web
```

以前紹介した通り、tfstateは「今、実際のAWS上に何が存在しているか」を記録している台帳でした。state listは、その台帳の中に、今どんな見出し(リソース名)が並んでいるかを一覧で確認できるコマンドです。

## terraform state show:1つのリソースの詳細を見る

一覧の中から、気になる1つを選んで詳しく見たい時に使います。

```bash
terraform state show aws_instance.web
```

実行すると、そのリソースの現在の設定内容が、細かい項目までまとめて表示されます。

```
# aws_instance.web:
resource "aws_instance" "web" {
    ami                    = "ami-0123456789abcdef0"
    instance_type          = "t2.micro"
    availability_zone      = "ap-northeast-1a"
    id                     = "i-0123456789abcdef0"
    private_ip             = "10.0.1.10"
    public_ip              = "3.112.xxx.xxx"
    tags                   = {
        "Name" = "web-server"
    }
    # ...(他にも多数の項目)
}
```

コードファイル(.tf)の中には自分で書いた項目しか載っていませんが、state showで表示される内容には、AWS側で自動的に割り振られた値(IDやプライベートIPなど)も含めて、実際の状態がすべて反映されています。

## なぜこの2つが実務で役立つのか

以前紹介したterraform importの回で、「importした後、実際の設定内容をコードに手作業で書き写す必要がある」という話をしました。その際、コードに書き写すべき値を確認する手段として、まさにこのstate showが活躍します。

```bash
# importした後、実際の内容を確認する
terraform state show aws_instance.web

# 表示された内容を見ながら、コード側に反映していく
```

また、「このリポジトリでは、そもそも何を管理しているんだっけ?」と分からなくなった時、state listで全体像をまず把握してから、気になるものだけstate showで深掘りする、という使い方もできます。ファイルを1つずつ開いて確認するより、コマンド一発で今の状態を確認できるのは、実務で規模の大きいプロジェクトを扱うときに助かりそうです。

## コードと見比べて、差分に気づく手がかりにもなる

コード(.tf)には書いていないはずの値がstate showの結果に含まれていたり、逆にコードに書いた値が反映されていなかったりする場合、それは何か想定と違うことが起きているサインかもしれません。以前紹介したgit diffで変更差分を確認する習慣と同じように、planを実行する前に、まず今の状態をstate showで目視確認しておく、という一手間が、意図しない変更を防ぐことに繋がりそうです。

## 安全性について

state listもstate showも、あくまでtfstateの中身を「読み取って表示する」だけのコマンドです。実際のAWSリソースはもちろん、tfstateファイル自体にも変更を加えません。以前紹介したfmt、validate、consoleと同じ、安全に何度でも実行できるコマンドの仲間です。

## まとめ

- terraform state listは、tfstateが管理しているリソースの一覧を表示するコマンド
- terraform state showは、その中の1つを選んで、詳細な設定内容を表示するコマンド
- どちらも中身を確認するだけで、実際のリソースやtfstateには変更を加えない
- import後の確認作業や、applyの前の状態確認に役立つ

tfstateという言葉だけは何度も使ってきましたが、実際にコマンドでその中身を覗いてみると、Terraformが裏側でどれだけ細かい情報を管理しているのか、改めて実感できました。それでは、また明日!
