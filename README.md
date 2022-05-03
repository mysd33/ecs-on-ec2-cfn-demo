# SpringBoot APをECS on EC2で動作させCode系でCI/CDするCloudFormationサンプルテンプレート

## 構成
* CDは標準のローリングアップデートとBlueGreenデプロイメントの両方に対応しており、以下のいずれか２つの構成が構築できる。
  * システム構成図　ローリングアップデート版
![システム構成図ローリングアップデート版](img/ecs-rolling-update.png)
  * システム構成図　BlueGreenデプロメント版
![システム構成図BlueGreenデプロイメント版](img/ecs-bluegreen-deployment.png)

* メトリックスのモニタリング
  * CloudWatch Container Insightsは有効化し、各メトリックスを可視化。
* ログの転送
  * 現状、awslogsドライバを使ったCloudWatch Logsへの転送に対応。
  * TBD: FireLensに対応したサンプルも追加検討中。
![ログドライバ](img/logdriver.png)

* オートスケーリング
  * 平均CPU使用率のターゲット追跡スケーリングポリシーによる例に対応している。
![オートスケーリング](img/autoscaling.png)
  * TBD: AutoScalingのカスタムキャパシティプロバイダによるクラスタオートスケーリングのサンプル追加検討中。

## 事前準備
* CodePipeline、CodeBuildのArtifact用のS3バケットを作成しておく
  * 後続の手順で、バケット名を変更するパラメータがあるところで指定
## IAM
### 1. IAMの作成
```sh
aws cloudformation validate-template --template-body file://cfn-iam.yaml
aws cloudformation create-stack --stack-name ECS-IAM-Stack --template-body file://cfn-iam.yaml --capabilities CAPABILITY_IAM
```
* CodePipeline、CodeBuildのArtifact用のS3バケット名を変えるには、それぞれのcnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」

* TBD
  * IAMポリシーの記載は精査中
  * RDB対応時は、タスクへのIAMロールの付与のため修正が必要

## CI環境
## 1. アプリケーションのCodeCommit環境
* 別途、以下の2つのSpringBootAPのプロジェクトが以下のリポジトリ名でCodeCommitにある前提
  * backend-for-frontend
    * BFFのAP
  * backend
    * BackendのAP

### 2. ECRの作成
```sh
aws cloudformation validate-template --template-body file://cfn-ecr.yaml
aws cloudformation create-stack --stack-name ECR-Stack --template-body file://cfn-ecr.yaml
```
### 3. CodeBuildのプロジェクト作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codebuild.yaml
aws cloudformation create-stack --stack-name BFF-CodeBuild-Stack --template-body file://cfn-bff-codebuild.yaml
aws cloudformation validate-template --template-body file://cfn-backend-codebuild.yaml
aws cloudformation create-stack --stack-name Backend-CodeBuild-Stack --template-body file://cfn-backend-codebuild.yaml
```
* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」

* 本当は、CloudFormationテンプレートのCodeBuildのSorceTypeをCodePipelineにするが、いったんDockerイメージ作成して動作確認したいので、今はCodeCommitになっている。動いてはいるので保留。

* TBD: Mavenのカスタムローカルキャッシュによるビルド時間短縮がうまく動いていない
  * ひょっとしたら、ローカルキャッシュではなくS3でないとうまくいかない？
    * https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/build-caching.html  
    * https://aws.amazon.com/jp/blogs/devops/how-to-enable-caching-for-aws-codebuild/

### 3. ECRへ最初のDockerイメージをプッシュ
2つのCodeBuildプロジェクトが作成されるので、それぞれビルド実行し、ECRにDockerイメージをプッシュさせる。

## ネットワーク環境
### 1. VPCおよびサブネット、Publicサブネット向けInternetGateway等の作成
```sh
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation create-stack --stack-name ECS-VPC-Stack --template-body file://cfn-vpc.yaml
```
### 2. Security Groupの作成
```sh
aws cloudformation validate-template --template-body file://cfn-sg.yaml
aws cloudformation create-stack --stack-name ECS-SG-Stack --template-body file://cfn-sg.yaml
```
* 必要に応じて、端末の接続元IPアドレス等のパラメータを指定
    * 「--parameters ParameterKey=TerminalCidrIP,ParameterValue=X.X.X.X/X」

### 3. VPC Endpointの作成とプライベートサブネットのルートテーブル更新
```sh
aws cloudformation validate-template --template-body file://cfn-vpe.yaml
aws cloudformation create-stack --stack-name ECS-VPE-Stack --template-body file://cfn-vpe.yaml
```
### 4.（作成任意）NAT Gatewayの作成とプライベートサブネットのルートテーブル更新
* 本手順では、ECRのイメージ転送量等にかかるNAT Gatewayのコストが全体の割合からすると大きく節約のためVPC Endpointを作成するので、NAT Gatewayは通常不要。
* 本手順をカスタマイズして、VPC Endpoint未作成のリソースアクセスに使用したい場合に以下を追加実行すればよい。
```sh
aws cloudformation validate-template --template-body file://cfn-ngw.yaml
aws cloudformation create-stack --stack-name ECS-NATGW-Stack --template-body file://cfn-ngw.yaml
```

## DB環境
* TBD:　今後Aurora等のRDBリソースのサンプル作成を検討

## ECS環境
### 1. ALBの作成
* ECSの前方で動作するALBとデフォルトのTarget Group等を作成
```sh
aws cloudformation validate-template --template-body file://cfn-alb.yaml
aws cloudformation create-stack --stack-name ECS-ALB-Stack --template-body file://cfn-alb.yaml
```
* BlueGreenデプロイメントの場合のみ以下実行し、2つ目（Green環境）用のTarget Groupを作成
```sh
aws cloudformation validate-template --template-body file://cfn-tg-bg.yaml
aws cloudformation create-stack --stack-name ECS-TG-BG-Stack --template-body file://cfn-tg-bg.yaml
```
* TBD: バックエンドサービスをALBではなく、CloudMapによるサービスディスカバリを使ったサンプル作成を検討

* ~~手順削除：ListenerRuleを使ったTargetGroupの設定~~
~~aws cloudformation validate-template --template-body file://cfn-tg.yaml~~
~~aws cloudformation create-stack --stack-name ECS-TG-Stack --template-body file://cfn-tg.yaml~~

### 2. ECSクラスタの作成
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-cluster.yaml
aws cloudformation create-stack --stack-name ECS-CLUSTER-Stack --template-body file://cfn-ecs-cluster.yaml
```
* 必要に応じてキーペア名のパラメータ値を修正して使用
  * 「Mappings:」の「FrontendClusterDefinitionMap:」の「KeyPairName:」
  * 「Mappings:」の「BackendClusterDefinitionMap:」の「KeyPairName:」  
    * 「myKeyPair」となっているところを自分のキーペア名に修正
### 3. ECSタスク定義の作成
* タスク定義
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-task.yaml
aws cloudformation create-stack --stack-name ECS-TASK-Stack --template-body file://cfn-ecs-task.yaml
```

### 4-1. ECSサービスの実行（ローリングアップデートの場合）
* ローリングアップデートの場合は以下を実行
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-service.yaml
aws cloudformation create-stack --stack-name ECS-SERVICE-Stack --template-body file://cfn-ecs-service.yaml
```
* パラメータMinimumHealthyPercentを0%にしていて、1,2分気持ちローリングアップデートの時間が短くなるようになっている
### 8-2. ECSサービスの実行（BlueGreenデプロイメントの場合）
* BlueGreenデプロイメントの場合は以下のパラメータを指定して起動
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-service.yaml
aws cloudformation create-stack --stack-name ECS-SERVICE-Stack --template-body file://cfn-ecs-service.yaml --parameters ParameterKey=DeployType,ParameterValue=CODE_DEPLOY
```

### 9. APの実行確認
* Backendアプリケーションの確認  
  * VPCのパブリックサブネット上にBationのEC2を起動
```sh
aws cloudformation validate-template --template-body file://cfn-bastion-ec2.yaml
aws cloudformation create-stack --stack-name Demo-Bastion-Stack --template-body file://cfn-bastion-ec2.yaml
```
  * 必要に応じてキーペア名等のパラメータを指定
    * 「--parameters ParameterKey=KeyPairName,ParameterValue=myKeyPair」
  * BastionのEC2のアドレスは、CloudFormationの「ECS-Bastion-Stack」スタックの出力「BastionDNSName」のURLを参照    
  * EC2にSSHでログインし、以下のコマンドを「curl http://(Private ALBのDNS名)/backend/api/v1/users」を入力するとバックエンドサービスAPのJSONレスポンスが返却
    * CloudFormationの「ECS-SERVICE-Stack」スタックの出力「BackendServiceURI」のURLを参照

* BFFアプリケーションの確認
  * ブラウザで「http://(Public ALBのDNS名)/backend-for-frontend/index.html」を入力しフロントエンドAPの画面が表示される
    * CloudFormationの「ECS-SERVICE-Stack」スタックの出力「FrontendWebAppServiceURI」のURLを参照

* うまく動作しない場合、Cloud Watch Logの以下のロググループのAPログにエラーが出ていないか確認するとよい
  * /ecs/logs/Demo-backend-ecs-group
  * /ecs/logs/Demo-bff-ecs-group

### 10. Application AutoScalingの設定
* 以下のコマンドで、ターゲット追跡スケーリングポリシーでオートスケーリング設定
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-autoscaling.yaml
aws cloudformation create-stack --stack-name ECS-AutoScaling-Stack --template-body file://cfn-ecs-autoscaling.yaml
```

* BastionのEC2から、ApacheBench (ab) ユーティリティを使用して、ロードバランサーに短期間に大量のHTTPリクエストを送信
  * abコマンドのインストール
```sh
sudo yum install httpd-tools
```
  * 以下のいずれかのabコマンドを実行
```sh
ab -n 1000000 -c 1000 http://(Private ALBのDNS名)/backend/api/v1/users
```
```sh
ab -n 1000000 -c 1000 http://(Public ALBのDNS名).ap-northeast-1.elb.
amazonaws.com/backend-for-frontend/index.html
```
* 対象のECSサービスに関するCPU使用率に関するCloudWatchアラームが出ていることを確認
* 対象のECSサービスがスケールアウトされ、1タスク追加され2タスクになっていることを確認
* abコマンドが終了し、しばらくたつと、対象のECSサービスがスケールインされ、1タスクに戻っていることを確認
## CD環境（標準のローリングアップデートの場合）
* ローリングアップデートの場合は、以下のコマンドを実行
### 1. ローリングアップデート対応のCodePipelineの作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codepipeline.yaml
aws cloudformation create-stack --stack-name Bff-CodePipeline-Stack --template-body file://cfn-bff-codepipeline.yaml

aws cloudformation validate-template --template-body file://cfn-backend-codepipeline.yaml
aws cloudformation create-stack --stack-name Backend-CodePipeline-Stack --template-body file://cfn-backend-codepipeline.yaml
```

* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」
### 2. CodePipelineの確認
  * CodePipelineの作成後、パイプラインが自動実行されるので、デプロイ成功することを確認する
### 3. ソースコードの変更
  * 何らかのソースコードの変更を加えて、CodeCommitにプッシュする
  * CodePipelineのパイプラインが実行され、新しいAPがデプロイされることを確認する

## CD環境（標準のBlueGreenデプロイメントの場合）
* BlueGreenデプロイメントの場合は、以下のコマンドを実行
### 1. CodeDeployの作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codedeploy.yaml
aws cloudformation create-stack --stack-name Bff-CodeDeploy-Stack --template-body file://cfn-bff-codedeploy.yaml

aws cloudformation validate-template --template-body file://cfn-backend-codedeploy.yaml
aws cloudformation create-stack --stack-name Backend-CodeDeploy-Stack --template-body file://cfn-backend-codedeploy.yaml
```

* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」  
### 2. BlueGreenデプロイメント対応のCodePipelineの作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codepipeline-bg.yaml
aws cloudformation create-stack --stack-name Bff-CodePipeline-BG-Stack --template-body file://cfn-bff-codepipeline-bg.yaml

aws cloudformation validate-template --template-body file://cfn-backend-codepipeline-bg.yaml
aws cloudformation create-stack --stack-name Backend-CodePipeline-BG-Stack --template-body file://cfn-backend-codepipeline-bg.yaml
```

* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」
### 3. CodePipelineの確認
  * CodePipelineの作成後、パイプラインが自動実行されるので、デプロイ成功することを確認する
### 4. ソースコードの変更
  * 何らかのソースコードの変更を加えて、CodeCommitにプッシュする
  * CodePipelineのパイプラインが実行され、新しいAPがデプロイされることを確認する

## （参考）CloudFormationコマンド文法メモ
* スタックの新規作成
```sh
aws cloudformation create-stack --stack-name myteststack --template-body file://cfn-ec2.yaml
```
* スタックの新規作成(削除保護設定あり)
```sh
aws cloudformation create-stack --stack-name myteststack --enable-termination-protection --template-body file://cfn-ec2.yaml
```
* スタックの更新
```sh
aws cloudformation update-stack --stack-name myteststack --template-body file://cfn-vpc.yaml
```
* テンプレートの検証
```sh
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
```
* スタックの削除保護設定解除
```sh
aws cloudformation update-termination-protection --stack-name myteststack --no-enable-termination-protection
```
* スタックの削除
```sh
aws cloudformation delete-stack --stack-name myteststack
```
* IAMロールのスタックの場合 --capabilities CAPABILITY_IAM（またはCAPABILITY_NAMED_IAM）つける
```sh
aws cloudformation create-stack --stack-name my-iamgroup-stack --template-body file://cfn-iamgroup.yaml --capabilities CAPABILITY_IAM
```
