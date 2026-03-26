# app01 — Hello World Webserver

> Source: [https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/](https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/)

A CloudFormation template that deploys an Apache webserver on EC2 behind an Application Load Balancer, serves a personalizable "Hello" page, and monitors availability with a CloudWatch Synthetics canary.

## Architecture

| Resource | Purpose |
|---|---|
| Application Load Balancer | Internet-facing ALB that routes HTTP traffic to the EC2 instance |
| ALB Security Group | Allows inbound TCP port 80 from the internet |
| ALB Target Group | Registers the EC2 instance with health checks on `/index.html` |
| ALB Listener | Listens on port 80 and forwards to the target group |
| EC2 Instance | Runs Apache httpd, serves `index.html` |
| EC2 Security Group | Allows inbound TCP port 80 from the ALB security group only |
| Elastic IP | Static public IP attached to the EC2 instance |
| IAM Role + Instance Profile | Grants the instance permission to push logs to CloudWatch |
| CloudWatch Log Groups | Stores Apache access and error logs (7-day retention) |
| Synthetics Canary | Hits the ALB endpoint every 1 minute and verifies a 200 response |
| CloudWatch Alarm | Fires when the canary success rate drops below 100% |
| S3 Bucket | Stores canary artifacts (auto-expires after 7 days) |

All taggable resources carry the tag `application_id = app01`.

## Prerequisites

- An IAM user (or role) with the permissions defined in `iam-policy.json` must be created before deploying the stack. To create the user and attach the policy:

```bash
aws iam create-user --user-name devops-user
aws iam put-user-policy \
  --user-name devops-user \
  --policy-name DevOpsAgentDeployPolicy \
  --policy-document file://iam-policy.json
aws iam create-access-key --user-name devops-user
```

- AWS CLI installed and configured with the above user's credentials.
- An existing VPC with two public subnets in different Availability Zones (required by the ALB).
- Both subnets' route tables must have a route to an Internet Gateway.

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `VpcId` | Yes | — | ID of the VPC (e.g. `vpc-0abc123`) |
| `PublicSubnetId` | Yes | — | ID of a public subnet |
| `PublicSubnetId2` | Yes | — | ID of a second public subnet in a different AZ |
| `InstanceType` | No | `t3.micro` | EC2 instance type |
| `LatestAmiId` | No | Amazon Linux 2023 (SSM lookup) | AMI to use |

## Deploy

```bash
aws cloudformation deploy \
  --template-file cloudformation-webserver.yaml \
  --stack-name DevOpsAgent-EC2-webserver-GitHub \
  --parameter-overrides \
      VpcId=vpc-0abc123 \
      PublicSubnetId=subnet-0def456 \
      PublicSubnetId2=subnet-0ghi789 \
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
| `PublicIp` | Elastic IP address of the EC2 instance |
| `ALBDnsName` | DNS name of the Application Load Balancer |
| `HelloWorldUrl` | Full URL to test the app via the ALB |

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
| `VPC_ID` | Target VPC ID |
| `PUBLIC_SUBNET_ID` | First public subnet ID |
| `PUBLIC_SUBNET_ID_2` | Second public subnet ID (different AZ) |

### Triggers

- **Push to `main`** — automatically deploys the stack.
- **Manual dispatch** — go to Actions → "Deploy CloudFormation Stack" → Run workflow, and choose `deploy` or `destroy`.

The IAM user needs the permissions defined in `iam-policy.json`.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name DevOpsAgent-EC2-webserver-GitHub
```
