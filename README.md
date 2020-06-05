# Amazon ECSの高セキュリティ環境検証
ECSを、VPC Endpointを利用したインターネット接続のない環境で利用する検証。

# 作成環境
<img src="./Documents/arch.png" whdth=500>

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
ここでは、合計5つのIAMロールを作成します。
<table>
<tr><th colspan=2>Class</th><th>IAM Role</th><th>principal</th><th>Policies summary</th><th>Remark</th></tr>
<tr><td colspan=2>Admin User</td><td>EC2-EcsManagerRole</td><td>ec2</td><td><lu><li>ecs read/write</li><li>ec2(vpc)read</li><li>ec2(ec2)read/write</li><li>iam read</li></lu><li>ECS FullAccess</li><li>Describe other resource</li><li>IAM PassRole</li></td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security_iam_id-based-policy-examples.html">ECS Developer Guide</a></td></tr>
<tr><td rowspan=5>ECS</td><td>Service Linked</td><td>AWSServiceRoleForECS
</td><td>ecs<br>(Service Linked)</td><td>(AWS managed)<br>AmazonECSServiceRolePolicy</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using-service-linked-roles.html">Service-Linked Role for Amazon ECS</a></td></tr>
<tr><td>[ELB]<br>ServiceRole for ECS</td><td>ecsServiceRole</td><td>-</td><td>-</td><td>Service-Linkedに統合されたため不要</td></tr>
<tr><td>Service AutoScaling</td><td>ecsAutoscaleRole</td><td>-</td><td>-</td><td>Service-Linkedに統合されたため不要</td></tr>
<tr><td>Worker(EC2)</td><td>AmazonEC2ContainerServiceforEC2Role</td><td>ec2</td><td>(AWS managed)<br>AmazonEC2ContainerServiceforEC2Role</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html">Amazon ECS Container Instance IAM Role</a></td></tr>
<tr><td>Task</td><td>ecsTaskExecutionRole</td><td>ecs-tasks</td><td>(AWS managed)<br>AmazonECSTaskExecutionRolePolicy</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html">Amazon ECS Task Execution IAM Role</a></td></tr>
<tr><td colspan=2>CloudWatch Events</td><td>AmazonEC2ContainerServiceEventsRole</td><td>events</td><td>(AWS managed)<br>AmazonEC2ContainerServiceEventsRole</td><td><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/CWE_IAM_role.html">ECS CloudWatch Events IAM Role</a></td></tr>
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
        "servicediscovery:Get*"
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
          "iam:PassedToService": "ecs-tasks.amazonaws.com"
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
```

### (5)-(d) タスク実行 IAM ロール
Amazon ECS コンテナエージェントと Fargate タスクの Fargate エージェントが利用するIAMロール
```shell
#ecsServiceRoleロールの作成
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

### (5)-(e) CloudWatch イベント IAM ロール
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
        --policy-name AmazonECSEventsTaskExecutionRole \
        --policy-document "${POLICY}"
```
















## (3) VPCEndpoint設定
必要となるVPC Endpointを作成します。
<img src="./Documents/Step3.png" whdth=500>

### (3)-(a) 構成情報取得
```shell
#構成情報取得
VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

VPC_CIDR=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')

PublicSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet1Id`].[OutputValue]')

PublicSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet2Id`].[OutputValue]')

PrivateSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')

PrivateSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')

PrivateSubnet1RouteTableId=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1RouteTableId`].[OutputValue]')

PrivateSubnet2RouteTableId=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2RouteTableId`].[OutputValue]')

echo -e "VPCID=$VPCID\nVPC_CIDR=$VPC_CIDR\nPublicSubnet1Id =$PublicSubnet1Id\nPublicSubnet2Id =$PublicSubnet2Id\nPrivateSubnet1Id=$PrivateSubnet1Id\nPrivateSubnet2Id=$PrivateSubnet2Id\nPrivateSubnet1RouteTableId=$PrivateSubnet1RouteTableId \nPrivateSubnet2RouteTableId=$PrivateSubnet2RouteTableId"
```
### (3)-(b) VPC: VPCEndpointの作成
```shell
#VPC Endpoint用SecurityGroup作成
VPCENDPOINT_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name VpcEndpointSG \
        --description "Allow https" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${VPCENDPOINT_SG_ID} \
        --tags "Key=Name,Value=VpcEndpointSG" ;

aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${VPCENDPOINT_SG_ID} \
        --protocol tcp \
        --port 443 \
        --cidr ${VPC_CIDR} ;

#Storage Gateway専用のSecurityGroup作成
VPCENDPOINT_STORAGEGW_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name SGW-VpcEndpointSG \
        --description "Allow https" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${VPCENDPOINT_STORAGEGW_SG_ID} \
        --tags "Key=Name,Value=SGW-VpcEndpointSG" ;

# AWS API通信
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${VPCENDPOINT_STORAGEGW_SG_ID} \
        --protocol tcp \
        --port 443 \
        --cidr ${VPC_CIDR} ;

#client-cp接続:Port 1026, proxy-app接続:Port1028
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${VPCENDPOINT_STORAGEGW_SG_ID} \
        --protocol tcp \
        --port 1026-1028 \
        --cidr ${VPC_CIDR} ;

#dp-1接続:Port1031
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${VPCENDPOINT_STORAGEGW_SG_ID} \
        --protocol tcp \
        --port 1031 \
        --cidr ${VPC_CIDR} ;

# Support-channel用(GWからEndpintの2222にsshする)
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${VPCENDPOINT_STORAGEGW_SG_ID} \
        --protocol tcp \
        --port 2222 \
        --cidr ${VPC_CIDR} ;

#S3用VPCEndpoint作成
aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${VPCID} \
        --service-name com.amazonaws.${REGION}.s3 \
        --route-table-ids ${PrivateSubnet1RouteTableId} ${PrivateSubnet2RouteTableId}

#StorageGateway用VPCEndpoint作成
aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${VPCID} \
        --vpc-endpoint-type Interface \
        --service-name com.amazonaws.${REGION}.storagegateway \
        --subnet-id ${PrivateSubnet1Id} ${PrivateSubnet2Id} \
        --security-group-id ${VPCENDPOINT_STORAGEGW_SG_ID} ;

#SSM用PCEndpoint作成
aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${VPCID} \
        --vpc-endpoint-type Interface \
        --service-name com.amazonaws.${REGION}.ssm \
        --subnet-id ${PrivateSubnet1Id} ${PrivateSubnet2Id} \
        --security-group-id ${VPCENDPOINT_SG_ID} ;

aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${VPCID} \
        --vpc-endpoint-type Interface \
        --service-name com.amazonaws.${REGION}.ec2messages \
        --subnet-id ${PrivateSubnet1Id} ${PrivateSubnet2Id} \
        --security-group-id ${VPCENDPOINT_SG_ID} ;

aws --profile ${PROFILE} \
    ec2 create-vpc-endpoint \
        --vpc-id ${VPCID} \
        --vpc-endpoint-type Interface \
        --service-name com.amazonaws.${REGION}.ssmmessages \
        --subnet-id ${PrivateSubnet1Id} ${PrivateSubnet2Id} \
        --security-group-id ${VPCENDPOINT_SG_ID} ;
```
## (4) Storage Gateway管理用のIAMロール(管理サーバ用)作成
<img src="./Documents/Step4.png" whdth=500>

### (4)-(a) IAMロール作成
```shell
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
#IAMロールの作成
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "Ec2-StorageGW-AdminRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200

#AWS管理ポリシーのアタッチ
# AWSStorageGatewayFullAccessのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "Ec2-StorageGW-AdminRole" \
        --policy-arn arn:aws:iam::aws:policy/AWSStorageGatewayFullAccess

# CloudWatch管理者権限のアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "Ec2-StorageGW-AdminRole" \
        --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# ReadOnlyAccessのアタッチ
aws --profile ${PROFILE} \
    iam attach-role-policy \
        --role-name "Ec2-StorageGW-AdminRole" \
        --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

#インスタンスプロファイルの作成
aws --profile ${PROFILE} \
    iam create-instance-profile \
        --instance-profile-name "Ec2-StorageGW-AdminRole-Profile";

aws --profile ${PROFILE} \
    iam add-role-to-instance-profile \
        --instance-profile-name "Ec2-StorageGW-AdminRole-Profile" \
        --role-name "Ec2-StorageGW-AdminRole" ;
```
## (5) Windows/Linuxクライアント、Linux-Manager作成
<img src="./Documents/Step5.png" whdth=500>

### (5)-(a) セキュリティーグループ作成(Bastion)
#### (i) Client - SSHログイン用 Security Group
```shell
# SSHログイン用セキュリティーグループ作成
SSH_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name SshSG \
        --description "Allow ssh" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${SSH_SG_ID} \
        --tags "Key=Name,Value=SshSG" ;

# セキュリティーグループにSSHのinboundアクセス許可を追加
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SSH_SG_ID} \
        --protocol tcp \
        --port 22 \
        --cidr 0.0.0.0/0 ;
```
#### (ii) Client - RDPログイン用 Security Group
```shell
# RDPログイン用セキュリティーグループ作成
RDP_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name RdpSG \
        --description "Allow rdp" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${RDP_SG_ID} \
        --tags "Key=Name,Value=RdpSG" ;

# セキュリティーグループにRDPのinboundアクセス許可を追加
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${RDP_SG_ID} \
        --protocol tcp \
        --port 3389 \
        --cidr 0.0.0.0/0 ;
```
#### (iii) Client識別用 Security Group
```shell
# クライアント識別用セキュリティーグループ作成
CLIENT_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name ClientSG \
        --description "Allow rdp" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${CLIENT_SG_ID} \
        --tags "Key=Name,Value=ClientSG" ;
```
#### (iv) Manager - SSHログイン用 Security Group
```shell
# SSHログイン用セキュリティーグループ作成
MGR_SSH_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name Mgr-SshSG \
        --description "Allow ssh" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${MGR_SSH_SG_ID} \
        --tags "Key=Name,Value=Mgr-SshSG" ;

# セキュリティーグループにSSHのinboundアクセス許可を追加
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${MGR_SSH_SG_ID} \
        --protocol tcp \
        --port 22 \
        --cidr 0.0.0.0/0 ;
```
#### (iiv) セキュリティーグループ設定情報の確認
```shell
SSH_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=SshSG' \
        --query 'SecurityGroups[].GroupId');

RDP_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=RdpSG' \
        --query 'SecurityGroups[].GroupId');

CLIENT_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=ClientSG' \
        --query 'SecurityGroups[].GroupId');

MGR_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=Mgr-SshSG' \
        --query 'SecurityGroups[].GroupId');

#設定情報の表示
echo -e "SSH_SG_ID   =${SSH_SG_ID}\nRDP_SG_ID   =${RDP_SG_ID}\nCLIENT_SG_ID=${CLIENT_SG_ID}\nMGR_SG_ID   =${MGR_SG_ID}"

```
### (5)-(b)インスタンス作成用の事前情報取得
```shell
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。  

#最新のAmazon Linux2のAMI IDを取得します。
AL2_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;

WIN2019_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=Windows_Server-2019-Japanese-Full-Base-????.??.??' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;
echo -e "KEYNAME=${KEYNAME}\nAL2_AMIID=${AL2_AMIID}\nWIN2019_AMIID=${WIN2019_AMIID}"
```
### (5)-(c) Liunux-Client作成
```shell
#インスタンスタイプ設定
#INSTANCE_TYPE="t2.micro"
INSTANCE_TYPE="m5d.8xlarge"

#タグ設定
TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "Linux-Client"
            }
        ]
    }
]'

#ユーザデータ設定
USER_DATA='
#!/bin/bash -xe
                
yum -y update
yum -y install bind bind-utils
hostnamectl set-hostname Linux-Client
'
# サーバの起動
aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${AL2_AMIID} \
        --instance-type ${INSTANCE_TYPE} \
        --key-name ${KEYNAME} \
        --subnet-id ${PublicSubnet1Id} \
        --security-group-ids ${SSH_SG_ID} ${CLIENT_SG_ID}\
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" \
        --user-data "${USER_DATA}" ;
```
### (5)-(d) Windows-Client作成
```shell
#インスタンスタイプ設定
#INSTANCE_TYPE="t2.micro"
INSTANCE_TYPE="m5d.8xlarge"

#タグ設定
TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "Windows-Client"
            }
        ]
    }
]'

# サーバの起動
aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${WIN2019_AMIID} \
        --instance-type ${INSTANCE_TYPE} \
        --key-name ${KEYNAME} \
        --subnet-id ${PublicSubnet1Id} \
        --security-group-ids ${RDP_SG_ID} ${CLIENT_SG_ID}\
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" ;
```
### (5)-(e) Manager(Linux)の作成
ファイルゲートウェイのアクティベーション作業を行う管理インスタンスを起動します。
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
                "Value": "Manager-Linux"
            }
        ]
    }
]'
#ユーザデータ設定
USER_DATA='
#!/bin/bash -xe
                
yum -y update
yum -y install bind bind-utils
hostnamectl set-hostname Mgr-linux
'
# サーバの起動
aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${AL2_AMIID} \
        --instance-type t2.micro \
        --key-name ${KEYNAME} \
        --subnet-id ${PublicSubnet2Id} \
        --security-group-ids ${MGR_SSH_SG_ID}\
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" \
        --user-data "${USER_DATA}" \
        --iam-instance-profile "Name=Ec2-StorageGW-AdminRole-Profile";
```
## (6) StorageGateway作成(事前準備)
Storage Gatewayで利用するS3のバケットと、S3アクセス用にStorage Gatewayが利用するIAMロールを作成します。
<img src="./Documents/Step6.png" whdth=500>

### (6)-(a) StorageGateway用のSecurityGroup作成
#### (i) SGW用 Security Group
```shell
# セキュリティーグループID取得

#Security Group ID取得
CLIENT_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=ClientSG' \
        --query 'SecurityGroups[].GroupId');

MGR_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=Mgr-SshSG' \
        --query 'SecurityGroups[].GroupId');

echo -e "CLIENT_SG_ID=${CLIENT_SG_ID}\nMGR_SG_ID   =${MGR_SG_ID}"

# SGW用セキュリティーグループ作成
SGW_SG_ID=$(aws --profile ${PROFILE} --output text \
    ec2 create-security-group \
        --group-name SGWSG \
        --description "Allow gateway" \
        --vpc-id ${VPCID}) ;

aws --profile ${PROFILE} \
    ec2 create-tags \
        --resources ${SGW_SG_ID} \
        --tags "Key=Name,Value=StorageGWSG" ;

# セキュリティーグループにStorageGatewayに必要となるinboundアクセス許可を追加
# gatewayへのアクティベーションコード取得のため
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 80 \
        --source-group ${MGR_SG_ID} ;

# gatewayへのコンソールログインのため
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 22 \
        --source-group ${MGR_SG_ID} ;

# クライアントとのSMB接続(1)
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 139 \
        --source-group ${CLIENT_SG_ID} ;

# クライアントとのSMB接続(2)
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 445 \
        --source-group ${CLIENT_SG_ID} ;

# クライアントとのNFS接続(1) NFS
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 2049 \
        --source-group ${CLIENT_SG_ID} ;

# クライアントとのNFS接続(2) rpcbind/sunrpc for NFSv3
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 111 \
        --source-group ${CLIENT_SG_ID} ;

# クライアントとのNFS接続(3) gensha for NFSv3
aws --profile ${PROFILE} \
    ec2 authorize-security-group-ingress \
        --group-id ${SGW_SG_ID} \
        --protocol tcp \
        --port 20048 \
        --source-group ${CLIENT_SG_ID} ;
```

### (6)-(b) KMSキー作成
S3暗号化用のKMSのCMK(Customer Master Key)を作成します。キーポリシーはIAMロール作成後に設定します。
```shell
KEY_ID=$( \
aws --profile ${PROFILE} --output text \
    kms create-key \
	    --description "CMK for S3 buckets" \
	    --origin AWS_KMS \
	--query 'KeyMetadata.KeyId' )

aws --profile ${PROFILE} \
    kms create-alias \
	    --alias-name alias/Key_For_S3Buckets \
	    --target-key-id ${KEY_ID}

```


### (6)-(c) StorageGateway用S3バケット作成
#### (i) バケット作成
```shell
BUCKET_NAME="storagegw-bucket-$( od -vAn -to1 </dev/urandom  | tr -d " " | fold -w 10 | head -n 1)"
REGION=$(aws --profile ${PROFILE} configure get region)

#バケット作成
aws --profile ${PROFILE} \
    s3api create-bucket \
        --bucket ${BUCKET_NAME} \
        --create-bucket-configuration LocationConstraint=${REGION};

#デフォルト暗号化設定
CONFIG='
{
    "Rules": [
        {
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "aws:kms",
                "KMSMasterKeyID": "'"${KEY_ARN}"'"
            }
        }
    ]
}'
#KMSによるデフォルト暗号化設定
aws --profile ${PROFILE} \
    s3api put-bucket-encryption \
        --bucket ${BUCKET_NAME} \
        --server-side-encryption-configuration "${CONFIG}"

```

#### (ii) バケットポリシー設定
特定のCMKでの暗号化を強制するバケットポリシーを設定します。
```shell
#情報の収集
#BUCKET_NAME=<バケット名を設定>
KEY_ARN=$(aws --profile ${PROFILE} --output text \
    kms describe-key \
        --key-id alias/Key_For_S3Buckets \
    --query 'KeyMetadata.Arn' \
)
#バケットポリシーJSON生成
POLICY='{
    "Version": "2012-10-17",
    "Id": "S3KeyPolicy",
    "Statement": [
        {
            "Sid": "Force KMS Key",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::'"${BUCKET_NAME}"'/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption-aws-kms-key-id": "'"${KEY_ARN}"'"
                }
            }
        }
    ]
}'
#バケットポリシー設定
aws --profile ${PROFILE} s3api \
    put-bucket-policy \
        --bucket ${BUCKET_NAME} \
        --policy "${POLICY}"
```

### (6)-(d) StorageGateway ファイル共有-S3アクセス用 IAMRole作成
```shell
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "storagegateway.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}'
#IAMロールの作成
aws --profile ${PROFILE} \
    iam create-role \
        --role-name "StorageGateway-S3AccessRole" \
        --assume-role-policy-document "${POLICY}" \
        --max-session-duration 43200

#S3バケットアクセス用 In-line Policyの追加
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OperatBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetAccelerateConfiguration",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning",
        "s3:ListBucket",
        "s3:ListBucketVersions",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": [
        "arn:aws:s3:::'"${BUCKET_NAME}"'"
      ]
    },
    {
      "Sid": "PuAndGetObject",
      "Effect": "Allow",
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:GetObject",
        "s3:GetObjectAcl",
        "s3:GetObjectVersion",
        "s3:ListMultipartUploadParts",
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": [
        "arn:aws:s3:::'"${BUCKET_NAME}"'/*"
      ]
    }
  ]
}'
#インラインポリシーの設定
aws --profile ${PROFILE} \
    iam put-role-policy \
        --role-name "StorageGateway-S3AccessRole" \
        --policy-name "AccessS3buckets" \
        --policy-document "${POLICY}";

#CloudWatch Logs用 In-line Policyの追加
POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLogs",
      "Effect": "Allow",
      "Action": [
            "logs:CreateLogStream",
            "logs:PutLogEvents"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}'
#インラインポリシーの設定
aws --profile ${PROFILE} \
    iam put-role-policy \
        --role-name "StorageGateway-S3AccessRole" \
        --policy-name "PutCLoudWatchLogs" \
        --policy-document "${POLICY}";

```

### (6)-(e) StorageGateway管理用Roleに上記IAMのPassRole権限付与
この後のStorageGatewayのファイル共有作成時(CreateSmbFileShare)に、ファイル共有からS3にPUT/GETするためのIAMロールを指定し割り当ています。この作業時に管理者のIAM権限としてeにPassRoleのアクションを許可する必要があります。
```shell
S3AccessRole_ARN=$(aws --profile ${PROFILE} --output text \
    iam get-role \
        --role-name "StorageGateway-S3AccessRole" \
    --query 'Role.Arn') ;

POLICY='{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PassRole",
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": [
        "'"${S3AccessRole_ARN}"'"
      ]
    }
  ]
}'
#インラインポリシーの設定
aws --profile ${PROFILE} \
    iam put-role-policy \
        --role-name "Ec2-StorageGW-AdminRole" \
        --policy-name "PassRole" \
        --policy-document "${POLICY}";
```

### (6)-(f) CMKキーポリシー設定
```shell
#情報取得
ADMIN_ARN=$(aws --profile ${PROFILE} --output text \
    sts get-caller-identity --query 'Arn')

ACCOUNT_ID=$(aws --profile ${PROFILE} --output text \
    sts get-caller-identity --query 'Account')

S3AccessRole_ARN=$(aws --profile ${PROFILE} --output text \
    iam get-role \
        --role-name "StorageGateway-S3AccessRole" \
    --query 'Role.Arn') ;

STORAGEGW_ADMIN_ROLE_ARN=$(aws --profile ${PROFILE} --output text \
    iam get-role \
        --role-name "Ec2-StorageGW-AdminRole" \
    --query 'Role.Arn') ;


KEY_ARN=$( aws --profile ${PROFILE} --output text \
    kms describe-key \
        --key-id "alias/Key_For_S3Buckets"  \
    --query 'KeyMetadata.Arn')

#キーポリシー用JSON作成
POLICY='{
    "Version": "2012-10-17",
    "Id": "key-default-1",
    "Statement": [
        {
            "Sid": "Administrator",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "'"${ADMIN_ARN}"'",
                    "'"${STORAGEGW_ADMIN_ROLE_ARN}"'",
                    "arn:aws:iam::'"${ACCOUNT_ID}"':role/OrganizationAccountAccessRole"
                ]
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "User",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "'"${S3AccessRole_ARN}"'"
                ]
            },
            "Action": [
                "kms:DescribeKey",
                "kms:Decrypt",
                "kms:GenerateDataKey"
            ],
            "Resource": "*"
        }
    ]
}'

#キーポリシー設定
aws --profile ${PROFILE} \
    kms put-key-policy \
        --key-id "${KEY_ARN}" \
        --policy-name "default" \
        --policy "${POLICY}" \
        --no-bypass-policy-lockout-safety-check ;
```

### (6)-(g) NTP接続不可回避用のRoute53 Private Hosted Zone設定
ファイルゲートウェイに設定されているNTPサーバ(同期先)は、インターネット上のNTPサーバ(x.amazon.pool.ntp.org
)である。そのためファイルゲートウェイを、インターネット接続ができない環境に設置した場合、時刻同期処理を行うことができない。そこで、Route53のPrivate Hosted Zoneを活用し、x.amazon.pool.ntp.orgのアクセス先をAWS time sync(169.254.169.123)にアクセスするようにさせる
```shell
#設定
REGION=$(aws --profile ${PROFILE} configure get region)

#Private Hosted zoneの作成
aws --profile ${PROFILE} \
    route53 create-hosted-zone \
        --name "amazon.pool.ntp.org" \
        --caller-reference $(date '+%Y-%m-%d-%H:%M') \
        --vpc VPCRegion=${REGION},VPCId=${VPCID} ;

HOSTED_ZONE_ID=$(aws --profile ${PROFILE} --output text \
    route53 list-hosted-zones-by-name \
        --dns-name "amazon.pool.ntp.org" \
    --query 'HostedZones[].Id' | sed -e 's/\/hostedzone\///') ;

#レコード登録
CHANGE_BATCH_JSON='{
  "Comment": "CREATE NTP records ",
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "0.amazon.pool.ntp.org",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "169.254.169.123"
          }
        ]
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "1.amazon.pool.ntp.org",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "169.254.169.123"
          }
        ]
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "2.amazon.pool.ntp.org",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "169.254.169.123"
          }
        ]
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "3.amazon.pool.ntp.org",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [
          {
            "Value": "169.254.169.123"
          }
        ]
      }
    }
  ]
}'
#x.amazon.poo..ntp.orgのAレコード登録
aws --profile ${PROFILE} \
    route53 change-resource-record-sets \
            --hosted-zone-id ${HOSTED_ZONE_ID} \
            --change-batch "${CHANGE_BATCH_JSON}";
```

## (7) ファイルゲートウェイの作成
ゲートウェイを作成、アクティベーションして利用可能な状態にします。
<img src="./Documents/Step7.png" whdth=500>

### (7)-(a) ファイルゲートウェイ・インスタンスの作成
#### (i)インスタンスの起動
```shell
# FileGatewayの最新のAMIIDを取得する
FGW_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=aws-storage-gateway-??????????' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' );

#Security Group ID取得
SGW_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=SGWSG' \
        --query 'SecurityGroups[].GroupId');


#ファイルゲートウェイインスタンスの起動
INSTANCE_TYPE=c5.4xlarge
TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "Fgw"
            }
        ]
    }
]'
BLOCK_DEVICE_MAPPINGS='[
    {
        "DeviceName": "/dev/xvda",
        "Ebs": {
            "DeleteOnTermination": true,
            "VolumeType": "io1",
            "Iops": 4000,
            "VolumeSize": 350,
            "Encrypted": false
        }
    },
    {
        "DeviceName": "/dev/sdm",
        "Ebs": {
            "DeleteOnTermination": true,
            "VolumeType": "io1",
            "Iops": 3000,
            "VolumeSize": 16384,
            "Encrypted": false
        }
    }
]'

aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${FGW_AMIID} \
        --instance-type ${INSTANCE_TYPE} \
        --key-name ${KEYNAME} \
        --subnet-id ${PrivateSubnet1Id} \
        --security-group-ids ${SGW_SG_ID} \
        --block-device-mappings "${BLOCK_DEVICE_MAPPINGS}" \
        --tag-specifications "${TAGJSON}" \
        --monitoring Enabled=true ;

```
#### (ii)AutoRecovery設定
```shell
#情報取得
export REGION=ap-northeast-1

GatewayInstanceID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-instances  \
        --filters "Name=tag:Name,Values=Fgw" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].InstanceId' )

#AutoRecovery設定
aws --profile ${PROFILE} \
    cloudwatch put-metric-alarm \
        --region "${REGION}" \
        --alarm-name "recover-ec2-instance-${GatewayInstanceID}" \
        --alarm-description "recover-FileGateway-ec2-instance" \
        --alarm-actions \
            "arn:aws:automate:${REGION}:ec2:recover" \
        --namespace AWS/EC2 \
        --metric-name StatusCheckFailed_System \
        --dimensions  Name=InstanceId,Value=${GatewayInstanceID} \
        --comparison-operator GreaterThanThreshold \
        --unit Count \
        --statistic Average \
        --period 60 \
        --threshold 1 \
        --evaluation-periods 1
```


### (7)-(b) Mgr-Linuxへのログインとセットアップ
以後の作業で、Mgr-Linuxを利用するため、sshログインとセットアップを行います。
#### (i) 作業端末からMgr-Linuxへのログイン(ここではMAC前提)
```shell
MgrIP=$(aws --profile ${PROFILE} --output text \
    ec2 describe-instances  \
        --filters "Name=tag:Name,Values=Manager-Linux" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].PublicIpAddress' )

ssh-add
ssh -A ec2-user@${MgrIP}
```
#### (ii) Mgr-Linuxのセットアップ
以下はSSHログインした、Mgr-Linux上での作業となります。
```shell
# ec2-userログインしてからの作業となります。

#AWS Cliアップデート
curl -o "get-pip.py" "https://bootstrap.pypa.io/get-pip.py" 
sudo python get-pip.py
pip install --upgrade --user awscli
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
. ~/.bashrc

# AWS cli初期設定
Region=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
aws configure set region ${Region}
aws configure set output json

#動作確認
aws sts get-caller-identity

#利用するプロファイル設定
export PROFILE=default
```
#### (iii) 構成情報の取得
```shell
export PROFILE=default
#構成情報取得
VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

VPC_CIDR=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')

PublicSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet1Id`].[OutputValue]')

PublicSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet2Id`].[OutputValue]')

PrivateSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')

PrivateSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')

PrivateSubnet1RouteTableId=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1RouteTableId`].[OutputValue]')

PrivateSubnet2RouteTableId=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2RouteTableId`].[OutputValue]')

echo -e "VPCID=$VPCID\nVPC_CIDR=$VPC_CIDR\nPublicSubnet1Id =$PublicSubnet1Id\nPublicSubnet2Id =$PublicSubnet2Id\nPrivateSubnet1Id=$PrivateSubnet1Id\nPrivateSubnet2Id=$PrivateSubnet2Id\nPrivateSubnet1RouteTableId=$PrivateSubnet1RouteTableId \nPrivateSubnet2RouteTableId=$PrivateSubnet2RouteTableId"
```

### (7)-(b) アクティベーションキーの取得
ファイルゲートウェイから、 アクティベーションキーを取得します。<br>
#### (i)アクティベーション用のURL作成
```shell
#構成情報取得
GatewayIP=$(aws --profile ${PROFILE} --output text \
    ec2 describe-instances  \
        --filters "Name=tag:Name,Values=Fgw" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].PrivateIpAddress' )
REGION=$(aws --profile ${PROFILE} configure get region)
VPCEndpointDNSname=$(aws --profile ${PROFILE} --output text \
    ec2 describe-vpc-endpoints \
        --filters \
            "Name=service-name,Values=com.amazonaws.ap-northeast-1.storagegateway" \
            "Name=vpc-id,Values=${VPCID}" \
    --query 'VpcEndpoints[*].DnsEntries[0].DnsName' );
echo ${GatewayIP} ${REGION} ${VPCEndpointDNSname}

#アクティベーション先のURL生成
ACTIVATION_URL="http://${GatewayIP}/?gatewayType=FILE_S3&activationRegion=${REGION}&vpcEndpoint=${VPCEndpointDNSname}&no_redirect"
echo ${ACTIVATION_URL}
```
#### (ii)アクティベーションキーの取得
````shell
ACTIVATION_KEY=$(curl "${ACTIVATION_URL}")
echo ${ACTIVATION_KEY}
````
参考:
https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/gateway-private-link.html#GettingStartedActivateGateway-file-vpc

### (7)-(c) ゲートウェイのアクティベーション
ファイルゲートウェイをアクティベーションします。
```shell
REGION=$(aws --profile ${PROFILE} configure get region)
aws --profile ${PROFILE} \
    storagegateway activate-gateway \
        --activation-key ${ACTIVATION_KEY} \
        --gateway-name SgPoC-Gateway-1 \
        --gateway-timezone "GMT+9:00" \
        --gateway-region ${REGION} \
        --gateway-type FILE_S3

#作成したGatewayのARN取得
# atewayState"が "RUNNING"になるまで待つ
#ARNがわからない場合は、下記コマンドで確認
#aws --profile ${PROFILE} storagegateway list-gateways
aws --profile ${PROFILE} storagegateway describe-gateway-information --gateway-arn <GATEWAYのARN>
```
＜参考 gateway-typeの説明>
- "STORED" : VolumeGateway(Store type)
- "CACHED" : VolumeGateway(Cache tyep)
- "VTL"    : VirtualTapeLibrary
- "FILE_S3": File Gateway

### (7)-(d) ローカルディスク設定
```shell
#ローカルストレージの確認
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')
DiskIds=$(aws --profile ${PROFILE} --output text storagegateway list-local-disks --gateway-arn ${GATEWAY_ARN} --query 'Disks[*].DiskId'| sed -e 's/\n/ /')
echo ${DiskIds}

#ローカルストレージの割り当て
aws --profile ${PROFILE} storagegateway \
    add-cache \
        --gateway-arn ${GATEWAY_ARN} \
        --disk-ids ${DiskIds}

#ローカルストレージの確認
# "DiskAllocationType"が"CACHE STORAGE"で、"DiskStatus"が"present"であることを確認
aws --profile ${PROFILE} --output text \
    storagegateway list-local-disks \
        --gateway-arn ${GATEWAY_ARN}
```
参照：https://docs.aws.amazon.com/ja_jp/storagegateway/latest/userguide/create-gateway-file.html

### (7)-(e) CloudWatch Logs設定
ファイルゲートウェイインスタンスから、Logsへのログ出力設定を行います。LogsにはS3へのDenyAccess情報などが記録されます。
```shell
#情報取得
GATEWAY_ARN=$(aws --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ match($0, /arn:aws:storagegateway:\S*/); print substr($0, RSTART, RLENGTH) }')
#Region=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
#AccuntId=$(curl -s http://169.254.169.254/latest/meta-data/identity-credentials/ec2/info|awk '/AccountId/{ i=gsub( /"/, "", $3); print $3}')
echo "GATEWAY_ARN=$GATEWAY_ARN  Region=${Region} AccuntId=${AccuntId}"

#CloudWatch Logsログストリーム作成
LOG_GROUP_NAME=${SgPoC-Gateway-1}
aws --profile ${PROFILE} \
    logs create-log-group \
        --log-group-name ${LOG_GROUP_NAME};

LOG_GROUP_ARN=$(aws --profile ${PROFILE} --output text \
    logs describe-log-groups \
        --log-group-name-prefix ${LOG_GROUP_NAME} \
    --query 'logGroups[].arn' );

#ファイルゲートウェイへの出力先ロググループ設定
aws --profile ${PROFILE} \
    storagegateway update-gateway-information \
        --gateway-arn ${GATEWAY_ARN} \
        --cloud-watch-log-group-arn ${LOG_GROUP_ARN}
```

## (8) File Gateway - ファイル共有設定(NFS)
NFSのファイル共有を作成し、LinuxクライアントからNFS接続します。
<img src="./Documents/Step8.png" whdth=500>

### (8)-(a)情報の確認と設定(S3, IAMロール、ゲートウェイ)
Linux Managerで下記設定を実行します。
上記(6)で作成したS3バケット以外のバケットを利用する場合は、(6)-(c)で作成した、"StorageGateway-S3AccessRole"ロールのリソース句に該当のS3バケットを追加してください。
```shell
#情報取得
BUCKET_NAME=<バケット名を個別に設定>
BUCKETARN="arn:aws:s3:::${BUCKET_NAME}"

ROLE="StorageGateway-S3AccessRole"
ROLEARN=$(aws --profile  ${PROFILE} --output text \
    iam get-role \
        --role-name "StorageGateway-S3AccessRole" \
    --query 'Role.Arn')

GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')
CLIENT_TOKEN=$(cat /dev/urandom | base64 | fold -w 38 | sed -e 's/[\/\+\=]/0/g' | head -n 1)
KEY_ARN=$( aws --profile ${PROFILE} --output text \
    kms describe-key \
        --key-id "alias/Key_For_S3Buckets"  \
    --query 'KeyMetadata.Arn')
echo -e "BUCKET=${BUCKETARN}\nROLE_ARN=${ROLEARN}\nGATEWAY_ARN=${GATEWAY_ARN}\nCLIENT_TOKEN=${CLIENT_TOKEN}\nKEY_ARN=${KEY_ARN}"
```
### (8)-(b)ファイル共有(NFS)の作成
```shell
#NFSデフォルト設定
#設定はこちらを参照: https://docs.aws.amazon.com/storagegateway/latest/APIReference/API_NFSFileShareDefaults.html#StorageGateway-Type-NFSFileShareDefaults-OwnerId
FILE_SHARE_DEFAULT_JSON='{
    "FileMode": "0666",
    "DirectoryMode": "0777",
    "GroupId": 65534,
    "OwnerId": 65534
}'

#NFSファイル共有作成
aws --profile ${PROFILE} storagegateway \
    create-nfs-file-share \
        --client-token ${CLIENT_TOKEN} \
        --gateway-arn "${GATEWAY_ARN}" \
        --location-arn "${BUCKETARN}" \
        --role "${ROLEARN}" \
        --nfs-file-share-defaults "${FILE_SHARE_DEFAULT_JSON}" \
        --client-list "0.0.0.0/0" \
        --squash "RootSquash" \
        --kms-encrypted \
        --kms-key ${KEY_ARN} ;
```

### (8)-(c) Linuxクライアントからの接続
作業端末から、Linuxクライアントに接続し、NFSマウントを実行します。

#### (i) 作業端末からMgr-Linuxへのログイン(ここではMAC前提)
以後の作業で、Linux-Clientを利用するため、sshログインとセットアップを行います。
```shell
LinuxClinetIP=$(aws --profile ${PROFILE} --output text \
    ec2 describe-instances  \
        --filters "Name=tag:Name,Values=Linux-Client" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].PublicIpAddress' )

ssh-add
ssh -A ec2-user@${LinuxClinetIP}
```
Linux-Clientにユーザ"ec2-user"でログイン完了した後に、下記コマンドでNFSマウントします。
```shell
sudo -i

#情報の設定と確認
FGWIP=<ファイルGWのPrivateIPを設定>
EXPORT_PATH=<ファイル共有のエクスポートパス情報を(/から始まる情報)を設定>
echo -e "FGWIP=${FGWIP}\nEXPORT_PATH=${EXPORT_PATH}"
```
情報確認後、マウント設定を行います。
```shell
mkdir /nfs

#/etc/fstabへのマウントポイント追加 $ mount実行
echo "${FGWIP}:${EXPORT_PATH}   /nfs     nfs    nolock,hard       0   2" >> /etc/fstab

mount -a
df
```
## (9) File Gateway - ファイル共有設定(SMB - Guest Access)
ActiveDirectoryを利用しない、ゲストアクセスタイプのSMBのファイル共有を作成し、WindowsクライアントからSMB(ゲストアクセス)接続します。
Linux Managerで下記設定を実行します。
<img src="./Documents/Step9.png" whdth=500>

### (9)-(a) SMB設定(SMBSecurityStrategy)
```shell
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')

aws --profile ${PROFILE} storagegateway \
    update-smb-security-strategy \
        --gateway-arn ${GATEWAY_ARN} \
        --smb-security-strategy MandatoryEncryption
```
### (9)-(b) ゲストアクセス用の SMB ファイル共有を設定
```shell
PASSWORD="HogeHoge@"
aws --profile ${PROFILE} storagegateway \
    set-smb-guest-password \
        --gateway-arn ${GATEWAY_ARN} \
        --password ${PASSWORD}
```
### (9)-(c) SMBファイル共有
上記(6)で作成したS3バケット以外のバケットを利用する場合は、(6)-(c)で作成した、"StorageGateway-S3AccessRole"ロールのリソース句に該当のS3バケットを追加してください。
```shell
#情報取得
BUCKET_NAME=<バケット名を個別に設定>
BUCKETARN="arn:aws:s3:::${BUCKET_NAME}"

ROLE="StorageGateway-S3AccessRole"
ROLEARN=$(aws --profile  ${PROFILE} --output text \
    iam get-role \
        --role-name "StorageGateway-S3AccessRole" \
    --query 'Role.Arn')
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')
CLIENT_TOKEN=$(cat /dev/urandom | base64 | fold -w 38 | sed -e 's/[\/\+\=]/0/g' | head -n 1)
KEY_ARN=$( aws --profile ${PROFILE} --output text \
    kms describe-key \
        --key-id "alias/Key_For_S3Buckets"  \
    --query 'KeyMetadata.Arn')
echo -e "BUCKET=${BUCKETARN}\nROLE_ARN=${ROLEARN}\nGATEWAY_ARN=${GATEWAY_ARN}\nCLIENT_TOKEN=${CLIENT_TOKEN}\nKEY_ARN=${KEY_ARN}"

#実行
aws --profile ${PROFILE} storagegateway \
    create-smb-file-share \
        --client-token ${CLIENT_TOKEN} \
        --gateway-arn "${GATEWAY_ARN}" \
        --location-arn "${BUCKETARN}" \
        --role "${ROLEARN}" \
        --object-acl bucket-owner-full-control \
        --default-storage-class S3_STANDARD \
        --guess-mime-type-enabled \
        --authentication GuestAccess \
        --kms-encrypted \
        --kms-key ${KEY_ARN} ;
```
### (9)-(d) Windows-ClinetからのSMBアクセス
Windows ClinetにRDPログインし、SMB接続をします。
説明は省略します。

## (10) File Gateway - ファイル共有設定(SMB - Active Directory)
ファイルゲートウェイとクライアントのWindowsサーバをADに参加させ、AD認証でファイル共有する手順です。この手順ではADに AWS Directory Serviceの[AWS Managed Microsoft AD](https://docs.aws.amazon.com/ja_jp/directoryservice/latest/admin-guide/directory_microsoft_ad.html)を利用しています。
<img src="./Documents/Step10.png" whdth=500>

### (10)-(a) AWS Managed Microsoft AD作成
作業端末で作業を実施ます。
#### (i)情報収集
```shell
PROFILE=<プロファイルを指定>

#構成情報取得
VPCID=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')

PublicSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PublicSubnet2Id`].[OutputValue]')

PrivateSubnet1Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet1Id`].[OutputValue]')

PrivateSubnet2Id=$(aws --profile ${PROFILE} --output text \
    cloudformation describe-stacks \
        --stack-name SGWPoC-VPC \
        --query 'Stacks[].Outputs[?OutputKey==`PrivateSubnet2Id`].[OutputValue]')

echo -e "VPCID=$VPCID\nPublicSubnet2Id=${PublicSubnet2Id}\nPrivateSubnet1Id=$PrivateSubnet1Id\nPrivateSubnet2Id=$PrivateSubnet2Id\n"
```
#### (ii) Managed Microsoft AD(MAD)作成のための情報設定
```shell
AD_NAME="sgwpoc.local"
AD_EDITION="Standard"           #Enterprise or Standard
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。 
AD_PASSWORD="$( cat /dev/urandom | base64 |tr -dc 'a-zA-Z0-9' | fold -w 16 | head -n 1)${RANDOM}"

#パスワードを控える
echo ${AD_PASSWORD}  #￥表示されたパスワードは後の手順で利用するためメモしておく

```
* ADのパスワードは、8～64 文字で指定し、「admin」という語は含めず、英小文字、英大文字、数字、特殊文字の 4 つのカテゴリのうちの 3 つを含める必要があります。

#### (iii) Managed Microsoft AD(MAD)作成
MADを作成します。MADのENIに付与されるセキュリティーグループは、ADサービスが自動的に作成します。
```shell
# MAD作成
aws --profile ${PROFILE} ds \
    create-microsoft-ad \
        --name "${AD_NAME}" \
        --short-name "SgwPoC" \
        --description "AD for StorageGateway PoC" \
        --password "${AD_PASSWORD}" \
        --edition "${AD_EDITION}" \
        --vpc-settings "VpcId=${VPCID},SubnetIds=${PrivateSubnet1Id},${PrivateSubnet2Id}" ;
```
### (10)-(b) AD管理用のWindows-AD-Mgr作成
#### (i)構成情報取得
```shell
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。 
INSTANCE_TYPE="t2.micro"

#以下は自動取得
WIN2019_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=Windows_Server-2019-Japanese-Full-Base-????.??.??' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;

RDP_SG_ID=$(aws --profile ${PROFILE} --output text \
        ec2 describe-security-groups \
                --filter 'Name=group-name,Values=RdpSG' \
        --query 'SecurityGroups[].GroupId');

#設定確認
echo -e "KEYNAME=${KEYNAME}\nINSTANCE_TYPE=${INSTANCE_TYPE}\nWIN2019_AMIID=${WIN2019_AMIID}\nRDP_SG_ID=${RDP_SG_ID}"
```
#### (ii) インスタンス作成
```shell
#タグ設定
TAGJSON='
[
    {
        "ResourceType": "instance",
        "Tags": [
            {
                "Key": "Name",
                "Value": "Windows-AD-Mgr"
            }
        ]
    }
]'

# サーバの起動
aws --profile ${PROFILE} \
    ec2 run-instances \
        --image-id ${WIN2019_AMIID} \
        --instance-type ${INSTANCE_TYPE} \
        --key-name ${KEYNAME} \
        --subnet-id ${PublicSubnet2Id} \
        --security-group-ids ${RDP_SG_ID} \
        --associate-public-ip-address \
        --tag-specifications "${TAGJSON}" ;
```
### (10)-(c) AD管理用のWindows-AD-MgrへのAD管理ツールセットアップとAD参加
* Windows-AD-MgrにRDPでログインします。
* AD管理に必要なツールをPowerShellでインストールします。
```ps1
Import-Module ServerManager
Install-WindowsFeature -Name GPMC,RSAT-AD-PowerShell,RSAT-AD-AdminCenter,RSAT-ADDS-Tools,RSAT-DNS-Server
```

* 以下の手順でドメイン参加させます。
+ DNSの参照先を`ADのDNSアドレス(IPアドレス)`に変更します。
```ps1
Get-NetAdapter | Set-DnsClientServerAddress -ServerAddresses <ADの1つ目のIPアドレス>,<ADの2つ目のIPアドレス>

#設定の確認
Get-NetAdapter | Get-DnsClientServerAddress
```
+ ドメイン`sgwpoc.local`に参加します。
```ps1
#ドメインに参加させます。実行すると adminのパスワードを聞かれるので、AD作成時(create-microsoft-ad)に設定したパスワードを入力します。
Add-Computer -DomainName sgwpoc.local -Credential admin

#有効にするためリブートします。
Restart-Computer
```
+ RDPで、ドメイン`sgwpoc.local`、ユーザ`Admin`でログインします。
+ Power Shellを起動し`sgwpoc.local`のドメイン情報を参照できることを確認します。
```ps1
 Get-ADDomain -Identity sgwpoc.local
```

### (10)-(d) Windowsクライアントのドメイン参加
Windows-Clientインスタンスについて、(10)-(C)と同じ手順で、ADに参加させます。
### (10)-(e) ファイルゲートウェイのドメイン参加

### (i) ファイルゲートウェイのDNS設定変更
ファイルゲートウェイのDNS参照先を、ADに変更します。変更は、ファイルゲートウェイにssh接続して変更します。
```shell
#Linux-Mgrにログイン
MgrIP=$(aws --profile ${PROFILE} --output text \
    ec2 describe-instances  \
        --filters "Name=tag:Name,Values=Manager-Linux" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].PublicIpAddress' )

ssh-add
ssh -A ec2-user@${MgrIP}
```

```shell
#ゲートウェイのIP取得
GatewayIP=$(aws --output text ec2 describe-instances --query 'Reservations[*].Instances[*].PrivateIpAddress' --filters "Name=tag:Name,Values=Fgw" "Name=instance-state-name,Values=running")

#sshログイン
ssh admin@${GatewayIP}
```
ファイルゲートウェイの下記メニューが表示されたら、以下の箇条書きの内容を参考にDNS設定を行います。
```shell
	AWS Storage Gateway - Configuration

	#######################################################################
	##  Currently connected network adapters:
	##
	##  eth0: 10.1.163.138	
	#######################################################################

	1: HTTP/SOCKS Proxy Configuration
	2: Network Configuration
	3: Test Network Connectivity
	4: View System Resource Check (0 Errors)
	5: License Information
	6: Command Prompt

	Press "x" to exit session

        Enter command: 
```
* `Network Configuration`を選び、`Edit DNS Configuration`を選択
* `Available adapters`で`eth0`を入力
* `Assign by DHCP [y/n]:`は、DNSを静的設定するため`n`を選択
* `Enter primary DNS`と`Enter secondary DNS`に、ADの`DNSアドレス`のIPアドレスを指定
* `	Apply config [y/n]:`で`y`で設定反映する
* 終了する

#### (ii)ゲートウェイのAD参加
```shell
#一般設定
PROFILE="default"
#AD設定
AD_DOMAIN_NAME="sgwpoc.local"
AD_USER="admin"
AD_PASSWORD="< (10)-(a) (ii)で設定したパスワード>"

#ゲートウェイID取得
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')

# 実行
aws --profile ${PROFILE} \
    storagegateway join-domain \
        --gateway-arn ${GATEWAY_ARN} \
        --domain-name ${AD_DOMAIN_NAME} \
        --user-name ${AD_USER} \
        --password ${AD_PASSWORD} ; 
```
#### (iii) SMBファイル共有(AD認証)の作成
上記(6)で作成したS3バケット以外のバケットを利用する場合は、(6)-(c)で作成した、"StorageGateway-S3AccessRole"ロールのリソース句に該当のS3バケットを追加してください。
```shell
#情報取得
BUCKET_NAME=storagegw-bucket-smb-ad #<バケット名を個別に設定>
BUCKETARN="arn:aws:s3:::${BUCKET_NAME}"

ROLE="StorageGateway-S3AccessRole"
ROLEARN=$(aws --profile  ${PROFILE} --output text \
    iam get-role \
        --role-name "StorageGateway-S3AccessRole" \
    --query 'Role.Arn')
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')
CLIENT_TOKEN=$(cat /dev/urandom | base64 | fold -w 38 | sed -e 's/[\/\+\=]/0/g' | head -n 1)
KEY_ARN=$( aws --profile ${PROFILE} --output text \
    kms describe-key \
        --key-id "alias/Key_For_S3Buckets"  \
    --query 'KeyMetadata.Arn')
echo -e "BUCKET=${BUCKETARN}\nROLE_ARN=${ROLEARN}\nGATEWAY_ARN=${GATEWAY_ARN}\nCLIENT_TOKEN=${CLIENT_TOKEN}\nKEY_ARN=${KEY_ARN}"

#実行
aws --profile ${PROFILE} storagegateway \
    create-smb-file-share \
        --client-token ${CLIENT_TOKEN} \
        --gateway-arn "${GATEWAY_ARN}" \
        --location-arn "${BUCKETARN}" \
        --role "${ROLEARN}" \
        --object-acl bucket-owner-full-control \
        --default-storage-class S3_STANDARD \
        --guess-mime-type-enabled \
        --authentication ActiveDirectory \
        --kms-encrypted \
        --kms-key ${KEY_ARN} ;
```
#### (iv) Windows-Clientからの接続
クライアントから接続確認します。
```shell
net use [WindowsDriveLetter]: \\10.1.163.138\storagegw-bucket-smb-ad
```

## (11)運用その他
### (11)-(a) キャッシュリフレッシュ
#### (i)対象バケット全体をリフレッシュする
```shell
FILE_SHARE_ARN="<操作したいファイル共有のARNを指定する>"
FOLDER_LIST='/'

#リフレッシュの実行(デフォルトの動作)
aws --profile ${PROFILE} storagegateway \
    refresh-cache \
        --file-share-arn ${FILE_SHARE_ARN} \
        --folder-list ${FOLDER_LIST} \
        --recursive ;
```
* デフォルト(パラメータ未指定)では、下記条件でリフレッシュされる
    * --folder-list: "/"(バケット全体が対象)
    * --recursive: フォルダーリストで指定したフォルダから、登録済みのサブフォルダを再帰的にリフレッシュする

#### (ii)特定のフォルダをリフレッシュする
```shell
FILE_SHARE_ARN=ファイル共有のARNを設定
FOLDER_LIST='/test2 /test3'

#ファイル共有のARNは、
#”aws --profile ${PROFILE} storagegateway list-file-shares”で確認

#リフレッシュの実行(デフォルトの動作)
aws --profile ${PROFILE} storagegateway \
    refresh-cache \
        --file-share-arn ${FILE_SHARE_ARN} \
        --folder-list ${FOLDER_LIST} \
        --recursive ;
```

### (11)-(b) ソフトウェアアップデート
#### (i)構成確認
```shell
#構成情報の取得
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')

#アップデート状況の確認
#"NextUpdateAvailabilityDate"と、"LastSoftwareUpdate"を確認します。
#"NextUpdateAvailabilityDate"が表示されない場合、現時点で予定されているアップデートはありません。
aws --profile ${PROFILE} storagegateway \
    describe-gateway-information \
        --gateway-arn ${GATEWAY_ARN};
```
#### (ii)ソフトウェアアップデート
```shell
aws --profile ${PROFILE} storagegateway \
    update-gateway-software-now \
        --gateway-arn ${GATEWAY_ARN};
```
#### (iii)アップデート確認
```shell
#アップデート状況の確認
#"NextUpdateAvailabilityDate"が表示されなくなり、"LastSoftwareUpdate"が直近の時間に変更されていることを確認します。
aws --profile ${PROFILE} storagegateway \
    describe-gateway-information \
        --gateway-arn ${GATEWAY_ARN};
```
### (11)-(c) イベント連携(リフレッシュ完了通知のイベント連携)
<img src="./Documents/Step11.png" whdth=500>

### (i)SNS準備
以下の作業は、`AdministratorAccess`のIAMポリシーがある作業端末で実行します。
```shell
EMAIL_ADDRESS='email@address'

#Topicの作成
TOPIC_ARN=$(aws --profile ${PROFILE} --output text \
    sns create-topic \
        --name Sgw-Topic)

#Sbscribeでメールを設定
aws --profile ${PROFILE} sns \
    subscribe \
        --topic-arn ${TOPIC_ARN} \
        --protocol email \
        --notification-endpoint ${EMAIL_ADDRESS}
        
#設定したメールアドレスに、確認メールが届くので"Confirm subscription"をクリックし、承認します。

#メール送信テスト
aws --profile ${PROFILE} sns \
    publish \
        --topic-arn ${TOPIC_ARN} \
        --message 'Hello World!!' ;
```

### (ii)CloudWatch Events Roule作成とSNS Topicリソースポリシー変更
StorageGatewayのリフレッシュキャッシュの完了通知を受けて、作成したSNS Topicに連携する、Eventsのルールを作成します。
```shell
#構成情報の取得
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')

FILE_SHARE_ARNs=$(aws --profile ${PROFILE} --output text \
    storagegateway list-file-shares \
        --gateway-arn ${GATEWAY_ARN} \
    --query FileShareInfoList[].FileShareARN )

TOPIC_ARN=$( aws --profile ${PROFILE} --output text \
    sns list-topics | awk '/Sgw-Topic/{print $2}' )

ACCOUNT_ID=$(aws --profile ${PROFILE} --output text \
    sts get-caller-identity --query 'Account')

#ルールパターン用JSON生成
EVENT_PATTERN='
{
  "source": [
    "aws.storagegateway"
  ],
  "resources":['"$( echo $(for i in $FILE_SHARE_ARNs;do echo '"'"${i}"'",';done)|sed -e 's/,$//')"'
  ],
  "detail-type": [
    "Storage Gateway Refresh Cache Event"
  ]
}'

# Event Rule作成
aws --profile ${PROFILE} --output text events \
    put-rule \
        --name SgwPoc-Finish-RefleshCache \
        --description "Receive notification of completion of specified file gateways RefreshCache and notify SNS topic" \
        --event-pattern "${EVENT_PATTERN}" \
        --schedule-expression "" \
        --state ENABLED ;

#ルールにターゲット設定
aws --profile ${PROFILE} events \
    put-targets \
        --rule SgwPoc-Finish-RefleshCache \
        --targets "Id=1,Arn=${TOPIC_ARN},Input=,InputPath=";

#SNS Topicのリソースポリシーにeventsのallowを追加
JSON='{
  "Version": "2012-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__default_statement_ID",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "SNS:GetTopicAttributes",
        "SNS:SetTopicAttributes",
        "SNS:AddPermission",
        "SNS:RemovePermission",
        "SNS:DeleteTopic",
        "SNS:Subscribe",
        "SNS:ListSubscriptionsByTopic",
        "SNS:Publish",
        "SNS:Receive"
      ],
      "Resource": "'"${TOPIC_ARN}"'",
      "Condition": {
        "StringEquals": {
          "AWS:SourceOwner": "'"${ACCOUNT_ID}"'"
        }
      }
    },
    {
      "Sid": "AWSEvents_SgwPoc-Finish-RefleshCache_1",
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sns:Publish",
      "Resource": "'"${TOPIC_ARN}"'"
    }
  ]
}'
#
aws --profile ${PROFILE} sns \
    set-topic-attributes \
        --topic-arn ${TOPIC_ARN} \
        --attribute-name Policy \
        --attribute-value "${JSON}"
```
### (iii)リフレッシュ実行
```shell
FILE_SHARE_ARN="<操作したいファイル共有のARNを指定する>"
FOLDER_LIST='/'

#リフレッシュの実行(デフォルトの動作)
aws --profile ${PROFILE} \
    storagegateway refresh-cache \
        --file-share-arn ${FILE_SHARE_ARN} \
        --folder-list ${FOLDER_LIST} \
        --recursive  ;
```
### (11)-(d) イベント連携(ファイルアップロード完了通知)
(11)-(c)のリフレッシュ完了通知と同じSNS Topicを利用し、ファイルのアップロード完了通知を行います。本作業の前提として、(11)-(c)の設定済みであることとします。
#### (i) CloudWatch Events設定
```shell
#構成情報の取得
GATEWAY_ARN=$(aws --profile ${PROFILE} --output text storagegateway list-gateways |awk '/SgPoC-Gateway-1/{ print $4 }')

FILE_SHARE_ARNs=$(aws --profile ${PROFILE} --output text \
    storagegateway list-file-shares \
        --gateway-arn ${GATEWAY_ARN} \
    --query FileShareInfoList[].FileShareARN )

TOPIC_ARN=$( aws --profile ${PROFILE} --output text \
    sns list-topics | awk '/Sgw-Topic/{print $2}' )

#ルールパターン用JSON生成
EVENT_PATTERN='
{
  "source": [
    "aws.storagegateway"
  ],
  "resources":['"$( echo $(for i in $FILE_SHARE_ARNs;do echo '"'"${i}"'",';done)|sed -e 's/,$//')"'
  ],
  "detail-type": [
    "Storage Gateway File Upload Event"
  ]
}'

# Event Rule作成
aws --profile ${PROFILE} --output text events \
    put-rule \
        --name SgwPoc-Finish-File-Upload \
        --description "Receive notification of completion of specified file gateways files upload and notify SNS topic" \
        --event-pattern "${EVENT_PATTERN}" \
        --schedule-expression "" \
        --state ENABLED ;

#ルールにターゲット設定
aws --profile ${PROFILE} events \
    put-targets \
        --rule SgwPoc-Finish-File-Upload  \
        --targets "Id=1,Arn=${TOPIC_ARN},Input=,InputPath=";
```

#### (ii) アップロード完了通知設定
```shell
#必要に応じ、FILE_SHARE_ARNを設定
FILE_SHARE_ARN="<操作したいファイル共有のARNを指定する>"

#何らかファイルコピー実行と同時に、下記のnotify-when-uploadedを実行
aws --profile ${PROFILE} storagegateway \
    notify-when-uploaded \
        --file-share-arn ${FILE_SHARE_ARN};
```


