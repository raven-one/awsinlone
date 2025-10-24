### This is used to get information from aws to update variables in params yaml files or when some codes an run.
Run from terminal that is connected to aws cli


```bash

REGION=eu-west-1

# --- 01: network ---
aws cloudformation deploy \
  --stack-name wp-net \
  --template-file 01network/network.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 01network/params-network.yaml) \
  --region "$REGION" --no-fail-on-empty-changeset
```
# grab outputs for the next stacks
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
# write into 02 params
yq -i "
  .VpcId=\"$VPC_ID\" |
  .Subnet1Id=\"$SUBNET1\" |
  .Subnet2Id=\"$SUBNET2\" |
  .RDSSGId=\"$RDS_SG\" |
  .EFSSGId=\"$EFS_SG\"
" 02db_efs/params-data.yaml -y
````
# --- 02: db + efs ---
aws cloudformation deploy \
  --stack-name wp-db-efs \
  --template-file 02db_efs/data.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 02db_efs/params-data.yaml) \
  --region "$REGION" --no-fail-on-empty-changeset

# after 02, fill vars for 03 + 04
DATA=wp-db-efs
EFS_ID=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$DATA" --query "Stacks[0].Outputs[?OutputKey=='EFSId'].OutputValue" --output text)
EFS_AP=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$DATA" --query "Stacks[0].Outputs[?OutputKey=='EFSAccessPointId'].OutputValue" --output text)
DB_ENDPOINT=$(aws cloudformation describe-stacks --region "$REGION" --stack-name "$DATA" --query "Stacks[0].Outputs[?OutputKey=='DBEndpoint'].OutputValue" --output text)

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

# --- 03: admin/provisioner (JSON form handles spaces like "Viking Mead") ---
aws cloudformation deploy \
  --stack-name wp-admin \
  --template-file 03admin_server/admin.yaml \
  --parameter-overrides "$(yq -o=json -I=0 'to_entries|map({ParameterKey:.key, ParameterValue:(.value|tostring)})' 03admin_server/params-admin.yaml)" \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$REGION" --no-fail-on-empty-changeset

# --- 04: web tier ---
aws cloudformation deploy \
  --stack-name wp-web \
  --template-file 04web_alb_scaling/web-tier.yaml \
  --parameter-overrides $(yq -r 'to_entries|.[]|"\(.key)=\(.value)"' 04web_alb_scaling/params-web.yaml) \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$REGION" --no-fail-on-empty-changeset

# show ALB URL
ALB_DNS=$(aws cloudformation describe-stacks --region "$REGION" --stack-name wp-web --query "Stacks[0].Outputs[?OutputKey=='ALBDNS'].OutputValue" --output text)
echo "ALB URL: http://$ALB_DNS"


```
