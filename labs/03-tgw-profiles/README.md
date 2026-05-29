![CrowdStrike Logo (Light)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-light-mode.png#gh-light-mode-only)
![CrowdStrike Logo (Dark)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-dark-mode.png#gh-dark-mode-only)
# Architecture 03 - TGW + Route 53 Profiles

> **Lab environment, not production guidance.** These examples are learning
> deployments that emulate cloud-native architectures we have seen in customer
> environments. They demonstrate how PrivateLink connectivity to the Falcon
> platform can be leveraged in various topologies. Use them to understand the
> network requirements, validate connectivity, and inform your own design.

## What this lab demonstrates

Large AWS estates commonly centralize VPC-to-VPC connectivity through Transit
Gateway. This lab shows how that existing hub-and-spoke backbone can provide
Falcon connectivity to spoke workloads without requiring each spoke
to maintain its own PrivateLink endpoints.

A hub networking account creates an endpoint VPC with PrivateLink interface
endpoints for Falcon, a Transit Gateway, and a Route 53 Profile that shares
`cloudsink.net` DNS resolution with spoke VPCs. The spoke account keeps its own
VPC, attaches to the TGW, and routes Falcon sensor traffic to the hub endpoint
VPC over same-region TGW. From the hub, cross-region PrivateLink connects
privately to the Falcon endpoint service in the Falcon cloud home Region.

The spoke instance has:

- **Zero inbound exposure.** No SSH, no open ports. Management is SSM Session
  Manager only.
- **Minimal egress.** HTTPS to the hub endpoint VPC (via TGW) for Falcon
  traffic, plus local S3 and SSM gateway/interface endpoints. No broad
  `0.0.0.0/0` rule anywhere.
- **No local Falcon endpoints.** DNS resolves Falcon hostnames to the hub's
  endpoint ENIs via Route 53 Profile sharing.

## Table of contents

- [Prerequisites](#prerequisites)
- [What cross-region PrivateLink changes for TGW](#what-cross-region-privatelink-changes-for-tgw)
- [Route 53 Profiles](#route-53-profiles)
- [When to pick this](#when-to-pick-this)
- [What this deployment creates](#what-this-deployment-creates)
- [Sensor delivery](#sensor-delivery)
- [Choose a deployment option](#choose-a-deployment-option)

## Prerequisites

Before deploying:

- You have raised a support ticket with CrowdStrike and the hub account has
  been whitelisted for the consumer Region and Falcon cloud endpoint services.
- The AWS identities and relevant SCPs allow cross-region PrivateLink creation,
  including `vpce:AllowMultiRegion`.
- The hub and spoke AWS CLI profiles are authenticated.
- You know the spoke account's 12-digit account ID.
- You have a Falcon API client ID and secret with `Sensor Download: Read`
  scope.

## What cross-region PrivateLink changes for TGW

Before native cross-region PrivateLink, customers with a US-2 Falcon CID
typically created an endpoint VPC in `us-west-2` (the Falcon home Region),
connected workload Regions back to it with inter-region TGW peering or VPC
peering, propagated routes for the endpoint VPC CIDRs, and made DNS resolve the
Falcon hostnames to that remote endpoint VPC.

At scale, every additional workload Region increased the amount of customer
networking to manage: regional TGWs, peering attachments, route table entries,
return paths, security group or CIDR rules, and DNS sharing across Regions.
The resulting architecture worked, but the Falcon traffic path was shaped by
the Falcon home Region rather than by where the workloads actually ran.

With cross-region PrivateLink, the endpoint VPC can live in the same consumer
Region as the spokes. Spokes still route Falcon traffic to the hub over the
same-region TGW, but the customer no longer needs inter-region routing just to
reach the Falcon home Region. From the hub endpoint VPC, PrivateLink connects
privately to the remote Falcon endpoint service.

## Route 53 Profiles

Route 53 Profiles avoid a per-spoke private hosted zone association workflow.
The hub account creates the `cloudsink.net` private hosted zone, adds it to a
Route 53 Profile, and RAM-shares the Profile to the spoke account. The spoke
then associates the Profile with its VPC.

That gives spoke workloads the same private Falcon hostname resolution without
creating a Route 53 association authorization for every spoke VPC. Profiles
are regional, so a multi-Region rollout creates one Profile per consumer
Region.

## When to pick this

- You already run TGW as a hub-and-spoke backbone.
- Workloads must remain in workload-owned VPCs.
- A central networking account should own the PrivateLink endpoints for Falcon
  and DNS.
- You want one CrowdStrike account-whitelisting request for the hub account and
  Region, rather than one per spoke account.

Do not introduce TGW only for this guide if you do not already need
hub-and-spoke routing. The Shared VPC lab is simpler when workloads can run in
shared subnets.

## What this deployment creates

In the hub account:

- 1 endpoint VPC with two private subnets.
- 3 PrivateLink interface endpoints for Falcon (cross-region to the Falcon
  home Region).
- 3 SSM interface endpoints and 1 S3 gateway endpoint for hub-side support.
- 1 Route 53 private hosted zone for `cloudsink.net`.
- 1 S3 bucket with `falcon-sensor.rpm` and bucket policy granting access to the
  spoke instance role.
- 1 endpoint security group with CIDR-based ingress from the spoke VPC.
- 1 Transit Gateway with explicit hub and spoke route tables (spoke-to-spoke
  isolation).
- 1 hub VPC TGW attachment and hub route table entry for the spoke CIDR.
- 1 Route 53 Profile containing the `cloudsink.net` private hosted zone.
- RAM shares for the Transit Gateway and Route 53 Profile.

In the spoke account:

- 1 spoke VPC with two private subnets.
- 1 TGW VPC attachment to the RAM-shared TGW.
- 1 spoke route table entry for the hub CIDR.
- 3 local SSM interface endpoints (Session Manager access).
- 1 local S3 gateway endpoint.
- 1 spoke endpoints security group (ingress from instance SG).
- 1 Route 53 Profile association to the spoke VPC.
- 1 IAM role and instance profile for SSM and S3 read access.
- 1 private AL2023 EC2 instance with no public connectivity, manageable only via
  SSM, where the Falcon sensor is installed on first boot.

## Sensor delivery

> [!NOTE]
> This lab uses an S3 bucket (created in the hub account) and gateway endpoints
> to stage and install the Falcon sensor RPM at first boot. The spoke EC2
> instance downloads the RPM from the hub bucket over its local S3 gateway
> endpoint. This is one option among many. Organizations commonly deliver the
> sensor through golden image pipelines (baked into AMIs) or configuration
> management tools (Ansible, Chef, SSM Distributor). We chose S3 because it is
> the simplest method that works inside a VPC with no internet access, keeping
> the lab self-contained.

## Choose a deployment option

| Option | Guide |
|---|---|
| Terraform | [Deploy with Terraform](terraform/README.md) |
| CloudFormation | [Deploy with CloudFormation](cloudformation/README.md) |
