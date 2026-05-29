# Per-VPC Endpoints: Deploy with CloudFormation

For architecture details see the [lab overview](../README.md).

## Prerequisites

Before deploying:

- AWS CLI v2 is installed.
- The CloudFormation deployer identity has the minimum IAM permissions
  required. See [iam-policy.json](iam-policy.json) for the reference policy.
- You have a Falcon API client ID and secret with `Sensor Download: Read`
  scope.

## Export credentials

```bash
export AWS_PROFILE=...  # AWS credentials for the target account
```

> **Note:** These labs avoid deploying to Falcon home Regions (`us-west-1`,
> `us-west-2`, `eu-central-1`) to showcase cross-region PrivateLink
> connectivity. Deploying in a home Region still works but does not exercise
> the cross-region path.

## Deploy the stack

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-privatelink-01 \
  --parameter-overrides \
    FalconCloud=us-2 \
    FalconClientId=YOUR-CLIENT-ID \
    FalconClientSecret=YOUR-CLIENT-SECRET \
    OwnerEmail=you@example.com \
  --capabilities CAPABILITY_NAMED_IAM
```

The CloudFormation stack provisions the VPC, endpoints, Route 53 private
hosted zone, IAM role, and EC2 instance. A Lambda custom resource
automatically downloads the latest Amazon Linux 2023 Falcon sensor RPM via the
Falcon API and stages it in S3 during deployment. The instance pulls the
sensor RPM from S3, waits for PrivateLink endpoint readiness, installs and
configures the Falcon sensor, and starts the service.

Expect about 1-2 minutes from stack completion to the host appearing in the
Falcon console.

## Verification

Run these checks after the stack completes. The instance needs 1-2 minutes to
boot, pull the sensor RPM from S3, and start the Falcon service before the SSM
session becomes available.

Get the instance ID from the stack outputs:

```bash
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name falcon-privatelink-01 \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)
```

Connect to the instance via SSM:

```bash
aws ssm start-session --target "$INSTANCE_ID"
```

Once connected, verify the sensor received a Falcon AID:

```bash
sudo /opt/CrowdStrike/falconctl -g --aid
```

A non-empty `aid` value confirms the sensor registered with the Falcon
cloud over PrivateLink. If the AID is not set after 5 minutes, troubleshoot
connectivity:

```bash
# Check the bootstrap log for installation errors
sudo cat /var/log/falcon-bootstrap.log

# Verify DNS resolves to private VPC IPs (not public addresses)
nslookup ts01-<cloud-slug>.cloudsink.net

# Verify TLS handshake succeeds over PrivateLink
curl -v https://ts01-<cloud-slug>.cloudsink.net:443 2>&1 | head -20

# Check sensor service status and recent logs
sudo systemctl status falcon-sensor --no-pager -l | tail -20
```

## Teardown

```bash
aws cloudformation delete-stack --stack-name falcon-privatelink-01
```

This removes all AWS resources created by the lab.
