LDAV DynamoDB Workshop — CloudShell Deployment

Workshop Link
- https://catalog.workshops.aws/dynamodb-labs/en-US/design-patterns/setup/user-account

This repo provides CloudFormation templates for the LDAV DynamoDB workshop. Cloud9 is no longer used; instead, the `CloudShell.yaml` template provisions only the resources you need and is designed to be operated directly from AWS CloudShell (or your local AWS CLI).

- Use `CloudShell.yaml` for CloudShell-based deployments.
- Use `CloudShell-eu.yaml` if you want to enforce deployment in EU regions (will fail if run outside EU).
- `C9-Original-Do-Not-Use.yaml` remains for reference only and should not be deployed.

Prerequisites
- AWS CLI v2 configured with a profile that has permissions for CloudFormation, IAM, EC2, S3, Lambda, and SSM (Session Manager). Example profile: `chetz-playground`.
- Choose your target region in the CLI config or via `--region`.

What Gets Created
- DDBReplicationRole: IAM role for DynamoDB replication labs.
- MigrationS3Bucket: S3 bucket for staging migration artifacts.
- DbSecurityGroup: EC2 SG allowing MySQL within the default VPC CIDR and optional SSH via EC2 Instance Connect prefix list.
- DBInstanceRole + DBInstanceProfile: IAM role/profile granting SSM access and S3 access.
- DbInstance: EC2 (Amazon Linux 2) installs MySQL 8 and preloads IMDB datasets via UserData.

Access to the EC2 instance is via SSM Session Manager (no public IP required). CloudShell can start SSM sessions out of the box.

Parameters
- DbMasterPassword: MySQL password for `dbuser` and root reset (required).
- DbMasterUsername: MySQL username (default `dbuser`).
- InstanceType: EC2 instance type (default `t3.medium`).
- DBLatestAmiId: SSM AL2 image parameter (override only if needed).
- EnvironmentName: Tagging/label (default `DynamoDBID`).

Deploy
Validate and deploy using your profile (replace the password):

`aws --profile chetz-playground cloudformation validate-template --template-body file://CloudShell.yaml`

`aws --profile chetz-playground cloudformation deploy --template-file CloudShell.yaml --stack-name ldav-cloudshell --capabilities CAPABILITY_NAMED_IAM --parameter-overrides DbMasterPassword='REPLACE_ME_SECURELY'`

EU‑only variant (fails if not deployed in an EU region):

`aws --profile chetz-playground --region eu-central-1 cloudformation deploy --template-file CloudShell-eu.yaml --stack-name ldav-cloudshell-eu --capabilities CAPABILITY_NAMED_IAM --parameter-overrides DbMasterPassword='REPLACE_ME_SECURELY'`

Fetch outputs (IDs and names used below):

`aws --profile chetz-playground --output json cloudformation describe-stacks --stack-name ldav-cloudshell | jq -r '.Stacks[0].Outputs[] | [.OutputKey,.OutputValue] | @tsv' | sed -e $'s/\t/: /'`

Common single-output lookups:

`DB_INSTANCE_ID=$(aws --profile chetz-playground --output json cloudformation describe-stacks --stack-name ldav-cloudshell | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="DbInstanceId").OutputValue')`

`BUCKET=$(aws --profile chetz-playground --output json cloudformation describe-stacks --stack-name ldav-cloudshell | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="MigrationS3BucketName").OutputValue')`

Connect From CloudShell
Start an SSM interactive shell on the EC2 instance:

`aws --profile chetz-playground ssm start-session --target "$DB_INSTANCE_ID"`

Port-forward MySQL to your CloudShell and connect with the MySQL client:

`aws --profile chetz-playground ssm start-session --target "$DB_INSTANCE_ID" --document-name AWS-StartPortForwardingSession --parameters portNumber=3306,localPortNumber=3306`

`mysql -h 127.0.0.1 -P 3306 -u dbuser -p`

Quick verification (should list IMDB and app DBs):

`mysql -h 127.0.0.1 -P 3306 -u dbuser -p -e "SHOW DATABASES;"`

Using the Migration Bucket
The output `MigrationS3BucketName` provides the S3 bucket for staging data and artifacts:

`aws --profile chetz-playground s3 cp localfile s3://$BUCKET/`

`aws --profile chetz-playground s3 ls s3://$BUCKET/`

Clean Up
Delete the stack when finished to avoid charges:

`aws --profile chetz-playground s3 rm s3://$BUCKET --recursive || true`

`aws --profile chetz-playground cloudformation delete-stack --stack-name ldav-cloudshell`

`aws --profile chetz-playground cloudformation wait stack-delete-complete --stack-name ldav-cloudshell`

Notes
- Security group allows MySQL within default VPC CIDR (`172.31.0.0/16`). CloudShell access uses SSM Session Manager, not direct network access.
- SSH via EC2 Instance Connect is allowed by prefix list but typically unnecessary.
- Costs: An EC2 instance and supporting resources run while the stack is deployed. Tear down after the workshop.
- Region-specific defaults: The template automatically selects a default subnet that supports your chosen instance type.

Legacy Cloud9 Template
`C9-Original-Do-Not-Use.yaml` is retained for historical context and should not be deployed now that AWS Cloud9 is deprecated for this workshop path.
