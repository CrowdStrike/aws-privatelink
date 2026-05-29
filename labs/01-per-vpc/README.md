![CrowdStrike Logo (Light)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-light-mode.png#gh-light-mode-only)
![CrowdStrike Logo (Dark)](https://raw.githubusercontent.com/CrowdStrike/.github/main/assets/cs-logo-dark-mode.png#gh-dark-mode-only)

# Architecture 01 - Per-VPC endpoints

> **Lab environment, not production guidance.** These examples are learning
> deployments that emulate cloud-native architectures we have seen in customer
> environments. They demonstrate how PrivateLink connectivity to the Falcon
> platform can be leveraged in various topologies. Use them to understand the
> network requirements, validate connectivity, and inform your own design.

## What this lab demonstrates

The Falcon sensor needs very little network access to operate. This lab proves
that by deploying an EC2 instance into a fully isolated VPC with no internet
gateway, no NAT gateway, no public IP, and no inbound access. The sensor
installs, configures, and communicates with the Falcon cloud entirely
over AWS PrivateLink.

The instance has:

- **Zero inbound exposure.** No SSH, no open ports. Management is SSM Session
  Manager only.
- **Minimal egress.** HTTPS (443) to exactly three PrivateLink
  endpoints for Falcon (`sensor_proxy`, `download_server`, `upload_server`)
  and one S3 gateway endpoint. Nothing else. Security groups enforce this
  with no broad `0.0.0.0/0` rule anywhere.

Every company has its own architecture. This repository provides three labs
(per-VPC, shared VPC, TGW hub-and-spoke) that each solve the same problem,
getting the Falcon sensor connected over PrivateLink, within a different
cloud-native networking pattern. This first lab is the simplest: a single VPC
owning its own PrivateLink connection to the Falcon platform.

## Table of contents

- [Prerequisites](#prerequisites)
- [When to pick this](#when-to-pick-this)
- [What this deployment creates](#what-this-deployment-creates)
- [Sensor delivery](#sensor-delivery)
- [Choose a deployment option](#choose-a-deployment-option)

## Prerequisites

Before deploying:

- You have raised a support ticket with CrowdStrike and your AWS account has
  been whitelisted for the consumer Region and Falcon cloud endpoint services.
- The AWS identity and any relevant SCP allow cross-region PrivateLink
  creation, including `vpce:AllowMultiRegion`.
- You have a Falcon API client ID and secret with `Sensor Download: Read`
  scope.

## When to pick this

This is the right topology when:

- **Self-contained VPCs.** Each workload VPC owns its own networking and does
  not depend on a shared hub or transit gateway. Example: a small team running
  a handful of accounts where each account has one or two VPCs with Falcon
  deployed.
- **Simple DNS ownership.** The same account that creates the PrivateLink
  endpoints also owns the Route 53 private hosted zone. Example: a single-team
  product environment where one Terraform workspace controls VPC, DNS, and
  compute.
- **Fastest path to validation.** You want a working PrivateLink lab with the
  fewest moving parts to prove connectivity before designing a multi-account
  rollout. Example: a solutions architect building a proof-of-concept before
  presenting a shared-VPC or TGW design to a customer.

## What this deployment creates

Per deployment:

- 1 VPC with two private subnets (AZs auto-derived), no internet gateway, no
  NAT gateway.
- 3 PrivateLink interface endpoints for Falcon.
- 3 SSM interface endpoints for Session Manager access.
- 1 S3 gateway endpoint to download and stage the Falcon sensor for an
  instance that has no internet connectivity.
- 1 Route 53 private hosted zone for `cloudsink.net` with alias records
  pointing at the PrivateLink endpoint ENIs.
- 1 S3 bucket containing the Falcon sensor RPM.
- 1 IAM role and instance profile granting SSM and S3 access.
- 2 security groups: one for the instance (egress-only to endpoints and S3),
  one for the endpoints (ingress-only from the instance SG).
- 1 private AL2023 EC2 instance with no public connectivity, manageable only
  via SSM, where the Falcon sensor is deployed.

## Sensor delivery

> [!NOTE]
> This lab uses an S3 bucket and a gateway endpoint to stage and install the
> Falcon sensor RPM at first boot. This is one option among many. Organizations
> commonly deliver the sensor through golden image pipelines (baked into AMIs)
> or configuration management tools (Ansible, Chef, SSM Distributor). We chose
> S3 because it is the simplest method that works inside a VPC with no internet
> access, keeping the lab self-contained.

## Choose a deployment option

| Option | Guide |
|---|---|
| Terraform | [Deploy with Terraform](terraform/README.md) |
| CloudFormation | [Deploy with CloudFormation](cloudformation/README.md) |
