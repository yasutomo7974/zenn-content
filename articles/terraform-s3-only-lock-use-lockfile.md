---
title: "DynamoDBなしで、S3だけでTerraformのロックができるか試してみた"
emoji: "🔏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

前回は、Terraformのリモートステートを、S3とDynamoDBで試しました。そのブログを、Xで紹介したところ、「S3だけでもロックできますよ」というコメントをいただきました。

そこで今回は、DynamoDBを使わずに、S3だけでロックできるかを、実際に試しました。AIに1つずつ手順を教えてもらいながら、進めました。

## 結論

DynamoDBがなくても、S3だけでロックできました。

main.tfの変更は、2つだけです。

- `dynamodb_table = ...` の行を消す
- `use_lockfile = true` を足す

ただし、この機能は **Terraform 1.10以上** が必要です。私のPCは1.5.7だったので、先にバージョンを上げました。

## そもそも、ロックとは

ロックは、「いま使用中」の札です。

2人が同時に、同じ記録(tfstate)を書き換えると、記録がぐちゃぐちゃになります。それを防ぐために、1人が作業している間は、札を出します。札が出ている間、ほかの人は作業できません。

前回は、この札をDynamoDBに置いていました。今回は、S3の中にファイルとして置きます。

## 手順1: Terraformのバージョンを上げる

まず、バージョンを確認しました。

```bash
terraform version
```

結果は、v1.5.7でした。Homebrewの標準版は、1.5.7で止まっていました。そこで、HashiCorp公式の置き場を追加して、入れ直しました。

```bash
brew tap hashicorp/tap
brew uninstall terraform
brew install hashicorp/tap/terraform
```

- `brew tap`: 公式の「ソフトの置き場」を追加する
- `brew uninstall`: 古いTerraformを消す(main.tfなどのコードは消えません)
- `brew install`: 新しいTerraformを入れる

もう一度 `terraform version` を打つと、v1.17.2になっていました。

## 手順2: S3バケットを作る

コンソールで、S3バケットを1つ作りました。バージョニングは、有効にしました。DynamoDBは、作りません。

## 手順3: main.tfを書き換える

backendの部分を、次のようにしました。

```hcl
backend "s3" {
  bucket       = "<バケット名>"
  key          = "lockfile-test/terraform.tfstate"
  region       = "ap-northeast-1"
  use_lockfile = true
  encrypt      = true
}
```

- `dynamodb_table` の行は、消しました
- `use_lockfile = true` を足しました。意味は「ロックを、S3の中のファイルでやる」です
- `key` は、前回と別の場所にしました(前回の記録と混ざらないようにするためです)

## 手順4: 初期化する

保存して、初期化しました。

```bash
terraform init
```

すると、次のエラーが出ました。

```
Error: Backend configuration changed
```

意味は、「保管場所の設定が変わったけど、どうする?」です。選択肢は2つあります。

- `-migrate-state`: 今の記録を、新しい場所に引っ越す
- `-reconfigure`: 引っ越さずに、新しい設定をそのまま使う

今回は、前回のリソースを全部消していて、引っ越す記録がありません。そこで、`-reconfigure` を選びました。

```bash
terraform init -reconfigure
```

「successfully initialized」と出て、成功しました。

## 手順5: applyして、S3の中を見る

SNSトピックを1つ書いておいて、applyしました。

```bash
terraform apply
```

「Apply complete! Resources: 1 added」と出ました。

そのあと、S3のバケットの中を見ました。`lockfile-test/` の中には、`terraform.tfstate` だけがありました。ロックのファイルは、ありませんでした。applyが終わると、消えるからです。

## 手順6: ロックのファイルを見る

ロックのファイルが出る瞬間を、見てみました。次のコマンドを使いました。

```bash
terraform destroy
```

destroyは、最後に `Enter a value:` と聞いて、入力を待ちます。ここで 何も打たずに、S3を開いて、更新ボタンを押しました。

すると、次のファイルが増えていました。

- `terraform.tfstate.tflock`(サイズは275B)

これが、ロックのファイルです。入力を待っている間は、使用中の札が出ている、ということです。

ターミナルに戻って `yes` と打つと、destroyが終わりました。もう一度S3を更新すると、`.tflock` は消えていました。

## 手順7: 後片付け

SNSトピックは、destroyで消えています。S3バケットは、「空にする」を押してから、削除しました。バージョニングが有効なので、過去のバージョンも一緒に消えます。

## 学んで整理したこと(実際には試していません)

今回は1人で試したので、「2人が同時にapplyしたらどうなるか」は、確認していません。調べた範囲では、2人目は「ロックを取れない」というエラーになり、1人目が終わるまで待たされます。

また、今後は、DynamoDBを使う方法より、S3だけの方法が主流になっていくと、理解しています。ただし、これは公式ドキュメントで、もう一度確認したいと思っています。

## まとめ

- S3だけで、ロックできた(DynamoDBは、いらない)
- 変更は、`dynamodb_table` を消して、`use_lockfile = true` を足すだけ
- Terraform 1.10以上が必要。1.5.7だったので、1.17.2に上げた
- ロックのファイル(`.tflock`)は、作業中だけS3に出て、終わると消える
- ロックは、「いま使用中」の札

コメントをくださった方、ありがとうございました。自分の手で試すと、理解がぐっと深まります。

それでは、また明日!
