### This is used to get information from aws to update variables in params yaml files or when some codes an run.
Run from terminal that is connected to aws cli



### Start 

REGION variable added
run step 1 yaml files
```bash

REGION=eu-west-1

# --- 01: network ---
aws cloudformation deploy \
  --stack-name wp-net \
  --template-file 01network/network.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 01network/params-network.yaml) \
  --region "$REGION" --no-fail-on-empty-changeset
```


When step 1 deployment is done and network is up and running
We will need to grab some new information and put them into varibales

### Run after step 1 is done
```bash
NET=wp-net
VPC_ID=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)
SUBNET1=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet1Id'].OutputValue" --output text)
SUBNET2=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet2Id'].OutputValue" --output text)
RDS_SG=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='RDSSGId'].OutputValue" --output text)
EFS_SG=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='EFSSGId'].OutputValue" --output text)
ADMIN_SG=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='AdminSGId'].OutputValue" --output text)
ALB_SG=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='ALBSGId'].OutputValue" --output text)
WEB_SG=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$NET" --query "Stacks[0].Outputs[?OutputKey=='WebSGId'].OutputValue" --output text)
```
Write needed information to the step 2 param.yaml file
```bash
# write into 02 params
yq -i "
  .VpcId=\"$VPC_ID\" |
  .Subnet1Id=\"$SUBNET1\" |
  .Subnet2Id=\"$SUBNET2\" |
  .RDSSGId=\"$RDS_SG\" |
  .EFSSGId=\"$EFS_SG\"
" 02db_efs/params-data.yaml -y
````

Deploy step 2, creation of DB and EFS
```bash
# 02: db + efs 
aws cloudformation deploy \
  --stack-name wp-db-efs \
  --template-file 02db_efs/data.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 02db_efs/params-data.yaml) \
  --region "$REGION" --no-fail-on-empty-changeset
```


When step 2 is done and running
we will get needed information from DB and EFS
```bash
DATA=wp-db-efs
EFS_ID=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$DATA" --query "Stacks[0].Outputs[?OutputKey=='EFSId'].OutputValue" --output text)
EFS_AP=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$DATA" --query "Stacks[0].Outputs[?OutputKey=='EFSAccessPointId'].OutputValue" --output text)
DB_ENDPOINT=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$DATA" --query "Stacks[0].Outputs[?OutputKey=='DBEndpoint'].OutputValue" --output text)
```
and than we will add the new varibale information into the params for step 3 and 4
```bash
# write into 03 + 04 params
yq -i "
  .EFSId=\"$EFS_ID\" |
  .EFSAccessPointId=\"$EFS_AP\" |
  .DBEndpoint=\"$DB_ENDPOINT\" |
  .SubnetId=\"$SUBNET1\" |
  .AdminSGId=\"$ADMIN_SG\"
" 03admin_server/params-admin.yaml -y

yq -i "
  .VpcId=\"$VPC_ID\" |
  .PublicSubnet1Id=\"$SUBNET1\" |
  .PublicSubnet2Id=\"$SUBNET2\" |
  .ALBSGId=\"$ALB_SG\" |
  .WebSGId=\"$WEB_SG\" |
  .EFSId=\"$EFS_ID\" |
  .EFSAccessPointId=\"$EFS_AP\"
" 04web_alb_scaling/params-web.yaml -y

```

```bash


# --- 04: web tier ---
aws cloudformation deploy \
  --stack-name wp-web \
  --template-file 04web_alb_scaling/web-tier.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 04web_alb_scaling/params-web.yaml) \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$REGION" --no-fail-on-empty-changeset
```

```bash
# show ALB URL
ALB_DNS=$(aws cloudformation describe-stacks --region "$REGION" --stack-name wp-web --query "Stacks[0].Outputs[?OutputKey=='ALBDNS'].OutputValue" --output text)
echo "ALB URL: http://$ALB_DNS"


```
