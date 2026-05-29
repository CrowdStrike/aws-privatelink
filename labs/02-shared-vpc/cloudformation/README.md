# Shared VPC: Deploy with CloudFormation

For architecture details see the [lab overview](../README.md).

## Prerequisites

- AWS CLI v2 installed and on your `PATH`.
- The hub and spoke AWS CLI profiles are authenticated.
- The hub account CloudFormation identity has the minimum IAM permissions
  required. See [iam-policy-hub.json](iam-policy-hub.json) for the reference
  policy.
- The spoke account CloudFormation identity has the minimum IAM permissions
  required. See [iam-policy-spoke.json](iam-policy-spoke.json) for the
  reference policy.

## Deploy the hub stack

Deploy the hub stack first. This creates the VPC, endpoints, DNS, bucket,
and RAM share. The stack automatically downloads the sensor RPM via a Lambda
custom resource during deployment.

```bash
aws cloudformation deploy \
  --profile your-hub-profile \
  --template-file template-hub.yaml \
  --stack-name falcon-privatelink-02-hub \
  --parameter-overrides \
    FalconCloud=us-2 \
    FalconClientId=YOUR-FALCON-CLIENT-ID \
    FalconClientSecret=YOUR-FALCON-CLIENT-SECRET \
    OwnerEmail=you@example.com \
    SpokeAccountId=111122223333 \
  --capabilities CAPABILITY_NAMED_IAM
```

> **Note:** These labs avoid deploying to Falcon home Regions (`us-west-1`,
> `us-west-2`, `eu-central-1`) to showcase cross-region PrivateLink
> connectivity. Deploying in a home Region still works but does not exercise
> the cross-region path.

## Deploy the spoke stack

Retrieve the outputs from the hub stack and pass them as parameters to the
spoke stack, including the CID that was extracted by the hub stack.

```bash
# Get hub stack outputs
VPC_ID=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' \
  --output text)

SUBNETS=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`SubnetIds`].OutputValue' \
  --output text)

SUBNET_1=$(echo $SUBNETS | cut -d',' -f1)
SUBNET_2=$(echo $SUBNETS | cut -d',' -f2)

ENDPOINTS_SG=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`EndpointsSgId`].OutputValue' \
  --output text)

BUCKET=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`SensorBucketName`].OutputValue' \
  --output text)

S3_PREFIX_LIST=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`S3PrefixListId`].OutputValue' \
  --output text)

CID=$(aws cloudformation describe-stacks \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub \
  --query 'Stacks[0].Outputs[?OutputKey==`FalconCID`].OutputValue' \
  --output text)

# Deploy spoke stack
aws cloudformation deploy \
  --profile your-spoke-profile \
  --template-file template-spoke.yaml \
  --stack-name falcon-privatelink-02-spoke \
  --parameter-overrides \
    HubVpcId=$VPC_ID \
    SubnetId1=$SUBNET_1 \
    SubnetId2=$SUBNET_2 \
    EndpointsSgId=$ENDPOINTS_SG \
    SensorBucketName=$BUCKET \
    S3PrefixListId=$S3_PREFIX_LIST \
    FalconCloud=us-2 \
    FalconCID=$CID \
  --capabilities CAPABILITY_NAMED_IAM
```

Expect about 3-5 minutes from stack creation to the spoke host appearing
in the Falcon console.

## Verification

Run these checks after the spoke stack completes. The instance needs 1-2
minutes to boot, pull the sensor RPM from S3, and start the Falcon service
before the SSM session becomes available.

Connect to the instance via SSM:

```bash
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --profile your-spoke-profile \
  --stack-name falcon-privatelink-02-spoke \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue' \
  --output text)

aws ssm start-session --target $INSTANCE_ID --profile your-spoke-profile
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

Delete the spoke stack first, then the hub stack.

```bash
aws cloudformation delete-stack \
  --profile your-spoke-profile \
  --stack-name falcon-privatelink-02-spoke

aws cloudformation wait stack-delete-complete \
  --profile your-spoke-profile \
  --stack-name falcon-privatelink-02-spoke

aws cloudformation delete-stack \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub

aws cloudformation wait stack-delete-complete \
  --profile your-hub-profile \
  --stack-name falcon-privatelink-02-hub
```

Before destroying the hub stack, confirm no unmanaged spoke ENIs are
still attached to the shared subnets.
