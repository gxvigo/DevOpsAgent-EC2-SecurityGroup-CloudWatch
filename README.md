# Hello World Webserver — CloudFormation

A single CloudFormation template that deploys an Apache webserver on EC2, serves a "Hello World" page, and monitors error rates with CloudWatch.

## Architecture

| Resource | Purpose |
|---|---|
| EC2 Instance | Runs Apache httpd, serves `index.html` |
| Security Group | Allows inbound TCP port 80 from anywhere |
| IAM Role + Instance Profile | Grants the instance permission to push logs to CloudWatch |
| CloudWatch Log Group | Stores Apache error logs (7-day retention) |
| CloudWatch Metric Filter | Counts error-level entries in the log group |
| CloudWatch Alarm | Fires when errors exceed 5 in a 5-minute window |

All taggable resources carry the tag `application_id = app01`.

## Prerequisites

- AWS CLI installed and configured with credentials that can create IAM roles, EC2 instances, and CloudWatch resources.
- An existing VPC with a public subnet (the subnet must auto-assign public IPs or have an associated Elastic IP).
- The public subnet's route table must have a route to an Internet Gateway so the instance is reachable.

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `VpcId` | Yes | — | ID of the VPC (e.g. `vpc-0abc123`) |
| `PublicSubnetId` | Yes | — | ID of a public subnet inside that VPC |
| `InstanceType` | No | `t3.micro` | EC2 instance type |
| `LatestAmiId` | No | Amazon Linux 2023 (SSM lookup) | AMI to use |

## Deploy

```bash
aws cloudformation deploy \
  --template-file cloudformation-webserver.yaml \
  --stack-name hello-world-webserver \
  --parameter-overrides \
      VpcId=vpc-0abc123 \
      PublicSubnetId=subnet-0def456 \
  --capabilities CAPABILITY_IAM
```

> `CAPABILITY_IAM` is required because the template creates an IAM role.

## Outputs

After deployment completes, retrieve the outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name hello-world-webserver \
  --query "Stacks[0].Outputs"
```

| Output | Description |
|---|---|
| `PublicIp` | Public IP address of the EC2 instance |
| `HelloWorldUrl` | Full URL to test the app (e.g. `http://<ip>/index.html`) |

## Test

Open the `HelloWorldUrl` output in a browser or use curl:

```bash
curl http://<PublicIp>/index.html
```

You should see:

```
Hello World
```

## Monitoring

- Apache error logs are shipped to the CloudWatch Log Group `/webserver/<stack-name>/error_log`.
- The CloudWatch Alarm (`WebServerErrorAlarm`) triggers when more than 5 errors are recorded in a 5-minute period.
- Check alarm state:

```bash
aws cloudwatch describe-alarms \
  --alarm-names "<stack-name>-WebServerErrorAlarm-*"
```

## CI/CD — GitHub Actions

The workflow at `.github/workflows/deploy.yml` handles deployment and teardown.

### Required GitHub Secrets

| Secret | Description |
|---|---|
| `AWS_ROLE_ARN` | ARN of an IAM role configured for [GitHub OIDC](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) (e.g. `arn:aws:iam::123456789012:role/github-deploy`) |
| `VPC_ID` | Target VPC ID |
| `PUBLIC_SUBNET_ID` | Target public subnet ID |

### Triggers

- **Push to `main`** — automatically deploys the stack.
- **Manual dispatch** — go to Actions → "Deploy CloudFormation Stack" → Run workflow, and choose `deploy` or `destroy`.

### IAM Role Trust Policy

The role referenced by `AWS_ROLE_ARN` needs a trust policy that allows GitHub OIDC. Minimal example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:*"
        }
      }
    }
  ]
}
```

The role itself needs permissions for CloudFormation, EC2, IAM, CloudWatch, and Logs.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name hello-world-webserver
```
