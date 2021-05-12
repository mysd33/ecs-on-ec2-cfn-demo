## 1. VPCおよびサブネット、InternetGateway等の作成
```sh
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation create-stack --stack-name ECS-VPC-Stack --template-body file://cfn-vpc.yaml
aws cloudformation update-stack --stack-name ECS-VPC-Stack --template-body file://cfn-vpc.yaml
```

## 2. NAT Gatewayの作成とプライベートサブネットのルートテーブル更新
```sh
aws cloudformation validate-template --template-body file://cfn-ngw.yaml
aws cloudformation create-stack --stack-name ECS-NATGW-Stack --template-body file://cfn-ngw.yaml
aws cloudformation update-stack --stack-name ECS-NATGW-Stack --template-body file://cfn-ngw.yaml
```

## 3. Security Groupの作成
```sh
aws cloudformation validate-template --template-body file://cfn-sg.yaml
aws cloudformation create-stack --stack-name ECS-SG-Stack --template-body file://cfn-sg.yaml
aws cloudformation update-stack --stack-name ECS-SG-Stack --template-body file://cfn-sg.yaml
```
## 4. IAMの作成
## 5. ALBの作成
* ALB
```sh
aws cloudformation validate-template --template-body file://cfn-alb.yaml
aws cloudformation create-stack --stack-name ECS-ALB-Stack --template-body file://cfn-alb.yaml
aws cloudformation update-stack --stack-name ECS-ALB-Stack --template-body file://cfn-alb.yaml
```
* Target Group
```sh
aws cloudformation validate-template --template-body file://cfn-tg.yaml
aws cloudformation create-stack --stack-name ECS-TG-Stack --template-body file://cfn-tg.yaml
aws cloudformation update-stack --stack-name ECS-TG-Stack --template-body file://cfn-tg.yaml
```
## 6. ECSクラスタの作成

## 7. ECSタスク定義の作成

## 8. ECSサービスの実行
## 8. ECS用のTargetGroup、Listenerの作成


## CloudFormationコマンド文法メモ
aws cloudformation create-stack --stack-name myteststack --enable-termination-protection --template-body file://cfn-ec2.yaml
aws cloudformation create-stack --stack-name myteststack --template-body file://cfn-ec2.yaml
aws cloudformation update-stack --stack-name myteststack --template-body file://cfn-vpc.yaml
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation update-termination-protection --stack-name myteststack --no-enable-termination-protection
aws cloudformation delete-stack --stack-name myteststack
aws cloudformation create-stack --stack-name my-iamgroup-stack --template-body file://cfn-iamgroup.yaml --capabilities CAPABILITY_NAMED_IAM