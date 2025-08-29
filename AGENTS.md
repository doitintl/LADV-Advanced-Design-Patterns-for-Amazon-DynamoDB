# Session Summary: Migrating from Cloud9 to CloudShell

This document records what we changed, tested, and deployed during the session to modernize the LDAV DynamoDB workshop for AWS CloudShell.

## Scope
- Replace Cloud9-based setup with a CloudShell-first workflow.
- Provide clean, student-friendly documentation for deployment and usage.
- Validate CloudFormation, deploy resources, verify connectivity, and tear down cleanly.
- Push all changes to GitHub for future reuse.

## Repository
- GitHub: https://github.com/doitintl/LADV-Advanced-Design-Patterns-for-Amazon-DynamoDB
- Branch updates:
  - Initial work on `feature/cloudshell-migration`
  - Pushed finalized changes directly to `main` (per request)

## Changes Made
- Added: `CloudShell.yaml`
  - CloudShell-compatible CloudFormation template (no Cloud9 dependency).
  - Creates: S3 migration bucket, EC2 MySQL host (AL2 + MySQL 8 + IMDB sample), IAM role/profile for SSM, SG for MySQL (VPC-only), and DDB replication role.
  - Includes a small custom resource Lambda to pick a default subnet that supports the selected instance type.
- Added: `CloudShell-eu.yaml`
  - Same as `CloudShell.yaml`, with an EU-region guard in the helper Lambda (eu-west-1/2/3, eu-central-1, eu-north-1, eu-south-1/2).
- Renamed (Deprecated): `C9.yaml` → `C9-Original-Do-Not-Use.yaml`
  - Marked as legacy; retained only for reference.
- Overhauled: `README.md`
  - Simplified step-by-step CloudShell instructions (download workshop.zip, deploy, connect via SSM port forwarding, use S3 bucket, cleanup).
  - Added workshop link and clear commands for students.

## Validation & Deployment (Profile: `chetz-playground`)
- Validated `CloudShell.yaml`:
  - `aws --profile chetz-playground cloudformation validate-template --template-body file://CloudShell.yaml`
- Dry-run change set for `ldav-cloudshell` (non-executed):
  - Created, inspected, and deleted a change set to confirm planned resources.
- Deployed `ldav-cloudshell` (global template) in `eu-central-1`:
  - `aws --profile chetz-playground --region eu-central-1 cloudformation deploy --template-file CloudShell.yaml --stack-name ldav-cloudshell --capabilities CAPABILITY_NAMED_IAM --parameter-overrides DbMasterPassword='REDACTED'`
  - Verified SSM: ran `AWS-RunShellScript` to confirm MySQL is active and port 3306 is listening.
  - Teardown: emptied S3 bucket and deleted the stack; waited for completion.
- Cross-region audit (EU + US):
  - Checked for stray EC2, SGs, Lambda, stacks, IAM, S3 across common EU/US regions; no leftovers found.
- Deployed `ldav-cloudshell-eu` (EU-only template) in `eu-central-1`:
  - `aws --profile chetz-playground --region eu-central-1 cloudformation deploy --template-file CloudShell-eu.yaml --stack-name ldav-cloudshell-eu --capabilities CAPABILITY_NAMED_IAM --parameter-overrides DbMasterPassword='REDACTED'`
  - Confirmed outputs: DbInstanceId, MigrationS3BucketName, etc.
  - User connected via SSM port forwarding; initial error stemmed from using the wrong CLI profile and was resolved.
  - Teardown: emptied bucket and deleted the stack; confirmed completion.

## Operational Notes
- Access pattern: CloudShell → SSM Session Manager (no public SSH). Optionally port-forward 3306 to CloudShell and connect with `mysql` client.
- Security group: MySQL allowed within default VPC CIDR (`172.31.0.0/16`); SSH allowed via EC2 Instance Connect prefix list but not required.
- IAM: Uses named roles/profiles; deployments require `--capabilities CAPABILITY_NAMED_IAM`.
- Costs: EC2 and related resources accrue cost until the stack is deleted.

## How To Use (Quick Steps)
1) Open CloudShell in your target region (e.g., eu-central-1).
2) Download workshop bundle:
   - `mkdir -p ~/workshop && cd ~/workshop && curl -L -o workshop.zip https://amazon-dynamodb-labs.com/assets/workshop.zip && unzip -q workshop.zip`
3) Deploy one of the templates:
   - Global: `aws cloudformation deploy --template-file CloudShell.yaml --stack-name ldav-cloudshell --capabilities CAPABILITY_NAMED_IAM --parameter-overrides DbMasterPassword='YOUR_SECURE_PASSWORD'`
   - EU-only: `aws --region eu-central-1 cloudformation deploy --template-file CloudShell-eu.yaml --stack-name ldav-cloudshell-eu --capabilities CAPABILITY_NAMED_IAM --parameter-overrides DbMasterPassword='YOUR_SECURE_PASSWORD'`
4) Get outputs and connect:
   - `DB_INSTANCE_ID=$(aws --output json cloudformation describe-stacks --stack-name ldav-cloudshell | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="DbInstanceId").OutputValue')`
   - Port forward: `aws ssm start-session --target "$DB_INSTANCE_ID" --document-name AWS-StartPortForwardingSession --parameters portNumber=3306,localPortNumber=3306`
   - Connect: `mysql -h 127.0.0.1 -P 3306 -u dbuser -p`
5) Cleanup:
   - `aws s3 rm s3://$BUCKET --recursive || true`
   - `aws cloudformation delete-stack --stack-name ldav-cloudshell && aws cloudformation wait stack-delete-complete --stack-name ldav-cloudshell`

## Workshop Link
- https://catalog.workshops.aws/dynamodb-labs/en-US/design-patterns/setup/user-account

## Status
- All changes pushed to `main`.
- Templates validated; deployments tested; teardown verified.
