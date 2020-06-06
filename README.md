# Amazon ECSの高セキュリティ環境検証
ECSを、VPC Endpointを利用したインターネット接続のない環境で利用する検証。

# 作成環境
<img src="./Documents/arch.png" whdth=500>

* ECS
    * クラスター名: <code>ecs-cluster-01</code>
    * Provider名: <code>Provider-ecs-autoscaling-group</code>
    * タスク定義名: <code>ecs-task-def</code>
    * サービス名: <code>ecs-service</code>
* Worker
    * AutoScalingGroup名: <code>ecs-autoscaling-group</code>
    * 起動テンプレート名: <code>ecs-worker-ec2-tamplate</code>
* ALB
    * ALB名: <code>ecs-front-balancer</code>
    * Target名: <code>ecs-target</code>
# 作成手順
## (1)事前設定
### (1)-(a) 作業環境の準備
下記を準備します。
* bashが利用可能な環境(LinuxやMacの環境)
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定

### (1)-(b) gitのclone
```shell
git clone https://github.com/Noppy/ECS-Security-PoC.git
cd ECS-Security-PoC
```

### (1)-(c) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<設定したプロファイル名称を指定。デフォルトの場合はdefaultを設定>
export REGION=$(aws --profile ${PROFILE} configure get region)

echo "REGION = ${REGION}"
```

## (2)VPCの作成(CloudFormation利用)
ECSの管理捜査用のインスタンスを稼働するVPCと、ECSのコンテナ(データプレーン)を稼働させるVPCの２つを作成します。

### (2)-(a) Management-VPC作成
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "ManagementVPC"
  },
  {
    "ParameterKey": "VpcInternalDnsName",
    "ParameterValue": "ecs-mgr.local."
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name EcsMgr-VPC \
    --template-body "file://./CloudFormation/vpc-2subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```

### (2)-(b) ECS-VPC作成
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "EcsWorkerVPC"
  },
  {
    "ParameterKey": "VpcInternalDnsName",
    "ParameterValue": "ecs-worker.local."
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name EcsWorker-VPC \
    --template-body "file://./CloudFormation/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```

## (3) Security Group設定

### (3)-(a) Management-VPC Security Groups設定
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name EcsMgr-VPC-SecurityGroup \
    --template-body "file://./CloudFormation/EcsMgrVPC-SG.yaml" ;
```

### (3)-(b) ECS-VPC Security Groups設定
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name EcsWorker-VPC-SecurityGroup \
    --template-body "file://./CloudFormation/EcsWorkerVPC-SG.yaml" ;

#スタック更新時
aws --profile ${PROFILE} cloudformation update-stack \
    --stack-name EcsWorker-VPC-SecurityGroup \
    --template-body "file://./CloudFormation/EcsWorkerVPC-SG.yaml" ;
```
## (4) VPC Endpoint設定
ECSWorker-VPCに、ECS、ECR(API, Docker)、Logs、S3のVPC Endpointを作成します。Endpoint policyはここでは設定せず、リソース作成後にCLIで設定します。
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name EcsWorker-VPC-Vpce \
    --template-body "file://./CloudFormation/EcsWorkerVPC-Vpce.yaml" ;
```

## (5) IAMロール作成
ここでは、ECS用に複数のIAMロールを作成します。またBasionからECRにイメージを登録するためBasion用のIAMインスタンスロールも作成します。
* ECS関連
<table>
<tr><th colspan=2>Class</th><th>IAM Role</th><th>principal</th><th>Policies summary</th><th>Remark</th></tr>
<tr><td colspan=2>Admin User</td><td>EC2-EcsManagerRole</td><td>ec2</td><td><lu><li>ecs read/write</li><li>ec2(vpc)read</li><li>ec2(ec2)read/write</li><li>iam read</li></lu><li>ECS FullAccess</li><li>Describe other resource</li><li>IAM PassRole</li></td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security_iam_id-based-policy-examples.html">ECS Developer Guide</a></td></tr>
<tr><td rowspan=6>ECS</td><td>Service Linked</td><td>AWSServiceRoleForECS
</td><td>ecs<br>(Service Linked)</td><td>(AWS managed)<br>AmazonECSServiceRolePolicy</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using-service-linked-roles.html">Service-Linked Role for Amazon ECS</a></td></tr>
<tr><td>[ELB]<br>ServiceRole for ECS</td><td>ecsServiceRole</td><td>-</td><td>-</td><td>Service-Linkedに統合されたため不要</td></tr>
<tr><td>Service AutoScaling</td><td>ecsAutoscaleRole</td><td>-</td><td>-</td><td>Service-Linkedに統合されたため不要</td></tr>
<tr><td>Worker(EC2)</td><td>AmazonEC2ContainerServiceforEC2Role</td><td>ec2</td><td>(AWS managed)<br>AmazonEC2ContainerServiceforEC2Role</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html">Amazon ECS Container Instance IAM Role</a></td></tr>
<tr><td>[Task]<br>ExecutionRole</td><td>ecsTaskExecutionRole</td><td>ecs-tasks</td><td>(AWS managed)<br>AmazonECSTaskExecutionRolePolicy</td><td><a href="https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task_execution_IAM_role.html">Amazon ECS Task Execution IAM Role</a>(for pulling ecr and putting logs)</td></tr>
<tr><td>[Task]<br>Task Role</td><td>今回は作成しない<br>(コンテナ内からAWS APIを実行する場合必要)</td><td>-</td><td>-</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html">IAM Roles for Tasks</a>(for pulling ecr and putting logs)</td></tr>
<tr><td colspan=2>Application Autscaling<br>for ECS</td><td>AWSServiceRoleForApplicationAutoScaling_ECSService</td><td>ecs.application-autoscaling</td><td>(AWS managed)<br>AWSApplicationAutoscalingECSServicePolicy</td><td><a href="https://docs.aws.amazon.com/en_us/autoscaling/application/userguide/application-auto-scaling-service-linked-roles.html">Service-Linked Roles for Application Auto Scaling</a></td></tr>
<tr><td colspan=2>CloudWatch Events</td><td>AmazonEC2ContainerServiceEventsRole</td><td>events</td><td>(AWS managed)<br>AmazonEC2ContainerServiceEventsRole</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/CWE_IAM_role.html">ECS CloudWatch Events IAM Role</a></td></tr>
</table>

* ECR操作用
<table>
<tr><th>Class</th><th>IAM Role</th><th>principal</th><th>Policies summary</th><th>Remark</th></tr>
<tr><td>Basion<br>ECR操作用</td><td>EC2-BastionRole</td><td>ec2</td><td>AmazonEC2ContainerRegistryPowerUser</td><td></td></tr>
</table>


### (5)-(a) ECS管理者用IAMロール
管理用のLinux-EC2インスタンスに付与する、インスタンスロールを作成します。
```shell
#事前設定
ACCOUNT_ID=$(aws --profile ${PROFILE} --output text \
    sts get-caller-identity --query 'Account')


# カスタマー管理ポリシー - 自分のアクセス許可の表示を許可
IamPolicyName='ECS-AllowUsersToViewTheirOwnIamPermissions'
IamDescription='Allow Users to View Their Own Permissions.'
IamPath='/ecs/'
CustomerPolicyDocument='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ViewOwnUserInfo",
      "Effect": "Allow",
      "Action": [
        "iam:GetUserPolicy",
        "iam:ListGroupsForUser",
        "iam:ListAttachedUserPolicies",
        "iam:ListUserPolicies",
        "iam:GetUser"
      ],
      "Resource": [
        "arn:aws:iam::*:user/${aws:username}"
      ]
    },
    {
      "Sid": "NavigateInConsole",
      "Effect": "Allow",
      "Action": [
        "iam:GetGroupPolicy",
        "iam:GetPolicyVersion",
        "iam:GetPolicy",
        "iam:ListAttachedGroupPolicies",
        "iam:ListGroupPolicies",
        "iam:ListPolicyVersions",
        "iam:ListPolicies",
        "iam:ListUsers"
      ],
      "Resource": "*"
    }
  ]
}'
#カスタマー管理ポリシー作成
aws --profile ${PROFILE} \
    iam create-policy \
        --policy-name "${IamPolicyName}" \
        --path "${IamPath}" \
        --policy-document "${CustomerPolicyDocument}" ;

# カスタマー管理ポリシー - ECSのFullAccess
IamPolicyName='ECS-FullAccess'
IamDescription='Administrative access to Amazon ECS resources.'
IamPath='/ecs/'
CustomerPolicyDocument='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEcs",
      "Effect": "Allow",
      "Action": [
        "ecs:*"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "ForSettingEcsProvider",
      "Effect": "Allow",
      "Action": [
        "autoscaling:CreateOrUpdateTags"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "ForSettingECSClusterTaskDefService",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "DescribeOtherServiceResources",
      "Effect": "Allow",
      "Action": [
        "application-autoscaling:Describe*",
        "autoscaling:Describe*",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:GetMetricStatistics",
        "sns:ListTopics",
        "lambda:ListFunctions",
        "ec2:Describe*",
        "elasticloadbalancing:Describe*",
        "events:List*",
        "logs:DescribeLogGroups",
        "route53:ListHostedZonesByName",
        "route53:GetHealthCheck",
        "servicediscovery:List*",
        "servicediscovery:Get*",
        "ecr:Describe*",
        "iam:Get*"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Action": "iam:PassRole",
      "Effect": "Allow",
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringLike": {
            "iam:PassedToService": [
                "ecs-tasks.amazonaws.com",
                "ecs.amazonaws.com"
            ]
        }
      }
    },
    {
      "Action": "iam:PassRole",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:iam::'"${ACCOUNT_ID}"':role/ecsInstanceRole*"
      ],
      "Condition": {
        "StringLike": {
          "iam:PassedToService": [
            "ec2.amazonaws.com"
          ]
        }
      }
    },
    {
      "Action": "iam:PassRole",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:iam::'"${ACCOUNT_ID}"':role/ecsAutoscaleRole*"
      ],
      "Condition": {
        "StringLike": {
          "iam:PassedToService": [
            "application-autoscaling.amazonaws.com"
          ]
        }
      }
    }
  ]
}'
#カスタマー管理ポリシー作成
aws --profile ${PROFILE} \
    iam create-policy \
        --policy-name "${IamPolicyName}" \
        --path "${IamPath}" \
        --policy-document "${CustomerPolicyDocument}" ;


#IAMロール作成
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "EC2-EcsManagerRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200


# カスタマー管理ポリシーのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "EC2-EcsManagerRole" \
        --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecs/ECS-AllowUsersToViewTheirOwnIamPermissions

aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "EC2-EcsManagerRole" \
        --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/ecs/ECS-FullAccess

#インスタンスプロファイルの作成
aws --profile ${PROFILE} \
    iam create-instance-profile \
        --instance-profile-name "EC2-EcsManagerRole-Profile";

aws --profile ${PROFILE} \
    iam add-role-to-instance-profile \
        --instance-profile-name "EC2-EcsManagerRole-Profile" \
        --role-name "EC2-EcsManagerRole" ;
```

### (5)-(b) ECS Service Linkd Role
ECSのサービスリンクを作成します。
```shell
#サービスロールの有無チェック
#このコマンドでロールが表示される場合は作成済みなのでスキップする
aws --profile ${PROFILE} \
    iam get-role --role-name AWSServiceRoleForECS

#ECSサービスロールの作成
aws --profile ${PROFILE} \
    iam create-service-linked-role \
        --aws-service-name ecs.amazonaws.com
```

### (5)-(c) Worker(EC2)用インスタンスロール
ECSのワーカーに設定するIAMロール(インスタンスロール)です。
```shell
#ecsServiceRoleロールの作成
POLICY='{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "AmazonEC2ContainerServiceforEC2Role" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200

# カスタマー管理ポリシーのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "AmazonEC2ContainerServiceforEC2Role" \
        --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

#インスタンスプロファイルの作成
aws --profile ${PROFILE} \
    iam create-instance-profile \
        --instance-profile-name "AmazonEC2ContainerServiceforEC2Role-Profile";

aws --profile ${PROFILE} \
    iam add-role-to-instance-profile \
        --instance-profile-name "AmazonEC2ContainerServiceforEC2Role-Profile" \
        --role-name "AmazonEC2ContainerServiceforEC2Role" ;
```

### (5)-(d) タスク実行 IAM ロール
Amazon ECS コンテナエージェントと Fargate タスクの Fargate エージェントが利用するIAMロール
```shell
#実行ロールの作成
POLICY='{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "ecsTaskExecutionRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200

# カスタマー管理ポリシーのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "ecsTaskExecutionRole" \
        --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### (5)-(e) Application Autoscaling for ECS用サービスロール
```shell
#サービスロールの有無チェック
#このコマンドでロールが表示される場合は作成済みなのでスキップする
aws --profile ${PROFILE} \
    iam get-role --role-name AWSServiceRoleForApplicationAutoScaling_ECSService

#ECSサービスロールの作成
aws --profile ${PROFILE} \
    iam create-service-linked-role \
        --aws-service-name ecs.application-autoscaling.amazonaws.com
```

### (5)-(f) CloudWatch イベント IAM ロール
Amazon ECS のスケジュールされたタスクを CloudWatch イベント のルールとターゲットで使用するには、Amazon ECS タスクを実行するためのアクセス許可が CloudWatch イベント サービスに必要。
#### (i) IAMロールの作成
```shell
#AmazonEC2ContainerServiceEventsRoleの作成
POLICY='{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "AmazonEC2ContainerServiceEventsRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200

# カスタマー管理ポリシーのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "AmazonEC2ContainerServiceEventsRole" \
        --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceEventsRole
```
#### (ii) タスク実行ロールのアクセス許可を CloudWatch イベント IAM ロールに追加
```shell
PolicyDocument='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::'"${ACCOUNT_ID}"':role/ecsTaskExecutionRole*"
      ]
    }
  ]
}'

aws --profile ${PROFILE} \
    iam put-role-policy \
        --role-name "AmazonEC2ContainerServiceEventsRole" \
        --policy-name "AmazonECSEventsTaskExecutionRole" \
        --policy-document "${PolicyDocument}" ;
```

### (5)-(g) Bastion Role
```shell
#IAMロール作成
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "EC2-BastionRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200


# カスタマー管理ポリシーのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "EC2-BastionRole" \
        --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

#インスタンスプロファイルの作成
aws --profile ${PROFILE} \
    iam create-instance-profile \
        --instance-profile-name "EC2-BastionRole-Profile";

aws --profile ${PROFILE} \
    iam add-role-to-instance-profile \
        --instance-profile-name "EC2-BastionRole-Profile" \
        --role-name "EC2-BastionRole" ;
```

## (6) ECS管理用/Bastionサーバ設置
### (6)-(a) 共通設定
```shell
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。  

#最新のAmazon Linux2のAMI IDを取得します。
AL2_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;

echo -e "KEYNAME=${KEYNAME}\nAL2_AMIID=${AL2_AMIID}"
```
### (6)-(b) ECS管理用インスタンス
#### (i) CloudFormationデータ取得
```shell
Subnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsMgr-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`Subnet1Id`].[OutputValue]')

SG_ID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsMgr-VPC-SecurityGroup \
        --query 'Stacks[].Outputs[?OutputKey==`MgrSGId`].[OutputValue]')

echo -e "Subnet1Id= $Subnet1Id\nSG_ID    = ${SG_ID}"
```
#### (ii) ECS管理用インスタンス作成
```shell
#インスタンスタイプ設定
INSTANCE_TYPE="t2.micro"

#タグ設定
TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "ECS-Manager"
            }
        ]
    }
]'

#ユーザデータ設定
USER_DATA='
#!/bin/bash -xe
                
yum -y update
yum -y install bind bind-utils
hostnamectl set-hostname ECS-Manager
'
# サーバの起動
aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${AL2_AMIID} \
        --instance-type ${INSTANCE_TYPE} \
        --key-name ${KEYNAME} \
        --subnet-id ${Subnet1Id} \
        --security-group-ids ${SG_ID} \
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" \
        --user-data "${USER_DATA}" \
        --iam-instance-profile "Name=EC2-EcsManagerRole-Profile"
```
### (6)-(c) (ECSWorkerVPC)Bastionインスタンス
#### (i) CloudFormationデータ取得
```shell
PublicSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet1Id`].[OutputValue]')

Bastion_SG_ID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC-SecurityGroup \
        --query 'Stacks[].Outputs[?OutputKey==`BastionSGId`].[OutputValue]')

echo -e "PublicSubnet1Id= $PublicSubnet1Id\nSG_ID    = ${Bastion_SG_ID}"
```
#### (ii) Basion用インスタンス作成
```shell
#インスタンスタイプ設定
INSTANCE_TYPE="t2.micro"

#タグ設定
TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "ECSWorker-Bastionr"
            }
        ]
    }
]'

#ユーザデータ設定
USER_DATA='
#!/bin/bash -xe
                
yum -y update
yum -y install bind bind-utils
hostnamectl set-hostname ECSWorker-Bastionr
'
# サーバの起動
aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${AL2_AMIID} \
        --instance-type ${INSTANCE_TYPE} \
        --key-name ${KEYNAME} \
        --subnet-id ${PublicSubnet1Id} \
        --security-group-ids ${Bastion_SG_ID} \
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" \
        --user-data "${USER_DATA}" \
        --iam-instance-profile "Name=EC2-BastionRole-Profile";
```

## (7) ALBの作成
ECSのコンテナのフロント用のALBを作成します。下記内容で作成します。
* ALB名: ecs-front-balancer
* ２つのPublic SubnetでMulti-AZ構成で作成
* ターゲット:
    * ターゲット名: ecs-target
    * protcol: HTTP
    * ヘルスチェックで、portは指定しない(その場合デフォルトのトラフィックポート利用となる)
* Listener
    * Port 80でListen
    * Protocol HTTP

## (7)-(a)データ設定
```shell
VpcId=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

PublicSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet1Id`].[OutputValue]')

PublicSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet2Id`].[OutputValue]')

ALB_SG_ID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC-SecurityGroup \
        --query 'Stacks[].Outputs[?OutputKey==`AlbSGId`].[OutputValue]')

echo -e "VpcId           = ${VpcId}\nPublicSubnet1Id = $PublicSubnet1Id\nPublicSubnet2Id = $PublicSubnet2Id\nALB_SG_ID       = ${ALB_SG_ID}"

```

## (7)-(b) ALB作成
```shell
#ALBの作成
aws --profile ${PROFILE} \
    elbv2 create-load-balancer \
        --name "ecs-front-balancer" \
        --scheme "internet-facing" \
        --subnets ${PublicSubnet1Id} ${PublicSubnet2Id}\
        --security-groups ${ALB_SG_ID}
#ALB ARNの取得
ALB_ARN=$(aws --profile ${PROFILE} --output text \
    elbv2 describe-load-balancers \
        --names ecs-front-balancer \
    --query 'LoadBalancers[].LoadBalancerArn' );
echo "ALB_ARN = ${ALB_ARN}"

```
## (7)-(c) ターゲット作成
```shell
#ターゲット作成
aws --profile ${PROFILE} \
    elbv2 create-target-group \
        --name "ecs-target" \
        --protocol "HTTP" \
        --port "80" \
        --vpc-id "${VpcId}" \
        --health-check-enabled \
        --health-check-protocol "HTTP" \
        --health-check-interval-seconds "15" \
        --health-check-timeout-seconds "5" \
        --healthy-threshold-count "5" \
        --unhealthy-threshold-count "2" \
        --matcher "HttpCode=200" ;

#Target ARNの取得
TARGET_ARN=$(aws --profile ${PROFILE} --output text \
    elbv2 describe-target-groups \
        --names ecs-target \
    --query 'TargetGroups[].TargetGroupArn' );
echo "TARGET_ARN = ${TARGET_ARN}"
```

## (7)-(d) リスナー作成
```shell
aws --profile ${PROFILE} \
    elbv2 create-listener \
        --load-balancer-arn ${ALB_ARN} \
        --protocol "HTTP" \
        --port "80" \
        --default-actions "Type=forward,TargetGroupArn=${TARGET_ARN}"
```

## (8) ECR
Dockerイメージを格納するECRのレポジトリを準備します。
### (8)-(a) ECRレポジトリ作成
ECRレポジトリを作成
```shell
aws --profile ${PROFILE} \
    ecr create-repository \
        --repository-name "simple-httpserver" \
        --image-tag-mutability "MUTABLE" \
        --image-scanning-configuration "scanOnPush=true" ;
```
### (8)-(b) ECRレポジトリリソースポリシー設定
別途アップデート

### (8)-(c) VPC Endpoint ECRエンドポイントポリシー設定 
別途アップデート

## (9) VPC Endpoint アップデート S3
別途アップデート

## (10) Dockerイメージ(simple-httpserver)の準備
### (10)-(a) Bastionのセットアップ
#### (i) Bastionへログイン
```shell
BastionIP=$( aws --profile ${PROFILE} --output text \
    ec2 describe-instances \
        --filters \
            "Name=instance-state-name,Values=running" \
            "Name=tag:Name,Values=ECSWorker-Bastionr" \
    --query 'Reservations[].Instances[].PublicIpAddress' );

ssh-add
ssh -A ec2-user@${BastionIP}
```
#### (ii) docker&aws cliセットアップ
```shell
# dockerセットアップ
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

# Setup AWS CLI
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
aws configure set region ${REGION}
aws configure set output json

#動作確認
aws sts get-caller-identity
```
### (10)-(b) Bastionでdockerイメージを作成しECR登録
Bastionインスタンス上で、簡単なhttpサーバーのコンテナイメージ(simple-httpd)を作成し、ECRに登録します。
#### (i) dockerイメージのs作成
```shell
#Bastionサーバ上で実行します
#コンテナイメージ用のディレクトリを作成し移動
mkdir httpd-container
cd httpd-container

#データ用フォルダを作成
mkdir src

#dockerコンテナの定義ファイルを作成
cat > Dockerfile << EOL
# setting base image
FROM php:7.0-apache

COPY src/ /var/www/html/
EOL

#
cat > src/index.php << EOL
<html>
  <head>
    <title>PHP Sample</title>
  </head>
  <body>
    <?php echo gethostname(); ?>
  </body>
</html>
EOL

#Docker build
docker build -t httpd-sample:ver01 .
docker images

#コンテナの動作確認
docker run -d -p 8080:80 httpd-sample:ver01
docker ps #コンテナが稼働していることを確認

#接続確認
# <title>PHP Sample</title>という文字が表示されたら成功！！
curl http://localhost:8080

```
#### (ii) ECRへの登録
```shell
# ECRレポジトリのURL取得
REPO_URL=$( aws --output text \
    ecr describe-repositories \
        --repository-names simple-httpserver \
    --query 'repositories[].repositoryUri' ) ;

# ECR登録用のタグを作成
docker tag httpd-sample:ver01 ${REPO_URL}:latest
docker images #作成したtagが表示されていることを確認

#ECRログイン
#"Login Succeeded"と表示されることを確認
aws ecr get-login-password | docker login --username AWS --password-stdin ${REPO_URL}

#イメージのpush
docker push ${REPO_URL}:latest

#ECR上のレポジトリ確認
aws ecr list-images --repository-name simple-httpserver

```

#### (iii) Bastion
Bastionからログアウトし、作業端末に戻ります。
```shell
exit
```

## (11) Workerの準備
ECSクラスターのAWS ECS Cluster Auto Scalingで利用するための、Worker用AutoScalingを用意します。
設定は[こちら](https://aws.amazon.com/jp/blogs/news/aws-ecs-cluster-auto-scaling-is-now-generally-available/)を参考にしています。

### (11)-(a) Autoscaling様のServiceLinkedRoleの作成
AutoScalingのサービス用に規定のIAMロール(ServiceLinkedRole)を作成します。
```shell
#サービスロールの有無チェック
#このコマンドでロールが表示される場合は作成済みなのでスキップする
aws --profile ${PROFILE} \
    iam get-role --role-name AWSServiceRoleForAutoScaling

#ECSサービスロールの作成
aws --profile ${PROFILE} \
    iam create-service-linked-role \
        --aws-service-name autoscaling.amazonaws.com
```
### (11)-(b) Worker用起動テンプレート

```shell
#ECS クラスター名
#クラスター名を変更する場合は修正して下さい。またECSクラスター作成時の名称も変更して下さい。
ECS_CLUSTER_NAME="ecs-cluster-01" 

#Worker用EC2設定
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。 
INSTANCE_TYPE="m5.large"

# ECSサービスで用意しているSSMのParameter StoreからAMIのIDを取得
ECS_OPTIMIZED_AMZ2_AMI=$(aws --profile ${PROFILE} --output text \
    ssm  get-parameters \
        --names "/aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id" \
    --query 'Parameters[].Value' );

#CloudFormationからの取得
WORKER_SG=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC-SecurityGroup \
        --query 'Stacks[].Outputs[?OutputKey==`WorkerSGId`].[OutputValue]')

#パラメータチェック
echo -e "ECS_CLUSTER_NAME       = ${ECS_CLUSTER_NAME}\nKEYNAME                = ${KEYNAME}\nINSTANCE_TYPE          = ${INSTANCE_TYPE}\nECS_OPTIMIZED_AMZ2_AMI = ${ECS_OPTIMIZED_AMZ2_AMI}\nWORKER_SG              = ${WORKER_SG}"

#テンプレートの作成
USER_DATA_BASE64=$(echo -e \
'#!/bin/bash
echo ECS_CLUSTER='"${ECS_CLUSTER_NAME}"' >> /etc/ecs/ecs.config
' | openssl enc -e -base64 | tr -d '\n')

JSON='{
    "ImageId":"'${ECS_OPTIMIZED_AMZ2_AMI}'",
    "InstanceType":"'${INSTANCE_TYPE}'",
    "KeyName":"'${KEYNAME}'",
    "IamInstanceProfile":{
        "Name": "AmazonEC2ContainerServiceforEC2Role-Profile"
    },
    "NetworkInterfaces":[
        {
            "DeviceIndex":0,
            "AssociatePublicIpAddress":false,
            "Groups":[
                "'${WORKER_SG}'"
            ],
            "DeleteOnTermination":true
        }
    ],
    "Monitoring": {
        "Enabled": true
    },
    "BlockDeviceMappings": [
        {
            "DeviceName": "/dev/xvdcz",
            "Ebs": {
                "VolumeSize": 22,
                "VolumeType": "gp2",
                "DeleteOnTermination": true,
                "Encrypted": true
                }
        }
    ],
    "MetadataOptions": {
        "HttpEndpoint": "enabled",
        "HttpTokens": "required",
        "HttpPutResponseHopLimit": 1
    },
    "UserData": "'"${USER_DATA_BASE64}"'",
    "TagSpecifications": [
        {
            "ResourceType":"instance",
            "Tags":[
                {
                    "Key":"Name",
                    "Value":"ECS-worker"
                }
            ]
        }
    ]
}'
#新規作成する場合
aws  --profile ${PROFILE} \
    ec2 create-launch-template \
        --launch-template-name "ecs-worker-ec2-tamplate" \
        --launch-template-data "${JSON}"

#アップデートする場合
aws  --profile ${PROFILE} \
    ec2 create-launch-template-version \
        --launch-template-name "ecs-worker-ec2-tamplate" \
        --launch-template-data "${JSON}"

```

### (11)-(c) Autoscaling グループの作成
* 起動設定: 作成した起動テンプレートを指定
* サイズ
    * min-size = 0 (ECSからインスタンスを起動するため)
    * desired-capacity = 0 (初期起動数。こちらも上記のminと同じ理由のため0設定)
    * max-size = 4 (任意の設定)


```shell
#構成情報の取得
PrivateSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')

PrivateSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name EcsWorker-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')

AZ1=$(aws --profile ${PROFILE} --output text \
    ec2 describe-subnets \
        --subnet-ids ${PrivateSubnet1Id} \
    --query 'Subnets[].AvailabilityZone' );

AZ2=$(aws --profile ${PROFILE} --output text \
    ec2 describe-subnets \
        --subnet-ids ${PrivateSubnet2Id} \
    --query 'Subnets[].AvailabilityZone' );

TEMPLATE_LATEST_VER=$(aws --profile ${PROFILE} --output text \
    ec2 describe-launch-templates \
        --launch-template-names "ecs-worker-ec2-tamplate" \
    --query 'LaunchTemplates[].LatestVersionNumber')

SERVICE_LINKED_ROLE_ARN=$(aws --profile ${PROFILE}  --output text \
    iam get-role \
        --role-name AWSServiceRoleForAutoScaling \
    --query 'Role.Arn' );

echo -e "PrivateSubnet1Id = ${PrivateSubnet1Id}\nAZ1              = ${AZ1}\nPrivateSubnet2Id = ${PrivateSubnet2Id}\nAZ2              = ${AZ2}\nTEMPLATE_LATEST_VER = ${TEMPLATE_LATEST_VER}\nSERVICE_LINKED_ROLE_ARN = ${SERVICE_LINKED_ROLE_ARN}"

# Autoscalingグループの作成
# subnetの指定は"--vpc-zone-identifier"、subnetのAZを"--availability-zones"に指定する
CONFIG_JSON='{
    "AutoScalingGroupName": "ecs-autoscaling-group",
    "MixedInstancesPolicy": {
        "LaunchTemplate": {
            "LaunchTemplateSpecification": {
                "LaunchTemplateName": "ecs-worker-ec2-tamplate",
                "Version": "'"${TEMPLATE_LATEST_VER}"'"
            },
            "Overrides": [
                {
                    "InstanceType": "c4.large"
                },
                {
                    "InstanceType": "c5.large"
                }
            ]
        },
        "InstancesDistribution": {
            "OnDemandPercentageAboveBaseCapacity": 0,
            "SpotAllocationStrategy": "lowest-price",
            "SpotInstancePools": 2
        }
    },
    "MinSize": 0,
    "MaxSize": 4,
    "DesiredCapacity": 0,
    "VPCZoneIdentifier": "'"${PrivateSubnet1Id},${PrivateSubnet2Id}"'",
    "AvailabilityZones": [
        "'"${AZ1}"'",
        "'"${AZ2}"'"
    ],
    "HealthCheckType": "EC2",
    "HealthCheckGracePeriod": 300,
    "TerminationPolicies": [
        "DEFAULT"
    ],
    "NewInstancesProtectedFromScaleIn": true,
    "ServiceLinkedRoleARN": "'"${SERVICE_LINKED_ROLE_ARN}"'"
}'
aws --profile ${PROFILE} \
    autoscaling create-auto-scaling-group \
        --cli-input-json "${CONFIG_JSON}"

#作成したAutoscalingグループの確認
aws --profile ${PROFILE} \
    autoscaling describe-auto-scaling-groups \
        --auto-scaling-group-names "ecs-autoscaling-group"

```
## (12) ECSクラスター設定

### (12)-(a) ECS管理インスタンスへのログインと初期設定
#### (i) ECS管理インスタンスへログイン
```shell
EcsMgrIP=$( aws --profile ${PROFILE} --output text \
    ec2 describe-instances \
        --filters \
            "Name=instance-state-name,Values=running" \
            "Name=tag:Name,Values=ECS-Manager" \
    --query 'Reservations[].Instances[].PublicIpAddress' );

ssh-add
ssh -A ec2-user@${EcsMgrIP}
```
#### (ii) cliセットアップ
```shell
# Setup AWS CLI
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
aws configure set region ${REGION}
aws configure set output json

#動作確認
aws sts get-caller-identity
```
### (12)-(b) ECSのARNフォーマットのロングネームオプトイン設定
```shell
aws --profile ${PROFILE} \
    ecs put-account-setting \
        --name serviceLongArnFormat \
        --value enabled 
```

### (12)-(c) Capacity Providerの作成
2020年6月時点では、作成したCapacity Providerを削除,変更する手段はなさそうです。[(参考:GithubのISSUE)](https://github.com/aws/containers-roadmap/issues/632)
そのため作成し直す場合は、別名でAutoScalingとCapacity Providerを作成してください。
```shell
PROFILE=default

#設定
ASG_NAME="ecs-autoscaling-group" #Autoscalingグループ名
CProvider_NAME="Provider-ecs-autoscaling-group"

#情報取得
ASG_ARN=$(aws --profile ${PROFILE} --output text \
    autoscaling describe-auto-scaling-groups \
        --auto-scaling-group-names "${ASG_NAME}" \
    --query 'AutoScalingGroups[].AutoScalingGroupARN') ;

echo -e "ASG_ARN = ${ASG_ARN}"

#Autoscaling用のProvider設定JSON
ProviderForASG_JSON='{
    "autoScalingGroupArn": "'"${ASG_ARN}"'",
    "managedScaling": {
        "status": "ENABLED",
        "targetCapacity": 100,
        "minimumScalingStepSize": 2,
        "maximumScalingStepSize": 10
    },
    "managedTerminationProtection": "ENABLED"
}'

#ECS Providerの作成
aws --profile ${PROFILE} \
    ecs create-capacity-provider \
        --name "${CProvider_NAME}" \
        --auto-scaling-group-provider "${ProviderForASG_JSON}"

#作成したECS Providerの確認
aws --profile ${PROFILE} \
    ecs describe-capacity-providers \
        --capacity-providers "${CProvider_NAME}"

```

### (12)-(d) ECSクラスターの作成
```shell
ECS_CLUSTER_NAME="ecs-cluster-01"
CProvider_NAME="Provider-ecs-autoscaling-group"

#ECSクラスター作成
CAPACITY_PROVIDER_STRATEGY_JSON='[
  {
    "capacityProvider": "'"${CProvider_NAME}"'",
    "weight": 1
  }
]'

aws --profile ${PROFILE} \
    ecs create-cluster \
        --cluster-name "${ECS_CLUSTER_NAME}" \
        --capacity-providers "${CProvider_NAME}" \
        --default-capacity-provider-strategy "${CAPACITY_PROVIDER_STRATEGY_JSON}"


#ECSクラスターの確認
aws --profile ${PROFILE} \
    ecs describe-clusters \
        --clusters "${ECS_CLUSTER_NAME}" 

```
## (13)ECSタスク定義の作成
ECSクラスターで稼働させるタスク(１つ以上のコンテナを定義した、ECSのコンテナ起動)の定義である、タスク定義を作成します。
### (13)-(a) 情報設定
```shell
#パラメータ設定
TASK_FAMILY="ecs-task-def"
EXEC_ROLE_NAME="ecsTaskExecutionRole"
ECR_REPOSITORY_NAME="simple-httpserver"

#上記パラメータから必要な情報を設定
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
EXEC_ROLE_ARN=$(aws --profile ${PROFILE} --output text \
    iam get-role \
        --role-name "${EXEC_ROLE_NAME}" \
    --query 'Role.Arn' );

REPO_URL=$( aws --output text \
    ecr describe-repositories \
        --repository-names "${ECR_REPOSITORY_NAME}" \
    --query 'repositories[].repositoryUri' ) ;
DOCKER_IMAGE_URL="${REPO_URL}:latest"
LOG_GROUP_NAME="/ecs/${TASK_FAMILY}"

echo -e "REGION           = ${REGION}\nEXEC_ROLE_ARN    = ${EXEC_ROLE_ARN}\nDOCKER_IMAGE_URL = ${DOCKER_IMAGE_URL}\nLOG_GROUP_NAME   = ${LOG_GROUP_NAME}"
         
``` 
### (13)-(b) LogGroup作成
タスク定義で指定しているLogsのロググループを作成する。(タスク実行ロールにはCreateLogsGroupの権限がないため)
```shell
aws --profile ${PROFILE} \
    logs create-log-group \
        --log-group-name ${LOG_GROUP_NAME};
```
### (13)-(c) ECSタスク定義作成
```shell
#タスク定義作成用のコンフィグ作成
TASK_DEF_JSON='{
    "family": "'"${TASK_FAMILY}"'",
    "requiresCompatibilities": [
            "EC2"
    ], 
    "executionRoleArn": "'"${EXEC_ROLE_ARN}"'",
    "networkMode": "bridge",
    "memory": "256",
    "containerDefinitions": [
        {
            "name": "httpd",
            "image": "'"${DOCKER_IMAGE_URL}"'",
            "cpu": 0, 
            "memory": 256, 
            "portMappings": [
                {
                    "hostPort": 0,
                    "protocol": "tcp",
                    "containerPort": 80
                }
            ],
            "environment": [],
            "mountPoints": [],
            "healthCheck": {
                "retries": 2,
                "interval": 15,
                "command": [
                    "CMD-SHELL",
                    "curl -f http://localhost/ || exit 1"
                ],
                "startPeriod": 60,
                "timeout": 5
            }, 
            "essential": true, 
            "volumesFrom": [],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "'"${LOG_GROUP_NAME}"'",
                    "awslogs-region": "'"${REGION}"'",
                    "awslogs-stream-prefix": "container-httpd"
                }
            }
        }
    ], 
    "placementConstraints": [], 
    "volumes": []
}'

#タスク定義作成
aws --profile ${PROFILE} \
    ecs register-task-definition \
        --cli-input-json "${TASK_DEF_JSON}" ;
```

## (14)ECSサービスの作成

### (14)-(a) 情報設定
```shell
#ECS定義 必要に応じて変更
ECS_SERVICE_NAME="ecs-service"

ECS_CLUSTER_NAME="ecs-cluster-01"
ECS_TASK_DEF_NAME="ecs-task-def"
ECS_CAPACITY_PROVIDER_NAME="Provider-ecs-autoscaling-group"

#ALB定義 必要に応じて変更
ALB_TARGET_NAME="ecs-target"

#タスク定義設定
ALB_CONTAINER_NAME="httpd"
ALB_CONTAINER_PORT="80"

#自動的に取得可能な助情報
#タスク定義の最新バージョン
ECS_TASK_DEF_REVISION=$(aws --profile ${PROFILE} --output text \
    ecs describe-task-definition \
        --task-definition "${ECS_TASK_DEF_NAME}" \
    --query 'taskDefinition.revision' )

#ELBのターゲットARN
TARGET_ARN=$(aws --profile ${PROFILE} --output text \
    elbv2 describe-target-groups \
        --names "${ALB_TARGET_NAME}" \
    --query 'TargetGroups[].TargetGroupArn' );

ECS_SERVICE_ROLE_ARN=$(aws --profile ${PROFILE} --output text \
    iam get-role \
        --role-name "AWSServiceRoleForECS" \
    --query 'Role.Arn' );


echo -e "TARGET_ARN = ${TARGET_ARN}\nECS_SERVICE_ROLE_ARN = ${ECS_SERVICE_ROLE_ARN}"
```

### (14)-(b) ECSサービス作成
```shell
SERVICE_DEF_JSON='{
    "cluster": "'"${ECS_CLUSTER_NAME}"'",
    "serviceName": "'${ECS_SERVICE_NAME}'",
    "taskDefinition": "'${ECS_TASK_DEF_NAME}:${ECS_TASK_DEF_REVISION}'",
    "capacityProviderStrategy": [
        {
            "capacityProvider": "'"${ECS_CAPACITY_PROVIDER_NAME}"'",
            "weight": 1,
            "base": 0
        }
    ],
    "schedulingStrategy": "REPLICA",
    "desiredCount": 2,
    "clientToken": "",
    "deploymentConfiguration": {
        "maximumPercent": 200,
        "minimumHealthyPercent": 100
    },
    "deploymentController": {
        "type": "CODE_DEPLOY"
    },
    "placementStrategy": [
        {
            "type": "spread",
            "field": "attribute:ecs.availability-zone"
        },
        {
            "type": "spread",
            "field": "instanceId"
        }
    ],
    "loadBalancers": [
        {
            "targetGroupArn": "'${TARGET_ARN}'",
            "containerName": "'"${ALB_CONTAINER_NAME}"'",
            "containerPort": '"${ALB_CONTAINER_PORT}"'
        }
    ],
    "role": "'"${ECS_SERVICE_ROLE_ARN}"'",
    "healthCheckGracePeriodSeconds": 0
}'

#サービス作成
aws --profile ${PROFILE} \
    ecs create-service \
        --cli-input-json "${SERVICE_DEF_JSON}" ;
```