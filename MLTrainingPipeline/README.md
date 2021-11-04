# S3をトリガーとした、StepFunctionsを使ったパイプライン

## 概要

IoTデバイスから上がってくるデータを処理するためのパイプラインや、機械学習のトレーニングを実行するためのパイプラインが必要となった際に、S3にデータが上がってくることをイベントとして、Step Functionsを実行できるサンプルです。IoTの要件によっては、IoT Core経由ではなく、既存設備からCSVを取り出してS3にアップすることもあるので、その様なユースケースでも利用できます。

機械学習用のパイプラインを検討する場合は、AWS Step Functions Data Science SDK を使ったサンプルもありますので、こちらを参考にしてください。
https://github.com/ohbuchim/sample-codes-for-aiml/blob/main/mlops/step-functions-model-train-evaluate-pipeline/step_functions_mlworkflow_scikit_learn_data_processing_and_model_evaluation.ipynb

## 動きの解説

とあるS3のバケットに必要なデータが保存され、全てが揃った後に`metainfo.json`というファイルをPutしてください(S3のイベントは特定の名前のオブジェクトがPutされると実行される様になっています)。
S3のフォルダ構成は以下の様に、同一のフォルダに異なるパイプラインで処理したいファイルを入れてしまうと、今回使うものといった管理が必要となってくるので、処理の単位でオブジェクトのキーを変えるのがいいでしょう。

例)
```
バケット
  /デバイス
    /学習の単位
      /この階層になんらかのデータや、画像ファイル、アノテーションデータ、`metainfo.json` を保存する
```

1. S3のバケットに`metainfo.json`という名前のファイルがPutされると、Lambdaが起動します。このLambdaは進捗と履歴管理のために、DynamoDBにファイルのオブジェクトキーなどが保存され、StepFunctionsのステートマシンが実行されます。
1. ステートマシンでは、最初にEC2インスタンスを起動し、DynamoDBのステータスを変更するLambdaを実行します。EC2期同時に実行スクリプトなどはこのLambdaでuser_dataとして`ec2.run_instances`に渡します。
  1. インスタンスサイズはt2.microとなっています。変更する場合は`InstanceType="t2.micro"`を修正します
  1. Amazon Linux2のAMIを利用しています。変更する場合は`ImageId="ami-0fe22bffdec36361c"`を修正します
  1. EBSのサイズを変更する場合は`BlockDeviceMappings=[{'Ebs':{'VolumeSize':20},"DeviceName" : '/dev/sda1'}],`を修正します
1. 起動されたEC2インスタンスで機械学習のトレーニングの様な固有の処理を実行し、結果をDynamoDBに反映してください(この処理は含まれていません)
  1. EC2インスタンスにはAWS Systems Managerを使ってSSHすることが可能になっています。
  1. https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html#start-sys-console
1. ステートマシンではDynamoDBのステータスをチェックするLambdaを定期的に実行し、完了ステータスになるまで待ちます
1. 完了ステータスになると、LambdaはEC2インスタンスを終了し、SNSを介して、データを通知します

### 注意

ステートマシンが正常に終了しないとEC2インスタンスは落とされませんので、一定時間以上起動していいるインスタンスがいれば削除する様な処理を別途用意するのがいいでしょう

## セットアップ

1. EC2インスタンスを起動するサブネットのIDを確認します。EC2から外部へアクセスする必要があるのでパブリックサブネットが必要です
2. CloudFormationから新しくスタックを作成します。
    1. https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/GettingStarted.Walkthrough.html
      * S3バケットの名前を指定します
      * サブネットIDを指定します

## 動作確認

1. CloudFormationのスタックを作った時に指定したS3のバケットにmetainfo.jsonという名前のファイルを置きます。すると、Lambdaが実行されStepFunctionsのステートマシンが開始されます
2. Worker Instanceという名前のEC2インスタンスが立ち上がります
    1. 実際は、このEC2インスタンス上でなんらかの処理が行われ、DynamoDBに結果が書き込まれます
    2. 動作確認では手動でDynamoDBの対象のレコードの*JobStatus*を*WorkerFinished*に変更します
3. ステータスが変更されると、StepFunctionの処理が進み、EC2インスタンスを停止してステートマシンが終了します。


## 補足情報

- S3のイベント
  - https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/with-s3-example.html
- DynamoDBの操作
  - https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/GettingStarted.Python.03.html
