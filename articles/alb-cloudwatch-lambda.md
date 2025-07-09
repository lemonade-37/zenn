---
title: "【ALB・CloudWatch・Lambda】サーバーダウン等の異常をALBで検知した際にSlackに通知する"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "CloudWatch", "Lambda", "ALB"]
published: true
published_at: 2024-05-30 12:00
---
## 自己紹介
はじめまして、はると申します。
駆け出しエンジニアとして働き始めて約3ヶ月が経過しました🐣


## 概要
サーバーダウンにいち早く気づき、異常の対処や再起動ができるために、ALBの異常を検知した際に、Slackに通知を送る機能を実装します。
下記の構成でEC2にデプロイされており、ロードバランサー（ALB）で1分毎にHTTPリクエストを送信し、サーバーが起動しているか確認する機能はすでに導入されている前提とします。

今回のインフラ構成図のイメージです。
![](/images/alb-cloudwatch-lambda/image01.png)

### 環境
- Docker
- Ruby 3.2.3
- Rails 7.1.3
- PostgreSQL

## Slackに通知するLambdaの作成
### Incoming Webhookの設定
1. Slackアプリ内で、Botの通知メッセージを送信させたいチャンネルを右クリックし、`チャンネル詳細を表示する`を開きます。
![](/images/alb-cloudwatch-lambda/image02.png)
　　
1. `インテグレーション`タブを開き、`アプリを追加する`を選択します。
![](/images/alb-cloudwatch-lambda/image03.png)

1. `Incoming Webhook`を検索し、`インストール`を選択します。
![](/images/alb-cloudwatch-lambda/image04.png)
　　
1. ブラウザに遷移するので、`Slackに追加`を選択します。
![](/images/alb-cloudwatch-lambda/image05.png)
　　
1. 通知メッセージを送信させたいチャンネルを選択し、`Incoming Webhookインテグレーションの追加`を選択します。
![](/images/alb-cloudwatch-lambda/image06.png)
　　
1. 次のページで、`セットアップの手順`内にある、`Webhook URL`を控えておきます。（後から確認することもできます）
確認できたら、一番下までスクロールし、`設定を保存する`をクリックします。
（任意で、Botの名前やアイコンのカスタマイズもできます）
![](/images/alb-cloudwatch-lambda/image07.png)
　　
1. これでSlack側の設定は以上です🎉

　　
### Lambda用のIAMロールを作成
1. `IAM`にアクセスし、サイドバーの`ロール`＞`ロールの作成`を開きます。
　　
1. 信頼されたエンティティタイプは`AWSサービス`、ユースケースで`Lambda`を選択します。
![](/images/alb-cloudwatch-lambda/image08.png)
　　
1. 許可ポリシーは、`AWSLambdaBasicExecutionRole`を検索し、選択します。
![](/images/alb-cloudwatch-lambda/image09.png)
　　
1. ロールの詳細は、任意のロール名を入力します。他はデフォルトのまま使用します。
![](/images/alb-cloudwatch-lambda/image10.png)

　　
### Lambdaの関数を作成
1. `Lambda`にアクセスし、`関数の作成`を開きます。
1. `一から作成`を選択し、任意の関数名を入力し、任意のランタイム（今回はRubyとします）、アーキテクチャを選択します。
デフォルトの実行ロールの変更では、`既存のロールを使用する`を選択し、先ほど作成したIAMロールを選択します。詳細設定はデフォルトのままにします。
このページでの設定内容は以下の画像の通りです。入力を終えたら、`関数の作成`をします。
![](/images/alb-cloudwatch-lambda/image11.png)

1. 関数が作成されたら、コードソースを下記の内容に編集します。
ここで、先ほど控えておいたWebhook URLが必要になりますが、直接入力せず環境変数を使用します。
**編集後は、`Deploy`をクリックするのを忘れずに！**
    ```ruby
    require 'json'     # JSONの生成に使用
    require 'net/http' # HTTPリクエスト送信に使用
    require 'uri'      # URIの読み込みに使用

    # CloudWatchのアラーム情報を引数として受け取りLambda関数を実行するメソッド
    def lambda_handler(event:, context:) # eventはLambda関数実行のトリガーとなったイベントのデータ
      puts "Received Event: #{event.inspect}"

      alarm_data = event.dig('alarmData') || {}                                   # alarmDataを取得
      alarm_name = alarm_data.dig('alarmName') || "No alarm name provided"        # alarmNameを取得
      state_value = alarm_data.dig('state', 'value') || "No state value provided" # state下のvalueを取得
      reason = alarm_data.dig('state', 'reason') || "No reason provided"          # state下のreasonを取得

      slack_message = "Alarm Name: #{alarm_name}\nState: #{state_value}\nReason: #{reason}" # Slackに送るメッセージを生成
      post_slack(slack_message) # ここで下記に定義しているメソッドを呼び出す
    end

    # Slackに通知するメソッド
    def post_slack(message)
      webhook_url = ENV['SLACK_WEBHOOK_URL'] # ここで後に取得したSlackのWebhookURLを使用します
      uri = URI.parse(webhook_url)
      header = {'Content-Type' => 'application/json'}
      send_data = {
        "text" => "<!channel> #{message}", # <!channel>を入れることで、@channelのメンション入りで通知できます
        "link_names": 1
      }.to_json # ハッシュをJSON文字列に変換

    	# HTTPリクエストを送信
      Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == 'https') do |http|
        request = Net::HTTP::Post.new(uri, header)
        request.body = send_data         # 先ほどJSONに変換したデータを設定
        response = http.request(request) # リクエストを送信し、レスポンスを受け取る
        puts response.body               # 受け取ったレスポンスをログに出力（CloudWatchのロググループでLambda実行のログを見ることができる）
      end
    end

    ```
    ![](/images/alb-cloudwatch-lambda/image12.png)
　　
1. 環境変数の設定
`設定タブ`＞サイドバーの`環境変数`から、環境変数を登録します。
キーは`SLACK_WEBHOOK_URL`、値はIncoming Webhookのページよりコピーした`Webhook URL`を使用します。
![](/images/alb-cloudwatch-lambda/image13.png)
![](/images/alb-cloudwatch-lambda/image14.png)
　　
1. 動作テスト
`テスト`タブを開き、`新しいイベントを作成`を選択し、任意のイベント名を設定します。
テンプレートオプションは今回は使用せず、イベントJSONにテスト用のJSONデータを直接記載していきます。
![](/images/alb-cloudwatch-lambda/image15.png)


    下記は、CloudWatchがアラーム状態になった時にLambdaに送られてくるデータを一部加工したテスト用のJSONです。
    ```json
    {
      "source": "aws.cloudwatch",
      "alarmArn": "test-alb-alarm",
      "accountId": "1234567890",
      "time": "2024-04-09T08:56:06.386+0000",
      "region": "ap-northeast-1",
      "isTest": true,
      "testDescription": "This is a test event for Lambda function.",
      "alarmData": {
        "alarmName": "test-alb-alarm",
        "state": {
          "value": "ALARM",
          "reason": "Threshold Crossed: 1 out of the last 1 datapoints [1.0 (09/04/24 08:54:00)] was greater than or equal to the threshold (1.0) (minimum 1 datapoint for OK -> ALARM transition).",
          "reasonData": "{\"version\":\"1.0\",\"queryDate\":\"2024-04-09T08:56:06.384+0000\",\"startDate\":\"2024-04-09T08:54:00.000+0000\",\"statistic\":\"Minimum\",\"period\":60,\"recentDatapoints\":[1.0],\"threshold\":1.0,\"evaluatedDatapoints\":[{\"timestamp\":\"2024-04-09T08:54:00.000+0000\",\"sampleCount\":2.0,\"value\":1.0}]}"
        },
        "previousState": {
          "value": "OK",
          "reason": "Threshold Crossed: 1 out of the last 1 datapoints [0.0 (08/04/24 10:13:00)] was not greater than or equal to the threshold (1.0) (minimum 1 datapoint for ALARM -> OK transition).",
          "reasonData": "{\"version\":\"1.0\",\"queryDate\":\"2024-04-08T10:15:06.384+0000\",\"startDate\":\"2024-04-08T10:13:00.000+0000\",\"statistic\":\"Minimum\",\"period\":60,\"recentDatapoints\":[0.0],\"threshold\":1.0,\"evaluatedDatapoints\":[{\"timestamp\":\"2024-04-08T10:13:00.000+0000\",\"sampleCount\":2.0,\"value\":0.0}]}"
        },
        "configuration": {
          "metrics": [
            {
              "id": "testid1234567890",
              "metricStat": {
                "metric": {
                  "namespace": "AWS/ApplicationELB",
                  "name": "UnHealthyHostCount",
                  "dimensions": {
                    "TargetGroup": "targetgroup/test-app/1234567890",
                    "LoadBalancer": "app/test-app-ALB/1234567890",
                    "AvailabilityZone": "ap-northeast-1a"
                  },
                  "period": 60,
                  "stat": "Minimum"
                },
                "returnData": true
              }
            }
          ]
        }
      }
    }
    ```
    　　
1. Lambdaのアクセス権限の追加
最後に、CloudWatchからLambdaの関数を実行できるためのアクセス権限を、Lambdaに追加します。
`設定`タブを開き、サイドバーの`アクセス権限`＞`リソースベースのポリシーステートメント`内にある`アクセス権限を追加`をクリックします。
`AWSアカウント`を選択し、ステートメントIDは任意のIDを入力し、
プリンシパルは`lambda.alarms.cloudwatch.amazonaws.com`と入力、
アクションは`lambda:InvokeFunction`を選択します。
これで、Lambdaの設定は以上です！🎊
![](/images/alb-cloudwatch-lambda/image16.png)

　　
### CloudWatchのアラームを作成
では、すでに動いているALBの異常を検知してアラームためのアラームを作成していきます。

1. `CloudWatch`にアクセスし、サイドバーの`アラーム`＞`すべてのアラーム`を開き、`アラームの作成`を選択します。
　　
1. メトリクスの選択で、`UnHealthyHostCount`と検索し、`AppELB 別、AZ 別、TG 別メトリクス`を選択します。
該当のALBのUnHealthyHostCountにチェックを入れて`メトリクスの選択`に進みます。
![](/images/alb-cloudwatch-lambda/image17.png)
　　
1. 下記画像のように設定します。
条件は、動作テストしやすいようにするため、最初は1分以内に1データポイントにしています。
![](/images/alb-cloudwatch-lambda/image18.png)
![](/images/alb-cloudwatch-lambda/image19.png)
　　
1. 次のページ、アクションの設定では、先ほど作成したLambdaを選択します。
![](/images/alb-cloudwatch-lambda/image20.png)
　　
1. 次のページでは、アラームに任意の名前を設定します。
次のプレビュー画面で入力内容が確認できたら作成完了です！
　　
1. 動作テスト
試しに本番環境EC2アプリ内でDockerを停止します。
　　
1. ALB側で異常を検知していることを確認します。
その後、CloudWatchのアラームが`アラーム状態`になることを確認します。
アラーム状態が続くと設定したSlackチャンネルにメッセージが届くことを確認します。
![](/images/alb-cloudwatch-lambda/image21.png)
　　
1. OKだったらCloudWatchアラームの条件を、5分以内に5回など、任意の条件に設定し直します。これで終了です！🎉


## まとめ
サーバーダウン時に通知が来るようにという目的で、インフラの知識が全くない中技術記事を参考に実装してみました。
S3位しか触ったことがなく、苦手意識のあったAWSでしたが、今回を機に調べたり触れたことで苦手意識が少しだけ薄れました🥺
理解はまだまだ浅いため、これからも学習を続けていきます！

　
## 参考記事
- [Amazon CloudWatch のメトリクスとアラームについて](https://zenn.dev/amarelo_n24/articles/5aaab63a85a9d0)
- [AWS LambdaでSlack通知してみる](https://qiita.com/suo-takefumi/items/b47922362366de897920)
- [RubyとAWS LambdaでサクッとSlackへポストしてみる](https://qiita.com/mashvalue1/items/34304d82cf9ae2538c54#slack%E3%81%AEincoming-webhook%E3%81%AE%E8%A8%AD%E5%AE%9A)
- [[アップデート] Amazon CloudWatch のアラームで、実行アクションに Lambda 関数を直接指定出来るようになりました](https://dev.classmethod.jp/articles/cloudwatch-alarms-lambda-change-action/)
- [ELBのヘルスチェック失敗をCloudWatchアラームで監視してメール通知してみた](https://dev.classmethod.jp/articles/elb-healthcheck-monitoring-by-cloudwatch-alarm/)
