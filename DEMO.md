# DevOps Agent Demo

This demo walks through a scenario where a code change breaks the webserver, and the DevOps Agent investigates the issue automatically.

> Source:  https://github.com/gxvigo/DevOpsAgent-EC2-SecurityGroup-CloudWatch/


## Prerequisites

- The CloudFormation stack (`DevOpsAgent-EC2-webserver-GitHub`) is deployed and working. See [README.md](README.md) for setup instructions.
- The Hello World page is accessible via the ALB DNS name:
  ```
  http://<ALBDnsName>/index.html
  ```
- Before running the demo, verify that the ALB security group CIDR includes your public IP address. Find your IP at [https://www.whatismyip.com/](https://www.whatismyip.com/) and confirm it falls within the `0.0.0.0/0` range (which allows all IPs). If the CIDR has been previously restricted, update `ALBSGIngress` in the template to include your IP (e.g. `150.107.175.244/32`) from <whatsmyip>
- DevOps Agent Space is configured in the same AWS account where the stack is deployed.

## Demo Steps

### 1. Verify the app is working

Open the ALB URL in a browser and confirm you see the Workshop welcome page with the agenda. Leave this tab open for the rest of the demo — the page polls the ALB every few seconds in the background, so the audience will see the state change live.

```
http://<ALBDnsName>/index.html
```

### 2. Introduce a breaking change in Kiro

Open `cloudformation-webserver.yaml` in Kiro and find the `ALBSGIngress` resource. Change the CIDR from `<whatsmyip>/32` to `2.0.0.0/8`:

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

> Note: use a subnet mask like `/8` or smaller. A `/0` mask (e.g. `2.0.0.0/0`) covers all IPs and AWS will normalize it to `<whatsmyip>/32`.

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

Switch back to the browser tab from step 1. Within a few seconds the page detects it can no longer reach the ALB and a full-screen red **"Webserver unreachable"** overlay drops in, including how long ago the last successful check was. No refresh required.

```
http://<ALBDnsName>/index.html
```

If you opened a fresh tab instead, the page will simply fail to load.

### 6. Start an investigation from DevOps Agent Space

Open DevOps Agent Space in the AWS account where the stack is deployed. Start a new investigation — the agent will:

- Detect the CloudWatch Synthetics canary (`webserver-heartbeat`) is failing
- Identify the security group change as the root cause
- Correlate the recent CloudFormation deployment with the failure
- Recommend reverting the CIDR back to `<whatsmyip>/32`

## Cleanup

Revert the change to restore access:

```yaml
      CidrIp: <whatsmyip>/32
```

> Note: when restricting the CIDR for testing, use a subnet mask like `/8` or smaller (e.g. `2.0.0.0/8`). A `/0` mask covers all IPs and AWS will normalize any address with `/0` to `0.0.0.0/0`.

Commit, push, and let the GitHub Action redeploy.
