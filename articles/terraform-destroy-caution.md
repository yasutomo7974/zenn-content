---
title: "terraform destroy、「元に戻せない」の重みを知った"
emoji: "💥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

これまで`plan`・`apply`・`fmt`・`validate`と、作る・確認する系のコマンドを見てきましたが、今日は逆に「壊す」コマンド、`terraform destroy`について整理します。練習では気軽に使ってきましたが、改めてその危険性と、安全に使うための工夫を確認してみます。

## 結論

terraform destroyは、そのディレクトリでTerraformが管理しているリソースを、まとめて全部削除するコマンドです。練習環境では便利な「片付けコマンド」ですが、本番環境では取り扱いを誤ると重大な事故につながる、諸刃の剣のようなコマンドでした。

## terraform destroyが何をするか

```bash
terraform destroy
```

これを実行すると、Terraformはtfstate(以前紹介した、今の状態を記録している台帳)を見て、「今管理しているリソースを全部削除します」という計画を立て、確認を求めてきます。

```
Plan: 0 to add, 0 to change, 5 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

applyの時と同じように、実行前に必ず確認プロンプトが出ます。ただしメッセージを見ると分かる通り、ここにはapplyにはなかった一文が入っています。「There is no undo(元に戻せません)」という警告です。

## なぜ「元に戻せない」のか

以前紹介したgit revertのように、Gitの世界では「間違えても打ち消しのコミットを追加すれば戻せる」という安全策がありました。しかしTerraformのdestroyは違います。一度実際にAWS上のリソース(EC2インスタンス、S3バケットの中身、RDSのデータベースなど)を削除してしまうと、その中に入っていたデータやファイルは、基本的に本当に消えてなくなります。コードの記録(tfstateやTerraformファイル)は残っていても、実際のデータそのものは戻ってきません。

これが、Gitの取り消し操作とTerraformのdestroyの、決定的に違うところです。

## 練習ではよく使うが、本番では要注意

個人の検証環境では、試して壊して作り直す、というサイクルの中でdestroyは日常的に使います。無駄なAWS利用料がかからないよう、検証が終わったらこまめに削除する、という使い方は健全です。しかし本番環境で同じ感覚で使ってしまうと、稼働中のサービスやデータベースの中身が丸ごと消える、という取り返しのつかない事故になります。

## 安全に使うための工夫

実務では、この危険なコマンドを誤って本番に対して実行しないよう、いくつかの工夫が組み合わされています。

- **環境ごとにディレクトリ・tfstateを分離する**:以前紹介した通り、dev環境とprod環境でtfstateの置き場所自体を分けておけば、devでdestroyしたつもりが本番に影響する、という事故を防げます
- **lifecycle設定で保護する**:特に重要なリソース(本番データベースなど)には、コードの中で削除を防止する設定を付けておくことができます
- **destroyを実行できる人・環境を制限する**:CI/CDのパイプライン上でも、destroyコマンドだけは手動承認を必須にするなど、特別に厳しい運用ルールを設けることが多いです

コードで防止する設定の例はこんな形です。

```hcl
resource "aws_db_instance" "prod" {
  # ...(データベースの設定)

  lifecycle {
    prevent_destroy = true
  }
}
```

`prevent_destroy = true`を設定しておくと、このリソースに対してdestroy(や、意図しない削除を伴うapply)を実行しようとした際に、Terraform自体がエラーを出して処理を止めてくれます。うっかりミスに対する、最後の安全網です。

## 部分的に削除したい場合

全部を削除するのではなく、特定のリソースだけを削除したい場合は、対象を指定するオプションも用意されています。

```bash
terraform destroy -target=aws_instance.web
```

ただし公式ドキュメントでも、この`-target`オプションは「特殊な状況でのみ使うことを想定した機能」とされており、通常の運用では推奨されていません。tfstateとコードの整合性が崩れやすくなるため、多用は避けた方が良いようです。

## まとめ

- terraform destroyは、管理しているリソースをまとめて削除するコマンド
- Gitの取り消し操作と違い、実際のデータは本当に消えてしまい元に戻せない
- 環境ごとのtfstate分離、lifecycle設定、実行権限の制限など、複数の安全策を組み合わせて事故を防ぐ
- `prevent_destroy = true`で、重要なリソースの削除自体をコード側でブロックできる

これまで気軽に打っていたdestroyにも、本番運用ではこれだけの慎重さが求められると分かり、改めて背筋が伸びる思いでした。それでは、また明日!
