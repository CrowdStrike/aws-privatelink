![CrowdStrike Logo (Light)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-light-mode.png#gh-light-mode-only)
![CrowdStrike Logo (Dark)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-dark-mode.png#gh-dark-mode-only)
# Architecture 02 - Shared VPC

> **Lab environment, not production guidance.** These examples are learning
> deployments that emulate cloud-native architectures we have seen in customer
> environments. They demonstrate how PrivateLink connectivity to the Falcon
> platform can be leveraged in various topologies. Use them to understand the
> network requirements, validate connectivity, and inform your own design.

## What this lab demonstrates

Instead of every spoke account owning its own VPC and PrivateLink endpoints
(Lab 1 pattern), this lab centralizes Falcon connectivity in a single
shared VPC. A hub account creates the VPC, PrivateLink endpoints, DNS, and
sensor bucket. A spoke account launches instances directly into RAM-shared
subnets inside that VPC, inheriting the hub's `cloudsink.net` private hosted
zone and endpoint access automatically.

This pattern proves three things:

- **Centralized PrivateLink management.** One hub account hosts all
  Falcon infrastructure. Only this account needs a CrowdStrike
  whitelisting request, regardless of how many spoke accounts use it.
- **Cross-account security group references.** Because the spoke ENI lives
  in the hub's VPC, the endpoint security group can reference the spoke
  instance's SG directly, even though the two resources are owned by different
  accounts.
- **Inherited DNS resolution.** Route 53 private hosted zones are VPC-scoped.
  Any ENI in the VPC, including RAM-shared spoke instances, resolves
  `*.cloudsink.net` to private endpoint IPs without additional DNS
  configuration in the spoke account.

The lab deploys one hub account and one spoke account in one consumer
Region (`us-east-2` by default). The RAM subnet share is same-region; deploy
one shared VPC per consumer Region if you need this pattern in multiple
Regions.

## Table of contents

- [Prerequisites](#prerequisites)
- [When to pick this](#when-to-pick-this)
- [What this deployment creates](#what-this-deployment-creates)
- [Sensor delivery](#sensor-delivery)
- [Choose a deployment option](#choose-a-deployment-option)

## Prerequisites

Before deploying:

- **AWS RAM sharing is enabled in your Organization.** The management account
  must have run `aws ram enable-sharing-with-aws-organization` at least once.
  VPC subnets can only be shared via RAM between accounts in the same
  Organization with this setting active. This is a one-time operation.
- You have raised a support ticket with CrowdStrike and the **hub account**
  has been whitelisted for the consumer Region and Falcon cloud endpoint
  services. Spoke accounts do not need separate whitelisting.
- The hub and spoke accounts are in the **same AWS Organization**.
- The AWS identities and relevant SCPs allow cross-region PrivateLink creation,
  including `vpce:AllowMultiRegion`.
- You know the spoke account's 12-digit account ID.
- You have a Falcon API client ID and secret with `Sensor Download: Read`
  scope.

## When to pick this

This is the right topology when:

- **Multi-account workload isolation.** You have multiple AWS accounts running
  workloads that need Falcon sensor connectivity, and centralizing PrivateLink
  management reduces operational overhead compared to per-account endpoints.
  Example: a platform team manages 20 spoke accounts and wants one
  CrowdStrike whitelisting request instead of 20.
- **Shared networking model.** Your organization's network architecture allows
  spoke accounts to launch instances in centrally managed subnets via AWS
  RAM. Example: a security team owns the VPC and controls egress rules, while
  application teams own the EC2 instances running inside it.
- **Same-region workloads.** All consumer workloads are in the same AWS region
  as the shared VPC. RAM subnet sharing is regional; multi-region deployments
  need one shared VPC per region.

Do **not** use this topology when:

- Workload teams must own their VPCs and cannot accept shared subnets.
- You need to serve multiple consumer regions from one hub VPC (not possible;
  RAM sharing is regional).
- TGW-based routing is required for Falcon traffic (see Lab 3).

## What this deployment creates

In the hub account:

- 1 VPC with two private subnets (AZs auto-derived), no internet gateway, no
  NAT gateway.
- 3 PrivateLink interface endpoints for Falcon.
- 3 SSM interface endpoints for Session Manager access.
- 1 S3 gateway endpoint.
- 1 Route 53 private hosted zone for `cloudsink.net` with alias records
  pointing at the PrivateLink endpoint ENIs.
- 1 S3 bucket containing `falcon-sensor.rpm` with a bucket policy allowing the
  spoke instance role to read the RPM cross-account.
- 1 endpoint security group with ingress from the spoke instance security
  group.
- 1 RAM resource share for the private subnets.

In the spoke account:

- 1 EC2 instance security group in the shared VPC (cross-account SG references
  work because both SGs are in the same VPC).
- 1 IAM role and instance profile granting SSM and S3 read access to the
  hub's sensor bucket.
- 1 private AL2023 EC2 instance launched into the shared subnet. CID and cloud
  are injected via user_data at launch time.

## Sensor delivery

> [!NOTE]
> This lab uses an S3 bucket in the hub account and cross-account IAM
> policies to stage and install the Falcon sensor RPM at first boot. This is
> one option among many. Organizations commonly deliver the sensor through
> golden image pipelines (baked into AMIs) or configuration management tools
> (Ansible, Chef, SSM Distributor). We chose S3 because it is the simplest
> method that works inside a VPC with no internet access, keeping the lab
> self-contained. The spoke instance pulls the RPM from the hub's bucket
> via the S3 gateway endpoint using cross-account bucket policy permissions.

## Choose a deployment option

| Option | Guide |
|---|---|
| Terraform | [Deploy with Terraform](terraform/README.md) |
| CloudFormation | [Deploy with CloudFormation](cloudformation/README.md) |
