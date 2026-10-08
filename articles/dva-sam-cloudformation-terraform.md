---
title: "SAMはCloudFormationの短縮記法だった。Terraformとの違いも整理した"
emoji: "📐"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["aws", "dva", "terraform", "資格"]
published: true
---

おはようございます、YASUです。

DVAの出題範囲を見ていると、「SAM」「CloudFormation」という言葉が頻繁に出てきます。これまでTerraformでインフラをコード管理する練習をしてきたので、「AWS版のTerraformみたいなものかな」と軽く考えていたのですが、整理してみると似ている部分と全然違う部分がありました。今日はその違いを、Terraformの経験と照らし合わせながらまとめます。

## 結論

CloudFormationはAWS純正のIaC(コードでインフラを管理する仕組み)、SAMはそのCloudFormationを「サーバーレス開発向けに短く書けるようにした拡張」でした。つまり、SAMのテンプレートは最終的にCloudFormationに変換されて動いています。「TerraformのAWS版」というより、「AWS純正の別ルートのIaC」と捉えた方が正確でした。

## まずCloudFormation:AWS純正のIaC

CloudFormationは、AWSのリソースをYAMLやJSONのテンプレートで定義して、そのとおりに作成・変更・削除してくれるサービスです。考え方自体はTerraformとほぼ同じで、「あるべき姿をコードで書いておけば、実際のAWS環境をそれに合わせてくれる」という発想です。

用語が少し違うので、Terraformの経験と対応させて整理してみました。

```
Terraform                    CloudFormation
─────────────────────────────────────────────
.tfファイル                  テンプレート(YAML/JSON)
リソースのまとまり            スタック
terraform plan               変更セット(Change Set)
terraform apply              スタックの作成・更新
terraform destroy            スタックの削除
```

「スタック」は、CloudFormationが管理するリソースのまとまりです。スタックを削除すれば、その中のリソースがまとめて削除されるので、感覚としてはTerraformのdestroyに近く、以前書いた通り、本番環境では取り扱い注意の操作だと感じました。

## Terraformとの大きな違い:状態管理の持ち方

一番大きな違いだと感じたのが、tfstateの扱いです。Terraformでは、「今の状態を記録する台帳」であるtfstateを自分で管理する必要がありました。S3に置いたり、ステートロックを設定したり、以前の記事で書いた通り、そこにはいろいろな気を使うポイントがありました。

CloudFormationの場合は、スタックの状態をAWS側が管理してくれるため、自分でtfstateのようなファイルを用意・保管する必要がありません。この「状態管理をAWSが引き受けてくれる」という点は、運用面で楽な反面、AWSの外のサービス(他のクラウドなど)は管理できない、という制約にもつながっています。Terraformがマルチクラウドに対応しているのと対照的です。

## SAM:サーバーレス専用の「短縮記法」

では、SAMは何なのか。SAMは、CloudFormationの拡張機能で、LambdaやAPI Gateway、DynamoDBといったサーバーレス関連のリソースを、CloudFormationよりずっと短く書けるようにしたものです。

テンプレートの先頭に、こんな一行を書くのが目印になります。

```yaml
Transform: AWS::Serverless-2016-10-31
```

この一行があると、「このテンプレートはSAM形式で書かれているので、CloudFormationの標準形式に変換してから処理してね」という意味になります。実際のイメージは、こんな感じです。

```yaml
Transform: AWS::Serverless-2016-10-31

Resources:
  HelloFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      Runtime: python3.12
      CodeUri: ./src/
      Events:
        HelloAPI:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

これだけで、Lambda関数と、それを呼び出すAPI Gatewayのエンドポイントが定義できます。実はこの短い記述の裏側で、SAMがLambda関数本体、実行に必要なIAMロール、API Gatewayとの接続設定、呼び出し権限など、複数のリソースを自動的に展開してくれています。以前学んだ「実行ロール権限」と「呼び出し権限」の設定を、SAMが自動でやってくれているわけです。

## Globalsセクションで、共通設定をまとめる

SAMには、複数のLambda関数に共通する設定を1か所にまとめるGlobalsという仕組みもあります。

```yaml
Globals:
  Function:
    Timeout: 30
    Runtime: python3.12
    MemorySize: 256
```

これを書いておけば、個々のLambda関数ごとにタイムアウトやメモリ量を繰り返し書く必要がなくなります。以前のTerraformモジュールの回で書いた「共通の型を1つ用意して、個別の値だけ変える」という発想と、とてもよく似ていると感じました。個別の関数で設定を上書きすることもできます。

## SAM CLIという、もう一つの柱

SAMは、テンプレートだけでなく、コマンドラインツール(SAM CLI)も含めた仕組みです。代表的なコマンドはこの2つです。

- `sam build`:アプリケーションをビルドして、デプロイ可能な形に整える
- `sam deploy`:ビルドした内容を、実際のAWS環境にデプロイする(初回は`--guided`オプションで、対話形式で設定を進められる)

SAM CLIには、Lambda関数をローカル環境でテストできる機能もあるとされています。AWSにデプロイする前に、手元で動作確認できるのは、開発者視点のDVAらしい内容だと感じました。

## CloudFormationには「ドリフト検出」という機能もある

以前SOA-C03の勉強で、「CloudFormationのドリフト検出」とCloudTrailの違いが苦手だったのを思い出しました。ドリフト検出は、スタックが管理しているはずのリソースが、後から手動で変更されて、テンプレートの内容とズレていないかを確認する機能です。以前のTerraformのimportの回で触れた、「コードと実際の状態がズレる問題」の、CloudFormation版の対策だと理解しました。

## 使い分けの整理

- サーバーレス(Lambda/API Gateway/DynamoDB中心)を作る → SAM
- AWS全般のリソースをコード管理したい → CloudFormation
- AWS以外も含めて、複数のクラウドを管理したい → Terraform

どれが優れているというより、「何を管理したいか」と「状態管理を誰に任せたいか」で選ぶもの、という整理になりました。

## まとめ

- CloudFormationはAWS純正のIaCで、Terraformと発想は近く、用語が少し違う(スタック、変更セットなど)
- CloudFormationはスタックの状態をAWSが管理してくれるため、tfstateのようなファイルを自分で管理する必要がない
- SAMは、CloudFormationのサーバーレス向け拡張で、先頭のTransformの一行が目印
- SAMは、Lambdaの実行ロールやAPI Gatewayとの接続などを、短い記述から自動で展開してくれる
- CloudFormationには、手動変更によるズレを検出する「ドリフト検出」機能がある

Terraformで学んだ考え方が、AWS純正のIaCを理解する上でも、そのまま土台になっていると実感できた回でした。それでは、また明日!
