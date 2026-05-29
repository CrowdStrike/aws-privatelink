![CrowdStrike Logo (Light)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-light-mode.png#gh-light-mode-only)
![CrowdStrike Logo (Dark)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-dark-mode.png#gh-dark-mode-only)
# TGW + Route 53 Profiles: Deploy with CloudFormation

For architecture details see the [lab overview](../README.md).

## Prerequisites

Before deploying:

- AWS CLI v2 is installed and on your `PATH`.
- The hub account identity has the minimum IAM permissions required. See
  [iam-policy-hub.json](iam-policy-hub.json) for the reference policy.
- The spoke account identity has the minimum IAM permissions required. See
  [iam-policy-spoke.json](iam-policy-spoke.json) for the reference policy.
- The hub and spoke AWS CLI profiles are authenticated.
- You have a Falcon API client ID and secret with `Sensor Download: Read`
  scope.

> **Note:** These labs avoid deploying to Falcon home Regions (`us-west-1`,
> `us-west-2`, `eu-central-1`) to showcase cross-region PrivateLink
> connectivity. Deploying in a home Region still works but does not exercise
> the cross-region path.

## Deploy the hub stack

The hub stack automatically downloads the latest Amazon Linux 2023 Falcon
sensor RPM from the Falcon API during deployment via a Lambda custom resource.

```bash
aws cloudformation deploy \
  --profile your-hub-profile \
  --template-file template-hub.yaml \
  --stack-name falcon-privatelink-03-hub \
  --parameter-overrides \
    FalconCloud=us-2 \
    FalconClientId=YOUR-CLIENT-ID \
    FalconClientSecret=YOUR-CLIENT-SECRET \
    OwnerEmail=you@example.com \
    SpokeAccountId=111122223333 \
  --capabilities CAPABILITY_NAMED_IAM
```

The hub stack creates:

- Endpoint VPC with PrivateLink endpoints for Falcon and SSM
- Transit Gateway with hub and spoke route tables
- Route 53 private hosted zone and Profile
- S3 bucket for the sensor RPM (automatically populated by Lambda)
- RAM shares for the TGW and Route 53 Profile

Wait for the stack to reach `CREATE_COMPLETE` before proceeding.

## Accept RAM shares

If both accounts are in the same AWS Organization and organization sharing is
enabled for RAM, the spoke account will automatically accept the TGW and Route
53 Profile shares.

If not, the spoke account must manually accept the RAM share invitations:

```bash
# List pending invitations
aws ram get-resource-share-invitations \
  --profile your-spoke-profile \
  --query 'resourceShareInvitations[?status==`PENDING`]'

# Accept each invitation
aws ram accept-resource-share-invitation \
  --profile your-spoke-profile \
  --resource-share-invitation-arn <arn-from-list>
```

## Deploy the spoke stack

Retrieve outputs from the hub stack:

```bash
TGW_ID=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`TgwId`].OutputValue' \
  --output text)

HUB_VPC_CIDR=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`HubVpcCidr`].OutputValue' \
  --output text)

PROFILE_ID=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`ProfileId`].OutputValue' \
  --output text)

BUCKET=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`SensorBucketName`].OutputValue' \
  --output text)

CID=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`FalconCID`].OutputValue' \
  --output text)
```

Deploy the spoke stack:

```bash
aws cloudformation deploy \
  --profile your-spoke-profile \
  --template-file template-spoke.yaml \
  --stack-name falcon-privatelink-03-spoke \
  --parameter-overrides \
    TgwId="$TGW_ID" \
    HubVpcCidr="$HUB_VPC_CIDR" \
    ProfileId="$PROFILE_ID" \
    SensorBucketName="$BUCKET" \
    FalconCloud=us-2 \
    FalconCID="$CID" \
  --capabilities CAPABILITY_NAMED_IAM
```

The spoke stack creates:

- Spoke VPC with TGW attachment
- Route table entry pointing hub CIDR to TGW
- Local SSM and S3 endpoints
- Route 53 Profile association
- IAM role for EC2 instance
- EC2 instance with Falcon sensor installed at first boot

Expect about 5-8 minutes from stack completion to the instance appearing in the
Falcon console. TGW attachment provisioning adds time compared with the first
two labs.

## Configure TGW route tables

After both stacks are deployed, configure the TGW route table associations and
propagations to enable hub-spoke traffic and spoke-to-spoke isolation.

Retrieve the spoke attachment ID and configure associations:

```bash
# Get spoke attachment ID
SPOKE_ATTACH_ID=$(aws ec2 describe-transit-gateway-attachments \
  --profile your-hub-profile \
  --filters "Name=transit-gateway-id,Values=$TGW_ID" \
            "Name=state,Values=available" \
            "Name=resource-type,Values=vpc" \
  --query 'TransitGatewayAttachments[?ResourceOwnerId==`'$SPOKE_ACCOUNT_ID'`].TransitGatewayAttachmentId' \
  --output text)

# Get hub and spoke route table IDs
HUB_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
  --profile your-hub-profile \
  --filters "Name=transit-gateway-id,Values=$TGW_ID" \
            "Name=tag:Name,Values=dev-hub-rt" \
  --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' \
  --output text)

SPOKE_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
  --profile your-hub-profile \
  --filters "Name=transit-gateway-id,Values=$TGW_ID" \
            "Name=tag:Name,Values=dev-spoke-rt" \
  --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' \
  --output text)

# Get hub attachment ID
HUB_ATTACH_ID=$(aws ec2 describe-transit-gateway-attachments \
  --profile your-hub-profile \
  --filters "Name=transit-gateway-id,Values=$TGW_ID" \
            "Name=state,Values=available" \
            "Name=resource-type,Values=vpc" \
  --query 'TransitGatewayAttachments[?ResourceOwnerId==`'$(aws sts get-caller-identity --profile your-hub-profile --query Account --output text)'`].TransitGatewayAttachmentId' \
  --output text)

# Associate spoke attachment with spoke route table
aws ec2 associate-transit-gateway-route-table \
  --profile your-hub-profile \
  --transit-gateway-route-table-id "$SPOKE_RT_ID" \
  --transit-gateway-attachment-id "$SPOKE_ATTACH_ID"

# Propagate spoke attachment to hub route table (hub learns spoke CIDR)
aws ec2 enable-transit-gateway-route-table-propagation \
  --profile your-hub-profile \
  --transit-gateway-route-table-id "$HUB_RT_ID" \
  --transit-gateway-attachment-id "$SPOKE_ATTACH_ID"

# Propagate hub attachment to spoke route table (spoke learns hub CIDR)
aws ec2 enable-transit-gateway-route-table-propagation \
  --profile your-hub-profile \
  --transit-gateway-route-table-id "$SPOKE_RT_ID" \
  --transit-gateway-attachment-id "$HUB_ATTACH_ID"
```

This configuration ensures spoke-to-spoke isolation: spokes can reach the hub
but not each other.

## Verification

After the spoke stack reaches `CREATE_COMPLETE`, retrieve the instance ID:

```bash
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --profile your-spoke-profile \
  --stack-name falcon-privatelink-03-spoke \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)
```

Connect to the instance via SSM Session Manager. The instance needs 2-3 minutes
to boot, pull the sensor from S3, and start the Falcon service before the
session becomes available.

```bash
aws ssm start-session \
  --profile your-spoke-profile \
  --target "$INSTANCE_ID"
```

Once connected, verify the sensor received a Falcon AID:

```bash
sudo /opt/CrowdStrike/falconctl -g --aid
```

A non-empty `aid` value confirms the sensor registered with the Falcon cloud
over PrivateLink. If the AID is not set after 5 minutes, troubleshoot
connectivity:

```bash
# Check the bootstrap log for installation errors
sudo cat /var/log/falcon-bootstrap.log

# Verify DNS resolves to private IPs in the hub endpoint VPC CIDR (10.70.x.x)
nslookup ts01-<cloud-slug>.cloudsink.net

# Verify TLS handshake succeeds over PrivateLink
curl -v https://ts01-<cloud-slug>.cloudsink.net:443 2>&1 | head -20

# Check sensor service status and recent logs
sudo systemctl status falcon-sensor --no-pager -l | tail -20
```

## Teardown

Delete the spoke stack first:

```bash
aws cloudformation delete-stack \
  --profile your-spoke-profile \
  --stack-name falcon-privatelink-03-spoke

aws cloudformation wait stack-delete-complete \
  --profile your-spoke-profile \
  --stack-name falcon-privatelink-03-spoke
```

Then delete the hub stack:

```bash
aws cloudformation delete-stack \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub

aws cloudformation wait stack-delete-complete \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub
```

If the hub stack delete fails due to the S3 bucket not being empty, manually
empty the bucket and re-run the delete command:

```bash
aws s3 rm "s3://$BUCKET" --recursive --profile your-hub-profile
aws cloudformation delete-stack --profile your-hub-profile \
  --stack-name falcon-privatelink-03-hub
```

Before deleting, confirm no unmanaged spokes are attached to the TGW or
associated with the shared Route 53 Profile. If a TGW-related delete fails
during dependency cleanup, wait a few minutes for AWS to finish cleaning up
attachment state, then re-run the delete command.
