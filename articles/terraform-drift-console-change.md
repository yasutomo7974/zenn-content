---
title: "Terraformをコンソールで手作業変更したらどうなる?ドリフトを実際に試した"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

Terraformで作ったリソースを、AWSコンソールで手作業で変更すると、どうなるのか。実際に試してみました。VPCを1つ作って、タグをコンソールで書き換え、planとapplyで何が起きるかを確認しています。今回も、コードとコマンドの意味をAIに解説してもらいながら、1ステップずつ進めました。

## 結論

コンソールからの変更は、エラーなく、できました。ただ、コードと実際の状態に「ずれ」(ドリフト)が起きて、planに差分として表示されました。そのままapplyすると、コードの値に戻りました。

## 試したこと

これまでのコードとは別のフォルダを作って、VPCを1つだけ作りました。タグは2つです。

```hcl
resource "aws_vpc" "drift_test" {
  cidr_block = "10.1.0.0/16"

  tags = {
    Name        = "drift-test-vpc"
    Environment = "lab"
  }
}
```

タグは、リソースに付けるラベルです。Nameはコンソールに名前として表示されるキーで、Environmentは自分で決めた、自由なキーです。動作に影響しない設定なので、実験に向いていると考えました。

## 実験の流れ

1. `terraform apply`で、VPCを作った(1 added)。Environmentはlab
2. AWSコンソールで、Environmentの値を、labからmanualに書き換えた。エラーは出なかった
3. `terraform plan`を実行した
4. `terraform apply`を実行した
5. コンソールで、値がlabに戻っていることを確認し、`terraform destroy`で片付けた

## planの結果

3番のplanでは、次のように表示されました。

```
# aws_vpc.drift_test will be updated in-place
  ~ tags = {
      ~ "Environment" = "manual" -> "lab"
        "Name"        = "drift-test-vpc"
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

- `~`は変更、`updated in-place`は、その場で変更するという意味。VPCは作り直されず、タグだけが書き換わる
- 矢印の左の`manual`が、今の実際のAWSの値。右の`lab`が、コードの値
- 最後の行は、追加0、変更1、削除0

コードは`lab`、実際のAWSは`manual`というずれを、Terraformが見つけて、差分として教えてくれました。

## applyの結果

applyを実行すると、`Apply complete! Resources: 0 added, 1 changed, 0 destroyed.`と表示されました。コンソールで見ると、Environmentの値は、labに戻っていました。Terraformは、コードに書かれた値を正解として扱い、実際のAWSを、コードに合わせて直します。

## ずれとは

同じタグについて、3つの場所に、値が書かれていました。

- Terraformのコード:lab
- Terraformの記録(tfstate):lab
- 実際のAWS(コンソールで変更したあと):manual

最初は、3つともそろっていました。コンソールで、実際のAWSだけを書き換えたので、食い違いが起きました。この食い違いを、ドリフトと呼びます。

## 分かったこと

- コンソールで変更すること自体は、できる。エラーにはならない
- 変更を残したい場合は、コードも同じ値に書き換える必要がある。書き換えなければ、次のapplyで、コードの値に戻る
- だから、最初から、コードで変更するのが原則。コードに変更が残り、planで、事前に何が変わるかも確認できる

## まとめ

- Terraformで作ったリソースをコンソールで変更すると、コード・記録・実際の状態がずれる(ドリフト)
- planに、ずれが差分として表示される(`"manual" -> "lab"`)
- applyすると、コードの値に戻る
- 変更は、コンソールではなく、コードで行うのが原則

記事で整理していた「コンソールで手作業をすると、コードとずれる」という内容を、実際にタグを書き換えて、画面で確認できました。それでは、また明日!
