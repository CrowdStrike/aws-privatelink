# Per-VPC Endpoints: Deploy with Terraform

For architecture details see the [lab overview](../README.md).

## Prerequisites

Before deploying:

- `terraform` CLI is installed.
- `uv` is installed and on your `PATH`.
- The Terraform identity has the minimum IAM permissions required. See
  [iam-policy.json](iam-policy.json) for the reference policy.

## Export credentials

```bash
# Required
export AWS_PROFILE=...                           # AWS credentials for the target account
export TF_VAR_falcon_client_id='...'             # Falcon API client ID (Sensor Download: Read)
export TF_VAR_falcon_client_secret='...'         # Falcon API client secret
export TF_VAR_falcon_cloud='us-2'                # Falcon cloud for this CID (us-1, us-2, eu-1)
export TF_VAR_owner_email='you@example.com'      # Owner tag for resource accountability
export TF_VAR_environment='dev'                  # Resource name prefix and environment tag

# Optional — modify only if needed, defaults are fine for most deployments
export TF_VAR_region='us-east-2'                 # Region where the lab is deployed
export TF_VAR_instance_type='t3.small'           # EC2 instance size for sensor hosts
export TF_VAR_ami_id='ami-0abcdef1234567890'     # Only set if the default is deprecated; must be Amazon Linux 2023
```

> **Note:** These labs avoid deploying to Falcon home Regions (`us-west-1`,
> `us-west-2`, `eu-central-1`) to showcase cross-region PrivateLink
> connectivity. Deploying in a home Region still works but does not exercise
> the cross-region path.

## Deploy the Terraform template

```bash
cd labs/01-per-vpc/terraform
terraform init
terraform apply
```

Terraform first downloads the sensor RPM from the Falcon API on your
workstation and stages it locally. It then provisions the infrastructure,
uploads the RPM to S3, and launches the private EC2 instance. The instance
waits for PrivateLink endpoint readiness, pulls the RPM from S3, configures
Falcon, and starts the sensor.

Expect about 1-2 minutes from apply completion to the host appearing in the
Falcon console.

## Verification

Run these checks after `terraform apply` completes. The instance needs 1-2
minutes to boot, pull the sensor RPM from S3, and start the Falcon service
before the SSM session becomes available.

Connect to the instance via SSM:

```bash
eval $(terraform output -json deployment | jq -r '.ssm_start_session_commands[0]')
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

This removes all AWS resources created by the lab.
