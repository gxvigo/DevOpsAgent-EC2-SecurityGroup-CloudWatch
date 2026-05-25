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

Open `cloudformation-webserver.yaml` in Kiro and find the `WebServerSGIngress` resource. Change `FromPort` and `ToPort` from `80` to `8080`:

```yaml
  WebServerSGIngress:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      GroupId: !Ref WebServerSecurityGroup
      IpProtocol: tcp
      FromPort: 8080    # <-- changed from 80
      ToPort: 8080      # <-- changed from 80
      SourceSecurityGroupId: !Ref ALBSecurityGroup
```

The ALB stays reachable from the internet, but it can no longer forward traffic to the EC2 instance on port 80. The ALB target group will mark the EC2 unhealthy within ~10 seconds.

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

Reload the ALB URL in the browser:

```
http://<ALBDnsName>/index.html
```

The ALB itself is reachable, but it can't forward to the EC2 instance, so it returns its built-in **503 Service Temporarily Unavailable** page within a fraction of a second. No more long browser spinner — the failure is immediate and obvious.

### 6. Start an investigation from DevOps Agent Space

Open DevOps Agent Space in the AWS account where the stack is deployed. Start a new investigation — the agent will:

- Detect the CloudWatch Synthetics canary (`webserver-heartbeat`) is failing
- Identify the security group change as the root cause
- Correlate the recent CloudFormation deployment with the failure
- Recommend reverting the CIDR back to `<whatsmyip>/32`

## Cleanup

Revert the change to restore access:

```yaml
  WebServerSGIngress:
    Properties:
      FromPort: 80
      ToPort: 80
```

Commit, push, and let the GitHub Action redeploy.
