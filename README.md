# CI環境
## 1. ECRの作成
TBD
## 2. CodeBuildのプロジェクト作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codebuild.yaml
aws cloudformation create-stack --stack-name BFF-CodeBuild-Stack --template-body file://cfn-bff-codebuild.yaml --capabilities CAPABILITY_IAM
aws cloudformation validate-template --template-body file://cfn-backend-codebuild.yaml
aws cloudformation create-stack --stack-name Backend-CodeBuild-Stack --template-body file://cfn-backend-codebuild.yaml --capabilities CAPABILITY_IAM
```

# ECS環境
## 1. VPCおよびサブネット、InternetGateway等の作成
```sh
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation create-stack --stack-name ECS-VPC-Stack --template-body file://cfn-vpc.yaml
```

## 2. NAT Gatewayの作成とプライベートサブネットのルートテーブル更新
```sh
aws cloudformation validate-template --template-body file://cfn-ngw.yaml
aws cloudformation create-stack --stack-name ECS-NATGW-Stack --template-body file://cfn-ngw.yaml
```

## 3. Security Groupの作成
```sh
aws cloudformation validate-template --template-body file://cfn-sg.yaml
aws cloudformation create-stack --stack-name ECS-SG-Stack --template-body file://cfn-sg.yaml
```
## 4. IAMの作成
```sh
aws cloudformation validate-template --template-body file://cfn-iam.yaml
aws cloudformation create-stack --stack-name ECS-IAM-Stack --template-body file://cfn-iam.yaml --capabilities CAPABILITY_IAM
```

## 5. ALBの作成
* ALB
```sh
aws cloudformation validate-template --template-body file://cfn-alb.yaml
aws cloudformation create-stack --stack-name ECS-ALB-Stack --template-body file://cfn-alb.yaml
```
* ECS上のアプリ用のTarget Group, Listener
```sh
aws cloudformation validate-template --template-body file://cfn-tg.yaml
aws cloudformation create-stack --stack-name ECS-TG-Stack --template-body file://cfn-tg.yaml
```

## 6. ECSクラスタの作成
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-cluster.yaml
aws cloudformation create-stack --stack-name ECS-CLUSTER-Stack --template-body file://cfn-ecs-cluster.yaml
```

## 7. ECSタスク定義の作成
* タスク定義
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-task.yaml
aws cloudformation create-stack --stack-name ECS-TASK-Stack --template-body file://cfn-ecs-task.yaml
```
* タスクへのIAMロールの付与
  * TBD: RDS,DynamoDBへのアクセスの必要に応じて
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-task-role.yaml
aws cloudformation create-stack --stack-name ECS-TASK-ROLE-Stack --template-body file://cfn-ecs-task-role.yaml
```

## 8. ECSサービスの実行
```sh
aws cloudformation validate-template --template-body file://cfn-ecs-service.yaml
aws cloudformation create-stack --stack-name ECS-SERVICE-Stack --template-body file://cfn-ecs-service.yaml
```

## 9. APの実行確認
* ブラウザで「http://(Public ALBのDNS名)/backend-for-frontend/index.html」を入力するとフロントエンドAPの画面表示
* VPCのパブリックサブネット上にEC2を起動し
　「curl 「http://(Private ALBのDNS名)/backend/api/v1/users」を入力するとバックエンドサービスAPのJSON返却

# CD環境
## 1. CodePipelineの作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codepipeline.yaml
aws cloudformation create-stack --stack-name Bff-CodePipeline-Stack --template-body file://cfn-bff-codepipeline.yaml --capabilities CAPABILITY_IAM

aws cloudformation validate-template --template-body file://cfn-backend-codepipeline.yaml
aws cloudformation create-stack --stack-name Backend-CodePipeline-Stack --template-body file://cfn-backend-codepipeline.yaml --capabilities CAPABILITY_IAM
```

## 2. BFFアプリケーションのapplication.yamlの変更
* 現状、application.yamlのservice.dnsのurl値を、CloudFormationで作成した内部ALBのURLに変更しCodeCommitにプッシュしてください
  * 環境変数から読み込む対応をできていないためです。このため、APでエラーが発生すると思います。
  * プッシュすると、CodePipelineのパイプラインが起動し最新のDockerイメージでECRへデプロイしてくれます
  * その後、APが正しく動きます。


# CloudFormationコマンド文法メモ
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
