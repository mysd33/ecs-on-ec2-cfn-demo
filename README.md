## 1. VPCの作成
```sh
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation create-stack --stack-name ECSVPCStack --template-body file://cfn-vpc.yaml
aws cloudformation update-stack --stack-name ECSVPCStack --template-body file://cfn-vpc.yaml








## 文法メモ
aws cloudformation create-stack --stack-name myteststack --enable-termination-protection --template-body file://cfn-ec2.yaml
aws cloudformation create-stack --stack-name myteststack --template-body file://cfn-ec2.yaml
aws cloudformation update-stack --stack-name myteststack --template-body file://cfn-vpc.yaml
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation update-termination-protection --stack-name myteststack --no-enable-termination-protection
aws cloudformation delete-stack --stack-name myteststack
aws cloudformation create-stack --stack-name my-iamgroup-stack --template-body file://cfn-iamgroup.yaml --capabilities CAPABILITY_NAMED_IAM