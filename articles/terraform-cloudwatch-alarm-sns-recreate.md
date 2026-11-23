---
title: "コンソールで作成したCloudWatchアラームとSNSの構成を、Terraformで再構成してみた"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "aws", "初学者"]
published: true
---

おはようございます、YASUです。

CloudWatchのアラームが鳴ったら、メールで知らせてくれる仕組みを、AWSの画面で作りました。次の日の朝に、同じものを、Terraformのコードで書いて、作って、消しました。今回も、画面とコードの意味をAIに解説してもらいながら、1ステップずつ進めました。

## 結論

画面で選んだ項目が、Terraformのコードの1行ずつに、そのままなっていました。コードなら、作るのも消すのも、1つのコマンドで済みます。アラームが鳴ると、通知のメールも、実際に届きました。

## 1. 画面で作った

作った順番は、次のとおりです。

1. SNSのトピックを作る(通知の入れ物)
2. トピックに、メールアドレスを登録して、確認メールを承認する
3. テストメッセージを発行して、メールが届くことを確認する
4. CloudWatchで、アラームを作る
5. メッセージを発行して、アラームを鳴らす

先にトピックを作るのは、アラームの「アクション」で、通知先のトピックを選ぶためです。

## アラームの設定

見張る対象は、トピックの「メッセージ発行数」にしました。メッセージを1通送れば、すぐ鳴らせるので、テストに向いています。

| 項目 | 設定 |
|---|---|
| メトリクス | NumberOfMessagesPublished(メッセージ発行数) |
| 統計 | 合計 |
| 期間 | 1分 |
| 条件 | 1以上 |
| アクション | アラーム状態になったら、SNSトピックに通知 |

意味は、「1分間に、メッセージが1通以上発行されたら、アラームを鳴らして、メールを送る」です。

## 鳴らした結果

SNSの画面から、メッセージを発行しました。しばらくすると、アラームの状態が「データ不足」から「アラーム状態」(赤)に変わり、通知のメールが届きました。メールには、次のように書かれていました。

```
State Change: INSUFFICIENT_DATA -> ALARM
```

データ不足からアラームに変わった、という意味です。

## 2. 同じものを、Terraformで作った

画面で作った3つを、コードにしました。対応は、次のとおりです。

| 画面でやったこと | コードでの名前 |
|---|---|
| SNSトピックを作る | aws_sns_topic |
| メールを登録する | aws_sns_topic_subscription |
| アラームを作る | aws_cloudwatch_metric_alarm |

練習用の、新しいフォルダを作って、main.tfに書きました。メールアドレスは、コードに直接書かず、変数にして、実行するときに入力する形にしています。

```hcl
variable "alert_email" {
  type = string
}

resource "aws_sns_topic" "alarm" {
  name = "tf-alarm-topic"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alarm.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

resource "aws_cloudwatch_metric_alarm" "publish_test" {
  alarm_name          = "tf-publish-test-alarm"
  namespace           = "AWS/SNS"
  metric_name         = "NumberOfMessagesPublished"
  dimensions = {
    TopicName = aws_sns_topic.alarm.name
  }
  statistic           = "Sum"
  period              = 60
  evaluation_periods  = 1
  threshold           = 1
  comparison_operator = "GreaterThanOrEqualToThreshold"
  alarm_actions       = [aws_sns_topic.alarm.arn]
}
```

アラームの部分は、画面で選んだ項目が、1行ずつ、そのまま書かれています。

- `metric_name`は、見張る対象(メトリクス)
- `statistic = "Sum"`は、統計の合計
- `period = 60`は、期間の1分(60秒)
- `threshold = 1`と`GreaterThanOrEqualToThreshold`は、「1以上」という条件
- `alarm_actions`は、鳴ったときに通知するトピック(画面の「アクション」)

## コマンドの流れ

1. `terraform init`:準備。AWS用の部品をダウンロードする
2. `terraform plan`:何が作られるかを確認する。`Plan: 3 to add`と表示された
3. `terraform apply`:実際に作る。メールアドレスを入力して、確認で`yes`と入力した。`Apply complete! Resources: 3 added`と表示された
4. 確認メールを承認して、画面と同じように、メッセージを発行した
5. アラームが「アラーム状態」(赤)になって、通知メールが届いた
6. もう一度`terraform apply`を実行すると、`No changes. Your infrastructure matches the configuration.`と表示された
7. `terraform destroy`:全部消す。`Destroy complete! Resources: 3 destroyed.`と表示された

6番の`No changes`は、コードと実際のAWSが、ぴったり一致している、という意味です。

コードで作ったものは、AWSのコンソールにも、同じように作られていました。画面で作るか、コードで作るかは、作り方が違うだけで、できあがるものは同じです。

## 画面とコードの違い

- 作る:画面は、1つずつ操作する。コードは、applyの1コマンド
- 消す:画面は、1つずつ消す。コードは、destroyの1コマンドで、コードに書いたものが全部消える
- 記録:画面の操作は、記録が残らない。コードは、変更の履歴が残る

## 学んで整理したこと(今回は試していない)

- アラームの設定を変えたいときは、コードを直して、planで確認してから、applyする。画面で変えると、コードとずれて(ドリフト)、次のapplyで、コードの値に戻る
- アラームは、選んだメトリクスだけを見張る。今回のメッセージ発行数は、テストで鳴らしやすいから選んだだけで、実際には、見張りたい内容に合わせて選ぶ(たとえば、EC2が重いかどうかなら、CPU使用率)
- アラームのアクションは、人に知らせるならSNS、自動で対処するならAuto Scalingなどを指定できる

## まとめ

- 順番は、SNSトピックとメール登録を先に作ってから、アラームのアクションで、トピックを指定する
- 画面で選んだ項目は、Terraformのコードの1行ずつに、そのまま対応している
- init → plan → apply → destroyの流れで、作るのも消すのも、1コマンドでできた
- 設定を変えるときは、画面ではなく、コードを直す

画面で一度作ってから、コードにすると、1行ずつの意味が、よく分かりました。それでは、また明日!
