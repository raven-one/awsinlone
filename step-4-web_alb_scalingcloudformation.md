##


## Deploy

```bash
aws cloudformation deploy \
  --stack-name wp-web \
  --template-file 04web_alb_scaling/web-tier.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 04web_alb_scaling/params-web.yaml) \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$REGION" --no-fail-on-empty-changeset


```


## params-web.yaml

```bash
# parameters.sample.yaml
ProjectName: wp-school
VpcId: vpc-XXXXX
PublicSubnet1Id: subnet-XXXXX
PublicSubnet2Id: subnet-XXXXX
ALBSGId: sg-XXXXX
WebSGId: sg-XXXXX
EFSId: fs-XXXXX
EFSAccessPointId: fsap-XXXXX
InstanceType: t3.micro
DesiredCapacity: 2
MinSize: 1
MaxSize: 3

```

## web-tier.yaml

```bash
AWSTemplateFormatVersion: '2010-09-09'
Description: ALB, TG, Listener, ASG web servers mounting EFS and serving WordPress

Parameters:
  ProjectName:
    Type: String
    Default: wp-school
  LatestAL2023Ami:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64
  VpcId:
    Type: AWS::EC2::VPC::Id
  PublicSubnet1Id:
    Type: AWS::EC2::Subnet::Id
  PublicSubnet2Id:
    Type: AWS::EC2::Subnet::Id
  ALBSGId:
    Type: AWS::EC2::SecurityGroup::Id
  WebSGId:
    Type: AWS::EC2::SecurityGroup::Id
  EFSId:
    Type: String       
  EFSAccessPointId:
    Type: String       
  InstanceType:
    Type: String
    Default: t3.micro
  DesiredCapacity:
    Type: Number
    Default: 2
  MinSize:
    Type: Number
    Default: 1
  MaxSize:
    Type: Number
    Default: 3

Resources:
  Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: ec2.amazonaws.com }
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  Profile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles: [ !Ref Role ]

  LT:
    Type: AWS::EC2::LaunchTemplate
    DependsOn: Profile
    Properties:
      LaunchTemplateData:
        ImageId: !Ref LatestAL2023Ami
        InstanceType: !Ref InstanceType
        SecurityGroupIds:
          - !Ref WebSGId
        IamInstanceProfile:
          Name: !Ref Profile
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            exec > >(tee -a /var/log/user-data.log) 2>&1
            set -Eeuo pipefail

            dnf -y upgrade --refresh --allowerasing
            dnf -y install --allowerasing \
              nginx \
              php php-fpm php-mysqlnd php-json php-mbstring php-xml php-gd php-curl \
              amazon-efs-utils nfs-utils unzip tar rsync

            systemctl enable php-fpm nginx

            mkdir -p /var/www/html
            if ! grep -q " ${EFSId}:/ /var/www/html " /etc/fstab; then
              echo "${EFSId}:/ /var/www/html efs _netdev,tls,accesspoint=${EFSAccessPointId},noresvport 0 0" >> /etc/fstab
            fi
            for i in {1..6}; do mount -a && break || sleep 5; done
            echo ok >/var/www/html/health || true

            cat >/etc/nginx/conf.d/wordpress.conf <<'NGINX'
            server {
              listen 80 default_server;
              server_name _;
              root /var/www/html;

              location = /health { return 200 'ok'; add_header Content-Type text/plain; }

              index index.php index.html;
              location / { try_files $uri $uri/ /index.php?$args; }
              location ~ \.php$ {
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_pass unix:/run/php-fpm/www.sock;
              }
            }
            NGINX

            sed -i 's/^user = .*/user = nginx/'  /etc/php-fpm.d/www.conf
            sed -i 's/^group = .*/group = nginx/' /etc/php-fpm.d/www.conf

            nginx -t
            systemctl restart php-fpm nginx

  TG:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      VpcId: !Ref VpcId
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckPath: /health
      HealthCheckProtocol: HTTP
      Matcher: { HttpCode: '200' }
      Name: !Sub '${ProjectName}-tg'

  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub '${ProjectName}-alb'
      Scheme: internet-facing
      Type: application
      SecurityGroups: [ !Ref ALBSGId ]
      Subnets:
        - !Ref PublicSubnet1Id
        - !Ref PublicSubnet2Id

  ListenerHttp:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TG

  ASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PublicSubnet1Id
        - !Ref PublicSubnet2Id
      DesiredCapacity: !Ref DesiredCapacity
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      TargetGroupARNs: [ !Ref TG ]
      LaunchTemplate:
        LaunchTemplateId: !Ref LT
        Version: !GetAtt LT.LatestVersionNumber
      Tags:
        - Key: Name
          Value: !Sub '${ProjectName}-web'
          PropagateAtLaunch: true

Outputs:
  ALBDNS:
    Description: Public URL for WordPress (SiteURL)
    Value: !GetAtt ALB.DNSName

```
