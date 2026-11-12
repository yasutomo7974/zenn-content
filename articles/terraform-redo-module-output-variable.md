---
title: "Terraformを白紙からやり直してみた(モジュール編)。出口と入口で値を受け渡す"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

前回までに、Terraformで、VPCからEC2まで(nginxの画面を確認して、destroyまで)を、1つのmain.tfで作りました。今回は、そのコードを、モジュールに分けてみました。networkとwebの2つのモジュールです。今回も、コードとコマンドの意味をAIに解説してもらいながら、1ステップずつ進めています。

## 結論

モジュールに分けても、planの結果は元と同じ7つで、作られる構成は変わりませんでした。変わったのは、コードの置き場所と、リソースの「住所」(`module.network.aws_vpc.main`のように、先頭にモジュール名が付く)だけです。モジュールの中の値は、外から直接は見えないので、outputで出して、variableで受け取る、という受け渡しが必要でした。

## 準備:作業用のブランチを作る

コードを大きく書き換える前に、`git switch -c refactor-modules`で、作業用のブランチを作りました。ブランチは、mainから枝分かれさせて、別の場所で作業を進める仕組みです。動くことを確認したmainは、そのまま残して、枝の上で書き換えます。

## モジュールとは

関連するリソースを、1つのフォルダにまとめた「部品」です。今回は、次のように分けました。

```
terraform-lab/
├── main.tf              ← モジュールを呼び出すだけ
└── modules/
    ├── network/         ← VPC、サブネット、インターネットゲートウェイ、ルートテーブル
    └── web/             ← セキュリティグループ、EC2
```

## 1. networkモジュール

`modules/network/main.tf`に、前回までのネットワークの5つのリソース(VPC、サブネット、インターネットゲートウェイ、ルートテーブル、関連付け)を、そのまま移しました。ルートのmain.tfには、呼び出しだけを書きます。

```hcl
module "network" {
  source = "./modules/network"
}
```

この状態で`terraform validate`を実行すると、次のエラーが出ました。

```
Error: Reference to undeclared resource
  vpc_id = aws_vpc.main.id
A managed resource "aws_vpc" "main" has not been declared in the root module.
```

「`aws_vpc.main`は、ルートモジュールの中に宣言されていない」という意味です。VPCは、networkモジュールの中に引っ越したので、ルートのmain.tfからは、直接呼べなくなりました。モジュールは、中身が外から見えない箱でした。

箱の中の値を外に渡すには、モジュールの中に、outputs.tfで出口を作ります。

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}
```

外側では、`module.network.vpc_id`のように、`module.モジュール名.output`の名前で受け取ります。セキュリティグループとEC2の該当する2行を、この形に書き換えると、validateが通りました。

planは`Plan: 7 to add`で、前回と同じでした。ネットワークの5つは、`module.network.aws_vpc.main`のように、名前の先頭に`module.network.`が付きました。applyのあと、ブラウザで「Welcome to nginx!」が表示されたことを確認して、destroyしました。

## 2. webモジュール

セキュリティグループ、AMIを探すdata、EC2を、`modules/web`に移しました。ただし、webは、VPCのIDとサブネットのIDを、外から受け取る必要があります。そこで、入口として`variables.tf`を作りました。

```hcl
variable "vpc_id" {
  type = string
}

variable "subnet_id" {
  type = string
}
```

main.tfの中では、`var.vpc_id`、`var.subnet_id`として使います。出口は、`outputs.tf`で、EC2のパブリックIPを渡しています。

```hcl
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

ルートのmain.tfは、次の形になりました。

```hcl
module "network" {
  source = "./modules/network"
}

module "web" {
  source    = "./modules/web"
  vpc_id    = module.network.vpc_id
  subnet_id = module.network.public_subnet_id
}

output "web_public_ip" {
  value = module.web.public_ip
}
```

networkの出口から値を受け取って、webの入口に渡しています。variables.tfを作る前は、`var.vpc_id`に、VS Codeの赤い波線(宣言されていない、という警告)が出ていましたが、作ったあとは消えました。

## 実行の流れ

- `terraform init`:`Initializing modules...`と、`- web in modules/web`が表示された。モジュールを足したり変えたりしたら、initをやり直す
- `terraform fmt`と`terraform validate`:Success!
- `terraform plan`:Plan: 7 to add。ネットワークの5つは`module.network.`、セキュリティグループとEC2は`module.web.`が付いた
- `terraform apply`:7 added。ログを見ると、VPCのあと、インターネットゲートウェイ、サブネット、セキュリティグループが同時に作られ始めていた。お互いに依存していないので、並行して作れる
- ブラウザで、「Welcome to nginx!」が表示されることを確認
- `terraform destroy`:7 destroyed。関連付けとEC2が先に、VPCが最後に消えた。state listは空になった

applyのあとの`terraform state list`には、`module.web.data.aws_ami.ubuntu`も出ていました。dataブロックの「調べた結果」も、tfstateに記録されるためです。

## Gitにコミットして、mainにマージ

モジュールごとに、コミットを残しました。

- `refactor: split network resources into a module`
- `refactor: split web resources into a module`

動作を確認したので、`git switch main`、`git merge refactor-modules`で、mainにマージしました。mainは、枝から枝分かれしたあと、進んでいなかったので、マージコミットは増えず、早送り(fast-forward)になりました。`git log --oneline`では、3つのコミットが、1本に並びました。

## まとめ

- モジュールは、関連するリソースをまとめた部品。ルートのmain.tfは、モジュールを呼び出すだけになった
- モジュールの中の値は、外から直接見えない。外に渡すのはoutput(出口)、受け取るのはvariable(入口)
- 外側では、`module.モジュール名.output`の名前で値を受け取る
- モジュールに分けても、planの結果は同じ7つ。変わったのは、リソースの住所(`module.network.`などが付く)
- モジュールを足したらinitをやり直す。作る順番は、参照関係から、Terraformが決める

以前、記事で整理していた「モジュールは、型を作って、材料を流し込む」という内容が、実際にコードを動かして、出口と入口の形で見えました。次は、tfstateをS3に置くリモートステートや、devとprodの切り替えを試したいと思います。それでは、また明日!
