## Deploy
```bash
aws cloudformation deploy \
  --stack-name wp-db-efs \
  --template-file data.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' params-data.yaml) \
  --region eu-west-1


```

## params-data.yaml

```bash
ProjectName: wp-school
VpcId: vpc-09e461f51e187e53d
Subnet1Id: subnet-xxxxxxxx
Subnet2Id: subnet-xxxxxxxx
RDSSGId: sg-xxxxxxxxxx
EFSSGId: sg-xxxxxxxxxx
DBName: wordpress
DBUser: wpuser
DBPassword: strongpassword
BackupRetentionDays: 1


```


## data.yaml

```bash
AWSTemplateFormatVersion: '2010-09-09'
Description: EFS (+ access point) and RDS MariaDB

Parameters:
  ProjectName: {Type: String, Default: wp-school}
  VpcId: {Type: AWS::EC2::VPC::Id}
  Subnet1Id: {Type: AWS::EC2::Subnet::Id}
  Subnet2Id: {Type: AWS::EC2::Subnet::Id}
  RDSSGId: {Type: AWS::EC2::SecurityGroup::Id}
  EFSSGId: {Type: AWS::EC2::SecurityGroup::Id}

  DBName: {Type: String, Default: wordpress}
  DBUser: {Type: String, Default: wpuser}
  DBPassword: {Type: String, NoEcho: true, MinLength: 8}
  BackupRetentionDays: {Type: Number, Default: 1}

Resources:
  # --- EFS ---
  EFS:
    Type: AWS::EFS::FileSystem
    Properties:
      Encrypted: true
      PerformanceMode: generalPurpose
      ThroughputMode: bursting
      FileSystemTags: [{Key: Name, Value: !Sub '${ProjectName}-efs'}]

  EFSAP:
    Type: AWS::EFS::AccessPoint
    Properties:
      FileSystemId: !Ref EFS
      PosixUser: {Uid: "1000", Gid: "1000"}
      RootDirectory:
        CreationInfo: {OwnerUid: "1000", OwnerGid: "1000", Permissions: "0775"}
        Path: /wordpress

  MountA:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref EFS
      SubnetId: !Ref Subnet1Id
      SecurityGroups: [!Ref EFSSGId]

  MountB:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref EFS
      SubnetId: !Ref Subnet2Id
      SecurityGroups: [!Ref EFSSGId]

  # --- RDS ---
  DBSubnets:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: !Sub '${ProjectName} db subnets'
      SubnetIds: [!Ref Subnet1Id, !Ref Subnet2Id]
      DBSubnetGroupName: !Sub '${ProjectName}-dbsubnets'

  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: mariadb
      EngineVersion: "10.11"
      DBInstanceClass: db.t4g.micro
      AllocatedStorage: 20
      StorageType: gp3
      BackupRetentionPeriod: !Ref BackupRetentionDays
      DeletionProtection: false
      MultiAZ: false
      PubliclyAccessible: false
      VPCSecurityGroups: [!Ref RDSSGId]
      DBSubnetGroupName: !Ref DBSubnets
      MasterUsername: !Ref DBUser
      MasterUserPassword: !Ref DBPassword
      DBName: !Ref DBName
      AutoMinorVersionUpgrade: true
      CopyTagsToSnapshot: true
      Tags: [{Key: Name, Value: !Sub '${ProjectName}-mariadb'}]

Outputs:
  EFSId: {Value: !Ref EFS, Export: {Name: !Sub '${ProjectName}:EFSId'}}
  EFSAccessPointId: {Value: !Ref EFSAP, Export: {Name: !Sub '${ProjectName}:EFSAPId'}}
  DBEndpoint:
    Value: !GetAtt DBInstance.Endpoint.Address
    Export: {Name: !Sub '${ProjectName}:DBEndpoint'}
  DBNameOut: {Value: !Ref DBName, Export: {Name: !Sub '${ProjectName}:DBName'}}
  DBUserOut: {Value: !Ref DBUser, Export: {Name: !Sub '${ProjectName}:DBUser'}}


```
