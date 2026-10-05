---
title: "AWS CloudFormation으로 2개 AZ 웹 아키텍처 구성하기"
date: "2026-10-04T13:39:54.947Z"
categories:
  - "aws"
  - "cloud"
  - "subnet"
  - "routing"
  - "autoscaler"
  - "disaster-recovery"
author: "현제 김_7254"
slug: "aws_cloudformation으로_2개_az_웹_아키텍처_구성하기"
---

## 오늘 진행한 작업

- CloudFormation 템플릿 기반 2개 AZ VPC 구성

- Internet Gateway, 퍼블릭·프라이빗 서브넷, 라우팅 테이블 구성

- AZ별 NAT Gateway와 Elastic IP 생성

- 인터넷-facing ALB와 Target Group 구성

- Ubuntu t3.micro Launch Template 및 Auto Scaling Group 구성

- PowerShell에서 CloudFormation 템플릿 검증과 스택 생성

- 현재 RDS 인스턴스를 제외한 리소스 생성 완료

> [그림 삽입 위치: CloudFormation 스택 생성 완료 화면]

> 캡션: RDS 인스턴스를 제외한 리소스가 생성된 상태

## 실습 환경

## 목표 아키텍처

<img width="1106" height="669" alt="image" src="https://github.com/user-attachments/assets/931f8477-b122-4128-8af8-4242b0bfbb42" />


이번 실습에서는 EC2를 퍼블릭 서브넷에 배치했다. ALB 뒤의 웹 서버를 프라이빗 서브넷에 배치하는 구성이 일반적으로 더 안전하지만, 이번 실습에서는 EC2 접속과 구조 확인을 위해 퍼블릭 서브넷 구성을 사용했다.

## CloudFormation Stack 구성

```
AWSTemplateFormatVersion: '2010-09-09'
Description: >-
  Two-AZ VPC with public/private subnets, one NAT Gateway per AZ,
  internet-facing ALB, public EC2 Auto Scaling Group, and private Multi-AZ RDS.

Parameters:
  EnvironmentName:
    Type: String
    Default: cfn-lab

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Existing EC2 key pair name

  AmiId:
    Type: AWS::EC2::Image::Id
    Description: Ubuntu Server 24.04 LTS AMI in the selected region

  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]

  DesiredCapacity:
    Type: Number
    Default: 6
    MinValue: 1

  MinSize:
    Type: Number
    Default: 2

  MaxSize:
    Type: Number
    Default: 6

  DBUsername:
    Type: String
    Default: admin
    MinLength: 4
    MaxLength: 16
    AllowedPattern: '[a-zA-Z][a-zA-Z0-9]*'

  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
    MaxLength: 41
    AllowedPattern: '[a-zA-Z0-9????$!%*?&]+'

  SSHCidr:
    Type: String
    Default: 0.0.0.0/0
    Description: Restrict this to your own public IP in real environments.

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-vpc'

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-public-a'}]

  PublicSubnetC:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: 10.0.2.0/24
      MapPublicIpOnLaunch: true
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-public-c'}]

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: 10.0.11.0/24
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-private-a'}]

  PrivateSubnetC:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: 10.0.12.0/24
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-private-c'}]

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties: {VpcId: !Ref VPC}

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: VPCGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetARouteAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PublicSubnetA, RouteTableId: !Ref PublicRouteTable}

  PublicSubnetCRouteAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PublicSubnetC, RouteTableId: !Ref PublicRouteTable}

  NatEIPA:
    Type: AWS::EC2::EIP
    DependsOn: VPCGatewayAttachment
    Properties: {Domain: vpc}

  NatEIPC:
    Type: AWS::EC2::EIP
    DependsOn: VPCGatewayAttachment
    Properties: {Domain: vpc}

  NatGatewayA:
    Type: AWS::EC2::NatGateway
    DependsOn: VPCGatewayAttachment
    Properties:
      AllocationId: !GetAtt NatEIPA.AllocationId
      SubnetId: !Ref PublicSubnetA
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-nat-a'}]

  NatGatewayC:
    Type: AWS::EC2::NatGateway
    DependsOn: VPCGatewayAttachment
    Properties:
      AllocationId: !GetAtt NatEIPC.AllocationId
      SubnetId: !Ref PublicSubnetC
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-nat-c'}]

  PrivateRouteTableA:
    Type: AWS::EC2::RouteTable
    Properties: {VpcId: !Ref VPC}

  PrivateRouteTableC:
    Type: AWS::EC2::RouteTable
    Properties: {VpcId: !Ref VPC}

  PrivateRouteA:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableA
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayA

  PrivateRouteC:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableC
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGatewayC

  PrivateSubnetARouteAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PrivateSubnetA, RouteTableId: !Ref PrivateRouteTableA}

  PrivateSubnetCRouteAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties: {SubnetId: !Ref PrivateSubnetC, RouteTableId: !Ref PrivateRouteTableC}

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: HTTP access to internet-facing ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0}
      SecurityGroupEgress:
        - {IpProtocol: -1, CidrIp: 0.0.0.0/0}

  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: HTTP from ALB, SSH and ICMP for lab administration
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 80, ToPort: 80, SourceSecurityGroupId: !Ref ALBSecurityGroup}
        - {IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: !Ref SSHCidr}
        - {IpProtocol: icmp, FromPort: -1, ToPort: -1, CidrIp: !Ref SSHCidr}
      SecurityGroupEgress:
        - {IpProtocol: -1, CidrIp: 0.0.0.0/0}

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: MySQL access only from web instances
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 3306, ToPort: 3306, SourceSecurityGroupId: !Ref WebSecurityGroup}
      SecurityGroupEgress:
        - {IpProtocol: -1, CidrIp: 0.0.0.0/0}

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VPC
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckProtocol: HTTP
      HealthCheckPath: /
      Matcher: {HttpCode: '200-399'}

  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internet-facing
      Type: application
      Subnets: [!Ref PublicSubnetA, !Ref PublicSubnetC]
      SecurityGroups: [!Ref ALBSecurityGroup]
      Tags: [{Key: Name, Value: !Sub '${EnvironmentName}-alb'}]

  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions: [{Type: forward, TargetGroupArn: !Ref TargetGroup}]

  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref AmiId
        InstanceType: !Ref InstanceType
        KeyName: !Ref KeyName
        SecurityGroupIds: [!Ref WebSecurityGroup]
        MetadataOptions:
          HttpTokens: required
          HttpEndpoint: enabled
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            set -eux
            apt-get update -y
            apt-get install -y apache2
            systemctl enable --now apache2
            TOKEN=$(curl -sX PUT 'http://169.254.169.254/latest/api/token' -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')
            IID=$(curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
            AZ=$(curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
            cat > /var/www/html/index.html <<EOF
            <html><body><h1>CloudFormation Web Server</h1><p>Instance: $IID</p><p>AZ: $AZ</p></body></html>
            EOF

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    DependsOn: Listener
    Properties:
      VPCZoneIdentifier: [!Ref PublicSubnetA, !Ref PublicSubnetC]
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      TargetGroupARNs: [!Ref TargetGroup]
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref DesiredCapacity
      HealthCheckType: ELB
      HealthCheckGracePeriod: 60
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-web'
          PropagateAtLaunch: true

  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Private subnets for Multi-AZ RDS
      SubnetIds: [!Ref PrivateSubnetA, !Ref PrivateSubnetC]

  RDSInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      DBInstanceClass: db.t3.micro
      Engine: mysql
      AllocatedStorage: 20
      StorageType: gp3
      DBName: mydb
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      MultiAZ: true
      PubliclyAccessible: false
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups: [!Ref DBSecurityGroup]
      BackupRetentionPeriod: 7
      DeletionProtection: false

Outputs:
  VPCId: {Value: !Ref VPC}
  ALBDNSName:
    Value: !GetAtt LoadBalancer.DNSName
    Description: Open this DNS name in a browser
  TargetGroupArn: {Value: !Ref TargetGroup}
  AutoScalingGroupName: {Value: !Ref AutoScalingGroup}
  RDSEndpoint:
    Value: !GetAtt RDSInstance.Endpoint.Address
    Description: Private RDS endpoint; not directly reachable from the Internet

```

## Parameter 구성

스택 생성 시 사용한 값은 다음과 같다.

```
KeyName         = cfn-lab-key
AmiId           = ami-0130d8d35bcd2d433
DesiredCapacity = 6
MinSize         = 2
MaxSize         = 6
SSHCidr         = 183.98.32.210/32
DBPassword      = 별도 입력값
```

## 템플릿 검증

```
aws cloudformation validate-template `
  --template-body file://".\cfn-alb-vpc-nat-asg-rds.yaml" `
  --region ap-northeast-2
```

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> tempalte 검증 성공시 오류 없이 항목들이 출력된다.

## 스택 생성

최종적으로 다음 명령으로 스택 생성을 요청했다.

```
aws cloudformation create-stack `
  --stack-name cfn-alb-vpc-lab `
  --template-body 'file://.\cfn-alb-vpc-nat-asg-rds.yaml' `
  --parameters `
    'ParameterKey=KeyName,ParameterValue=cfn-lab-key' `
    'ParameterKey=AmiId,ParameterValue=ami-0130d8d35bcd2d433' `
    'ParameterKey=DesiredCapacity,ParameterValue=6' `
    'ParameterKey=MinSize,ParameterValue=2' `
    'ParameterKey=MaxSize,ParameterValue=6' `
    'ParameterKey=SSHCidr,ParameterValue=183.98.32.210/32' `
    'ParameterKey=DBPassword,ParameterValue=db-password' `
  --region ap-northeast-2
```

PowerShell에서는 각 ParameterKey=... 항목 전체를 따옴표로 감싸는 방식으로 전달한다.

## 리소스 생성 상태 확인

스택 상태는 다음 명령으로 확인한다.

```
aws cloudformation describe-stacks `
  --stack-name cfn-alb-vpc-lab `
  --region ap-northeast-2 `
  --query "Stacks[0].[StackName,StackStatus,CreationTime]" `
  --output table
```

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

create-complet 명령으로 스택 생성 확인

스택이 생성하고 있는 리소스 목록 확인. 총 32개가 확인되며 rds 인스턴스가 가장 프로비져닝이 오래걸렸으며 16분정도 소요되었다.

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> CloudFormation Stack events

리소스 생성 확인. 모두 상태가 CREATE_COMPLETE 인지 확인한다

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> :VPC, 서브넷, NAT Gateway, ALB, Launch Template, Auto Scaling Group의 생성 결과

## 생성된 구성요소

### VPC와 라우팅

퍼블릭 서브넷은 0.0.0.0/0 → Internet Gateway 경로를 사용한다. 프라이빗 서브넷은 AZ별 NAT Gateway를 기본 경로로 사용한다. 같은 VPC내부에 있는 서브넷들은 로컬 경로를 사용하므로 별도의 경로 구성없이도 서로 통신이 가능하다.

DB는 보안을 고려해 Private subnet에 구성하고 반드시 nat를 거쳐서 통신하도록 구성한다. NAT Gateway는 퍼블릭 서브넷에 배치되며 프라이빗 서브넷 리소스의 외부 아웃바운드 통신에 사용된다.

통신할 수 있도록 한다. 보안그룹은 stateful 방화벽이기 때문에 외부로 나가면 돌아오는 트래픽은 자동으로 허용된다. 즉, 초기 환경구성을 하기 위해 패키지 다운로드가 가능하다.

```
Public Route Table
10.0.0.0/16 → local
0.0.0.0/0  → Internet Gateway

Private Route Table A
10.0.0.0/16 → local
0.0.0.0/0  → NAT Gateway A

Private Route Table C
10.0.0.0/16 → local
0.0.0.0/0  → NAT Gateway C
```

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> VPC 라우팅 테이블 화면

> 캡션: 퍼블릭 라우팅 테이블의 IGW 경로와 프라이빗 라우팅 테이블의 NAT 경로

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

블랙홀?

### ALB와 Target Group

ALB는 두 퍼블릭 서브넷에 연결하고 TCP 80 Listener를 구성했다.

```
인터넷 사용자
  → ALB :80
  → Target Group
  → EC2 Apache :80
```

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> ALB의 80번 Listener와 Target Group 연결 상태

### 보안그룹

RDS 보안그룹은 인터넷 전체가 아니라 EC2 보안그룹을 소스로 지정한다.

```
ALB Security Group → Web Security Group : TCP 80
Web Security Group → DB Security Group  : TCP 3306
```

## Auto Scaling Group

Auto Scaling Group의 목표 용량은 6으로 설정했다.

```
$ASG = aws cloudformation describe-stacks `
  --stack-name cfn-alb-vpc-lab `
  --query "Stacks[0].Outputs[?OutputKey=='AutoScalingGroupName'].OutputValue" `
  --output text

aws autoscaling describe-auto-scaling-groups `
  --auto-scaling-group-names $ASG `
  --region ap-northeast-2 `
  --query "AutoScalingGroups[0].[MinSize,DesiredCapacity,MaxSize,Instances]" `
  --output json
```

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> Auto Scaling Group 인스턴스 목록

Launch Template의 UserData는 Ubuntu에 Apache를 설치하고 인스턴스 ID와 Availability Zone을 웹 페이지에 표시하도록 구성했다.

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> 실행중인 인스턴스 목록 

### 로드밸런싱 테스트 

alb로 트래픽을 보내 정상적으로 로드밸런싱을 하는지 확인한다. 각 인스턴스에는 시작 템플릿에 구성한

명령을 통해 http 요청을 받으면 인스턴스 id를 출력하도록 구성했다.

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png

> 각 가용영역에 올바르게 로드밸런싱하는 모습 

인스턴스에서 db로 접근 가능한지 확인

!/assets/image_4015550d-923e-4d3f-b7a2-9bb823603ba7.png
