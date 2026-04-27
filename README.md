# app01 — Hello World Webserver

> Source: [https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/](https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/)

A CloudFormation template that deploys an Apache webserver on EC2 behind an Application Load Balancer, serves a personalizable "Hello" html page, and monitors availability with a CloudWatch Synthetics canary.

## Architecture

| Resource | Purpose |
|---|---|
| VPC + Subnets | Isolated network with 2 public subnets (ALB) and 1 private subnet (EC2) |
| Internet Gateway | Provides internet access for the public subnets |
| NAT Gateway | Allows the EC2 instance in the private subnet to reach the internet |
| Application Load Balancer | Internet-facing ALB that routes HTTP traffic to the EC2 instance |
| ALB Security Group | Allows inbound TCP ports 80 and 443 from the internet (`0.0.0.0/0`) |
| ALB Target Group | Registers the EC2 instance with health checks on `/index.html` |
| ALB Listener | HTTP redirects to HTTPS; HTTPS authenticates via Cognito then forwards to the target group |
| EC2 Instance | Runs Apache httpd in a private subnet, serves `index.html` |
| EC2 Security Group | Allows inbound TCP port 80 from the ALB security group only |
| IAM Role + Instance Profile | Grants the instance permission to push logs to CloudWatch |
| CloudWatch Log Groups | Stores Apache access and error logs (7-day retention) |
| Synthetics Canary | Hits the ALB endpoint every 1 minute and verifies a 200 response |
| CloudWatch Alarm | Fires when the canary success rate drops below 100% |
| S3 Bucket | Stores canary artifacts (auto-expires after 7 days) |
| Cognito User Pool | Provides authentication for the ALB via hosted UI |

All taggable resources carry the tag `application_id = app01`.

## Prerequisites

- An IAM user (or role) with the permissions defined in `iam-policy-infra.json` and `iam-policy-monitoring.json` must be created before deploying the stack. To create the user and attach both policies:

```bash
aws iam create-user --user-name devops-user

aws iam create-policy \
  --policy-name devops-policy-infra \
  --policy-document file://iam-policy-infra.json

aws iam create-policy \
  --policy-name devops-policy-monitoring \
  --policy-document file://iam-policy-monitoring.json

aws iam attach-user-policy \
  --user-name devops-user \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/devops-policy-infra

aws iam attach-user-policy \
  --user-name devops-user \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/devops-policy-monitoring

aws iam create-access-key --user-name devops-user
```

- AWS CLI installed and configured with the above user's credentials.

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `CertificateArn` | Yes | — | ACM certificate ARN for HTTPS |
| `InstanceType` | No | `t3.micro` | EC2 instance type |
| `LatestAmiId` | No | `ami-0c3389a4fa5bddaad` | EC2 AMI ID (pinned to avoid uncontrolled instance replacement) |

## Deploy

The template creates its own VPC, subnets, Internet Gateway, and NAT Gateway — no existing networking required.

```bash
aws cloudformation deploy \
  --template-file cloudformation-webserver.yaml \
  --stack-name DevOpsAgent-EC2-webserver-GitHub \
  --parameter-overrides \
      CertificateArn=arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERTIFICATE_ID> \
  --capabilities CAPABILITY_IAM
```

> `CAPABILITY_IAM` is required because the template creates an IAM role.

## Outputs

After deployment completes, retrieve the outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name DevOpsAgent-EC2-webserver-GitHub \
  --query "Stacks[0].Outputs"
```

| Output | Description |
|---|---|
| `PrivateIp` | Private IP address of the EC2 instance |
| `ALBDnsName` | DNS name of the Application Load Balancer |
| `HelloWorldUrl` | Full URL to test the app via the ALB |
| `CognitoSignUpUrl` | Cognito hosted UI for user self-registration |

## Post-Deploy: Create a Cognito User

After the stack is deployed, create at least one user in the Cognito User Pool to access the app:

```bash
aws cognito-idp admin-create-user \
  --user-pool-id <UserPoolId> \
  --username user@example.com \
  --temporary-password TempPass1 \
  --user-attributes Name=email,Value=user@example.com Name=email_verified,Value=true \
  --region us-east-1
```

Find the User Pool ID:

```bash
aws cognito-idp list-user-pools --max-results 10 \
  --query "UserPools[?Name=='devops-webserver-users'].Id"
```

Alternatively, users can self-register using the `CognitoSignUpUrl` from the stack outputs.

## Test

The page accepts an optional `name` query parameter to personalize the greeting. It defaults to "World" if not provided.

```bash
# Default greeting
curl http://<ALBDnsName>/index.html
# → Hello World

# Custom greeting
curl http://<ALBDnsName>/index.html?name=Kiro
# → Hello Kiro
```

Open in a browser:
- `http://<ALBDnsName>/index.html` → Hello World
- `http://<ALBDnsName>/index.html?name=Alice` → Hello Alice

## Monitoring

- Apache access logs are shipped to `/webserver/<stack-name>/access_log`.
- Apache error logs are shipped to `/webserver/<stack-name>/error_log`.
- A CloudWatch Synthetics canary (`webserver-heartbeat`) hits the ALB endpoint every 1 minute and verifies a 200 response.
- The `CanaryFailedAlarm` fires when the canary success rate drops below 100%.
- Check canary status:

```bash
aws synthetics get-canary --name webserver-heartbeat
```

## CI/CD — GitHub Actions

The workflow at `.github/workflows/deploy.yml` handles deployment and teardown.

### Required GitHub Secrets

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key ID |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret access key |
| `CERTIFICATE_ARN` | ACM certificate ARN for HTTPS |



### Triggers

- **Push to `main`** — automatically deploys the stack when `cloudformation-webserver.yaml` or `.github/workflows/deploy.yml` are changed. Other file changes (README, docs, etc.) do not trigger a deployment.
- **Manual dispatch** — go to Actions → "Deploy CloudFormation Stack" → Run workflow, and choose `deploy` or `destroy`.

The IAM user needs the permissions defined in `iam-policy-infra.json` and `iam-policy-monitoring.json`.

## AMI Management

The `LatestAmiId` parameter is pinned to a specific AMI ID to prevent uncontrolled EC2 instance replacement during deployments. Since `ImageId` is an immutable property on `AWS::EC2::Instance`, any AMI change forces CloudFormation to replace the instance.

To update the AMI intentionally:

```bash
# Get the current latest Amazon Linux 2023 AMI
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text
```

Then update the `LatestAmiId` default in `cloudformation-webserver.yaml` and the value in `.github/workflows/deploy.yml`.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name DevOpsAgent-EC2-webserver-GitHub
```
