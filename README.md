LDAV DynamoDB Workshop — CloudShell Setup

Quick Link
- Workshop: https://catalog.workshops.aws/dynamodb-labs/en-US/design-patterns/setup/user-account

Overview
- Run everything from AWS CloudShell — no Cloud9 needed.
- Use `CloudShell.yaml` (global) or `CloudShell-eu.yaml` (EU-only).
- `C9-Original-Do-Not-Use.yaml` is legacy for reference only.

What Gets Deployed
- S3 bucket for migration files.
- EC2 instance (Amazon Linux 2) with MySQL 8 and IMDB data.
- IAM role/profile for SSM access; security group for MySQL in VPC.

Before You Start
- Open AWS CloudShell in your target region (e.g., eu-central-1).
- If using a local CLI instead of CloudShell, add `--profile <your-profile>` to the commands.

1) Get The Workshop Files (CloudShell)
```
mkdir -p ~/workshop
cd ~/workshop
curl -L -o workshop.zip https://amazon-dynamodb-labs.com/assets/workshop.zip
unzip -q workshop.zip
```

2) Deploy The Stack
- Global template (any region):
```
aws cloudformation deploy \
  --template-file CloudShell.yaml \
  --stack-name ldav-cloudshell \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides DbMasterPassword='YOUR_SECURE_PASSWORD'
```
- EU-only template:
```
aws --region eu-central-1 cloudformation deploy \
  --template-file CloudShell-eu.yaml \
  --stack-name ldav-cloudshell-eu \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides DbMasterPassword='YOUR_SECURE_PASSWORD'
```

3) Get Outputs
```
aws --output json cloudformation describe-stacks --stack-name ldav-cloudshell \
  | jq -r '.Stacks[0].Outputs[] | [.OutputKey,.OutputValue] | @tsv' \
  | sed -e $'s/\t/: /'

DB_INSTANCE_ID=$(aws --output json cloudformation describe-stacks --stack-name ldav-cloudshell \
  | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="DbInstanceId").OutputValue')
BUCKET=$(aws --output json cloudformation describe-stacks --stack-name ldav-cloudshell \
  | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="MigrationS3BucketName").OutputValue')
```

4) Connect To MySQL
- Port forward from the instance, then connect:
```
aws ssm start-session \
  --target "$DB_INSTANCE_ID" \
  --document-name AWS-StartPortForwardingSession \
  --parameters portNumber=3306,localPortNumber=3306
```
Open a second CloudShell tab:
```
mysql -h 127.0.0.1 -P 3306 -u dbuser -p
```
If `mysql` is missing: `sudo yum -y install mysql`.

5) Use The Migration Bucket
```
aws s3 cp localfile s3://$BUCKET/
aws s3 ls s3://$BUCKET/
```

6) Clean Up
```
aws s3 rm s3://$BUCKET --recursive || true
aws cloudformation delete-stack --stack-name ldav-cloudshell
aws cloudformation wait stack-delete-complete --stack-name ldav-cloudshell
```

Notes
- EC2 access is via SSM only (no public SSH required).
- Security group allows MySQL within the default VPC only.
- Remember to delete the stack after the workshop to stop costs.

Legacy Template
- `C9-Original-Do-Not-Use.yaml` is kept for historical reference.
