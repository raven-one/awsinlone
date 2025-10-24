### Contains Network.yaml and params-network.yaml


## Deploy
```bash
aws cloudformation deploy \
  --stack-name wp-net \
  --template-file network.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' params-network.yaml) \
  --region eu-west-1


```


## params-network.yaml

```bash
ProjectName: wp-school
VPCCidr: 10.0.0.0/16
PublicSubnet1Cidr: 10.0.0.0/24
PublicSubnet2Cidr: 10.0.1.0/24

# För test syfte *. I produktion direkt kopplad till bastion/ssh_server
MyIP: 0.0.0.0/0



```



## Network.yaml

```bash
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC, public subnets, routing and security_groups for ALB, Web, Admin, SQL, EFS

Parameters:
  ProjectName:
    Type: String
    Default: wp-school
  VPCCidr:
    Type: String
    Default: 10.0.0.0/16
  PublicSubnet1Cidr:
    Type: String
    Default: 10.0.0.0/24
  PublicSubnet2Cidr:
    Type: String
    Default: 10.0.1.0/24
  MyIP:
    Type: String
    Description: Your IP/CIDR for SSH (admin server). Use 0.0.0.0/0 for demo.
    Default: 0.0.0.0/0

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VPCCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-vpc'}]

  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-igw'}]

  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      CidrBlock: !Ref PublicSubnet1Cidr
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-public-a'}]

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      CidrBlock: !Ref PublicSubnet2Cidr
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-public-b'}]

  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-public-rt'}]

  PublicRoute0:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW
    DependsOn: IGWAttachment

  Assoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRT

  Assoc2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRT

  # --- Security Groups ---
  ALBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB security group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0}
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-alb-sg'}]

  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web EC2 instances (behind ALB)
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSG
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-web-sg'}]

  AdminSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Admin/provisioning EC2
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: !Ref MyIP}
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-admin-sg'}]

  RDSSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: RDS MariaDB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 3306, ToPort: 3306, SourceSecurityGroupId: !Ref WebSG}
        - {IpProtocol: tcp, FromPort: 3306, ToPort: 3306, SourceSecurityGroupId: !Ref AdminSG}
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-rds-sg'}]

  EFSSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: EFS NFS
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 2049, ToPort: 2049, SourceSecurityGroupId: !Ref WebSG}
        - {IpProtocol: tcp, FromPort: 2049, ToPort: 2049, SourceSecurityGroupId: !Ref AdminSG}
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-efs-sg'}]

Outputs:
  VpcId: {Value: !Ref VPC, Export: {Name: !Sub '${ProjectName}:VpcId'}}
  PublicSubnet1Id: {Value: !Ref PublicSubnet1, Export: {Name: !Sub '${ProjectName}:PublicSubnet1Id'}}
  PublicSubnet2Id: {Value: !Ref PublicSubnet2, Export: {Name: !Sub '${ProjectName}:PublicSubnet2Id'}}
  ALBSGId: {Value: !Ref ALBSG, Export: {Name: !Sub '${ProjectName}:ALBSGId'}}
  WebSGId: {Value: !Ref WebSG, Export: {Name: !Sub '${ProjectName}:WebSGId'}}
  AdminSGId: {Value: !Ref AdminSG, Export: {Name: !Sub '${ProjectName}:AdminSGId'}}
  RDSSGId: {Value: !Ref RDSSG, Export: {Name: !Sub '${ProjectName}:RDSSGId'}}
  EFSSGId: {Value: !Ref EFSSG, Export: {Name: !Sub '${ProjectName}:EFSSGId'}}


```


