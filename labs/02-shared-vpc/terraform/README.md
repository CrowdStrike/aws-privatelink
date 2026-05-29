# Shared VPC: Deploy with Terraform

For architecture details see the [lab overview](../README.md).

## Prerequisites

- `terraform` CLI installed and on your `PATH`.
- `uv` installed and on your `PATH`.
- The hub and spoke AWS CLI profiles are authenticated.
- The hub account Terraform identity has the minimum IAM permissions
  required. See [iam-policy-hub.json](iam-policy-hub.json) for the reference
  policy.
- The spoke account Terraform identity has the minimum IAM permissions
  required. See [iam-policy-spoke.json](iam-policy-spoke.json) for the
  reference policy.

## Export credentials

```bash
# Required
export TF_VAR_hub_profile='my-sso-hub'              # AWS SSO profile for the hub account
export TF_VAR_spoke_profile='my-sso-spoke'          # AWS SSO profile for the spoke account
export TF_VAR_spoke_account_id='111122223333'       # 12-digit AWS account ID for the spoke account
export TF_VAR_owner_email='you@example.com'         # Owner tag for resource accountability
export TF_VAR_falcon_client_id='...'                # Falcon API client ID (Sensor Download: Read)
export TF_VAR_falcon_client_secret='...'            # Falcon API client secret
export TF_VAR_falcon_cloud='us-2'                   # Falcon cloud for this CID (us-1, us-2, eu-1)

# Optional — defaults are fine for most deployments
export TF_VAR_region='us-east-2'                    # Region where the lab is deployed
export TF_VAR_instance_type='t3.small'              # EC2 instance size for sensor hosts
```

Refresh both profiles before applying:

```bash
aws sso login --profile "$TF_VAR_hub_profile"
aws sso login --profile "$TF_VAR_spoke_profile"
```

> **Note:** These labs avoid deploying to Falcon home Regions (`us-west-1`,
> `us-west-2`, `eu-central-1`) to showcase cross-region PrivateLink
> connectivity. Deploying in a home Region still works but does not exercise
> the cross-region path.

## Deploy

```bash
cd aws-privatelink/labs/02-shared-vpc/terraform
terraform init
terraform apply
```

A single Terraform state coordinates both accounts through provider aliases.
On first apply, Terraform will:

1. Run `scripts/fetch_sensor.py` through `uv run` to download the latest
   AL2023 sensor RPM and fetch your tenant CID.
2. Create the hub VPC, subnets, endpoints, private hosted zone, bucket,
   bucket policy, endpoint security group, and RAM share.
3. Create the spoke instance security group, IAM role, and EC2 test host in
   the shared subnet.
4. Install and start the Falcon sensor on first boot via user_data.

Expect about 3-5 minutes from `apply complete` to the spoke host appearing
in the Falcon console.

## Verification

Run these checks after `terraform apply` completes. The instance needs 1-2
minutes to boot, pull the sensor RPM from S3, and start the Falcon service
before the SSM session becomes available.

Connect to the instance via SSM:

```bash
eval "$(terraform output -json deployment | jq -r '.ssm_start_session_commands[0]') --profile $(terraform output -json deployment | jq -r '.spoke_profile')"
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
terraform destroy
```

Destroy removes the spoke account resources first, then the hub account
RAM share, endpoints, VPC, and bucket. Before destroying, confirm no unmanaged
spoke ENIs are still attached to the shared subnets.
