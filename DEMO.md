# DevOps Agent Demo

This demo walks through a scenario where a code change breaks the webserver, and the DevOps Agent investigates the issue automatically.

> Source:  https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/


## Prerequisites

- The CloudFormation stack (`DevOpsAgent-EC2-webserver-GitHub`) is deployed and working. See [README.md](README.md) for setup instructions.
- The Hello World page is accessible via the ALB DNS name:
  ```
  http://<ALBDnsName>/index.html
  ```
- DevOps Agent Space is configured in the same AWS account where the stack is deployed.

## Demo Steps

### 1. Verify the app is working

Open the ALB URL in a browser and confirm you see the Hello World page:

```
http://<ALBDnsName>/index.html
http://<ALBDnsName>/index.html?name=Demo
```

### 2. Introduce a breaking change in Kiro

Open `cloudformation-webserver.yaml` in Kiro and find the `ALBSGIngress` resource. Change the CIDR from `0.0.0.0/0` to `2.0.0.0/8`:

```yaml
  ALBSGIngress:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !GetAtt ALBSecurityGroup.GroupId
      IpProtocol: tcp
      FromPort: 80
      ToPort: 80
      CidrIp: 2.0.0.0/8    # <-- changed from 0.0.0.0/0
```

This restricts the ALB security group to only allow traffic from the `2.0.0.0/8` range, effectively blocking most users.

> Note: use a subnet mask like `/8` or smaller. A `/0` mask (e.g. `2.0.0.0/0`) covers all IPs and AWS will normalize it to `0.0.0.0/0`, which won't actually restrict anything.

### 3. Commit and push

In the Kiro terminal:

```bash
git add cloudformation-webserver.yaml
git commit -m "Minor cosmetic chnage"  --> (Not to give away the issue)
git push
```

### 4. Watch the GitHub Action deploy

Go to the repository Actions tab:

```
https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/actions
```

Wait for the "Deploy CloudFormation Stack" workflow to complete successfully.

### 5. Confirm the page is broken

Refresh the ALB URL in the browser:

```
http://<ALBDnsName>/index.html
```

The page should now time out or be unreachable, since your IP is no longer in the `0.0.0.0/0` range.

### 6. Start an investigation from DevOps Agent Space

Open DevOps Agent Space in the AWS account where the stack is deployed. Start a new investigation — the agent will:

- Detect the CloudWatch Synthetics canary (`webserver-heartbeat`) is failing
- Identify the security group change as the root cause
- Correlate the recent CloudFormation deployment with the failure
- Recommend reverting the CIDR back to `0.0.0.0/0`

## Cleanup

Revert the change to restore access:

```yaml
      CidrIp: 0.0.0.0/0
```

> Note: when restricting the CIDR for testing, use a subnet mask like `/8` or smaller (e.g. `2.0.0.0/8`). A `/0` mask covers all IPs and AWS will normalize any address with `/0` to `0.0.0.0/0`.

Commit, push, and let the GitHub Action redeploy.
