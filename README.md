# Scalable Web Application with ALB and Auto Scaling (EC2-Based)

**Production-grade, highly available web application architecture on AWS** — built with a multi-AZ VPC, Application Load Balancer, EC2 Auto Scaling, CloudFront, WAF, and Multi-AZ RDS.

[🚧 Feature request](../../issues/new?labels=enhancement&title=) | [🐛 Bug report](../../issues/new?labels=bug&title=) | [❓ Ask a question](../../issues/new?labels=question&title=)

---

## Table of Contents

- [Solution Overview](#solution-overview)
- [Architecture Diagram](#architecture-diagram)
- [Architecture Flow](#architecture-flow)
- [AWS Services Used](#aws-services-used)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Configure Variables](#2-configure-variables)
  - [3. Deploy the Infrastructure](#3-deploy-the-infrastructure)
  - [4. Validate the Deployment](#4-validate-the-deployment)
  - [5. Tear Down](#5-tear-down)
- [Configuration Notes](#configuration-notes)
- [Security](#security)
- [Cost Considerations](#cost-considerations)
---

## Solution Overview

This project deploys a **production-grade, three-tier web application** on AWS using EC2 instances inside a properly segmented VPC. The architecture spans **two Availability Zones** for high availability and uses an **Application Load Balancer (ALB)** and **Auto Scaling Group (ASG)** to distribute traffic and scale compute capacity automatically in response to demand.

Static assets are cached at the edge with **Amazon CloudFront**, and the origin is protected by **AWS WAF** using managed rules aligned with the OWASP Top 10. The database tier runs on **Amazon RDS in a Multi-AZ configuration**, providing automated failover to a standby replica in a second AZ. All compute and database resources live in private subnets with no direct internet access — operators reach instances securely through **AWS Systems Manager Session Manager**, eliminating the need for a bastion host. **Amazon Route 53** provides DNS resolution and health checks, and **Amazon CloudWatch + SNS** provide monitoring, dashboards, and alerting across the stack.

This repository contains Infrastructure as Code (IaC) to deploy the full stack end-to-end, along with the architecture diagram and supporting documentation.

## Architecture Diagram

![Architecture Diagram](./architecture/aws_production_web_app_architecture.png)

## Architecture Flow

1. **User → Route 53** — DNS resolves the domain via an alias record pointing at CloudFront/ALB, with health checks to detect unhealthy endpoints.
2. **Route 53 → CloudFront + WAF** — Requests hit the CloudFront distribution, which caches static assets at edge locations and reduces latency. AWS WAF inspects requests against managed rule groups (OWASP Top 10) before they reach the origin.
3. **CloudFront → Application Load Balancer** — The ALB, deployed across public subnets in two AZs, performs Layer 7 routing and distributes traffic to healthy targets.
4. **ALB → Public Subnets (AZ-a / AZ-b)** — Each public subnet hosts an ALB node and a NAT Gateway, allowing outbound internet access for resources in the private subnets.
5. **Public Subnets → Private App Subnets** — Traffic is forwarded to EC2 instances managed by an Auto Scaling Group, launched from a shared Launch Template, one ASG spanning both AZs for redundancy.
6. **Private App Subnets → Private DB Subnets** — Application instances connect to an RDS primary instance in AZ-a, which synchronously replicates to a standby instance in AZ-b for automated failover.
7. **Operational access** — AWS Systems Manager Session Manager provides shell access to EC2 instances without a bastion host or open SSH ports.
8. **Observability** — Amazon CloudWatch collects metrics/logs and drives alarms; Amazon SNS delivers notifications to operators.
9. **Security & scaling controls** — Security Groups and Network ACLs enforce layered access control between every tier, and target tracking / step scaling policies adjust ASG capacity automatically based on load.

## AWS Services Used

| Service | Role in this Architecture |
|---|---|
| **Amazon VPC** | Public & private subnets across 2 AZs, route tables, NAT Gateways, Security Groups, NACLs |
| **Amazon EC2 + Auto Scaling** | Application compute via Launch Template; target tracking & step scaling policies |
| **Elastic Load Balancing (ALB)** | Layer 7 routing, listener rules, target group health checks |
| **AWS WAF** | Managed rule groups aligned with OWASP Top 10, attached to CloudFront |
| **Amazon CloudFront** | Edge caching for static assets, latency reduction, HTTPS termination |
| **Amazon RDS (Multi-AZ)** | MySQL/PostgreSQL primary + synchronous standby replica with automated failover |
| **Amazon Route 53** | Alias record to CloudFront/ALB, DNS health checks |
| **AWS Systems Manager** | Session Manager for bastion-free, auditable EC2 access |
| **Amazon CloudWatch + SNS** | Dashboards, alarms, and operator notifications |

## Repository Structure

```
.
├── architecture/
│   └── aws_production_web_app_architecture.png   # Architecture diagram
├── infrastructure/
│   ├── vpc/                 # VPC, subnets, route tables, NAT Gateways
│   ├── security/            # Security Groups, NACLs, WAF rules
│   ├── compute/             # Launch Template, Auto Scaling Group, scaling policies
│   ├── load-balancing/      # ALB, listeners, target groups
│   ├── cdn/                 # CloudFront distribution
│   ├── database/            # RDS Multi-AZ instance, subnet groups, parameter groups
│   ├── dns/                 # Route 53 hosted zone, alias records, health checks
│   └── monitoring/          # CloudWatch dashboards, alarms, SNS topics
├── scripts/
│   └── deploy.sh            # Convenience deployment script
├── docs/
│   └── runbook.md           # Operational runbook (failover, scaling, patching)
└── README.md
```

> Adjust this tree to match your actual IaC tool (Terraform modules, AWS CDK stacks, or CloudFormation templates).

## Prerequisites

- An AWS account with permissions to create VPC, EC2, ELB, RDS, CloudFront, WAF, Route 53, IAM, Systems Manager, and CloudWatch resources
- [AWS CLI](https://aws.amazon.com/cli/) v2, configured with a named profile
- Infrastructure as Code tooling of your choice:
  - [Terraform](https://developer.hashicorp.com/terraform) ≥ 1.5, **or**
  - [AWS CDK](https://aws.amazon.com/cdk/) with Node.js 20.x, **or**
  - AWS CloudFormation templates (no additional tooling required)
- A registered domain (or existing Route 53 hosted zone) if you want to use custom DNS
- An ACM certificate in `us-east-1` if attaching HTTPS to CloudFront

## Deployment

### 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/scalable-web-app-alb-asg.git
cd scalable-web-app-alb-asg
export MAIN_DIRECTORY=$PWD
```

### 2. Configure Variables

Copy the example variables file and set values for your environment (VPC CIDR, instance type, database engine/size, domain name, alert email, etc.):

```bash
cp infrastructure/terraform.tfvars.example infrastructure/terraform.tfvars
```

Key variables to review:

| Variable | Description | Example |
|---|---|---|
| `vpc_cidr` | CIDR block for the VPC | `10.0.0.0/16` |
| `availability_zones` | Two AZs to deploy across | `["us-east-1a", "us-east-1b"]` |
| `instance_type` | EC2 instance type for the ASG | `t3.medium` |
| `asg_min` / `asg_max` / `asg_desired` | Auto Scaling capacity bounds | `2 / 6 / 2` |
| `db_engine` | RDS engine | `postgres` or `mysql` |
| `db_instance_class` | RDS instance size | `db.t3.medium` |
| `domain_name` | Domain served via Route 53/CloudFront | `app.example.com` |
| `alert_email` | SNS subscription for CloudWatch alarms | `ops@example.com` |

### 3. Deploy the Infrastructure

```bash
cd $MAIN_DIRECTORY/infrastructure
terraform init
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars
```

> If using AWS CDK or CloudFormation instead, replace this step with `cdk deploy` or `aws cloudformation deploy` against the templates in `infrastructure/`.

### 4. Validate the Deployment

- Confirm the ALB target group shows all targets as `healthy`.
- Confirm the RDS instance shows a Multi-AZ standby in a second AZ.
- Visit the CloudFront/Route 53 domain and confirm the application loads.
- Confirm CloudWatch alarms and the SNS subscription are active.
- Connect to an instance via Systems Manager to confirm bastion-free access:

```bash
aws ssm start-session --target <instance-id> --profile <PROFILE_NAME>
```

### 5. Tear Down

```bash
cd $MAIN_DIRECTORY/infrastructure
terraform destroy -var-file=terraform.tfvars
```

> Remember to empty/delete any S3 buckets used for CloudFront logs or ALB access logs before destroying, as non-empty buckets can block teardown.

## Configuration Notes

- **Subnet layout**: Each AZ has one public subnet (ALB node + NAT Gateway), one private app subnet (ASG/EC2), and one private DB subnet (RDS).
- **Scaling policies**: Target tracking (e.g., average CPU utilization) is the default; step scaling can be layered on for faster response to traffic spikes.
- **Failover**: RDS Multi-AZ failover is automatic and transparent to the application via the RDS endpoint — no application-level changes are needed during failover.
- **Access model**: There is intentionally no bastion host or public SSH access; all administrative access goes through Systems Manager Session Manager, which is logged and IAM-controlled.
- **Edge security**: WAF rules should be evaluated in "count" mode first in non-production environments before switching to "block" mode.

## Security

- No inbound SSH/RDP is opened to the internet; all instances sit in private subnets.
- Security Groups follow least-privilege, tier-to-tier rules (ALB → App, App → DB only).
- NACLs provide a secondary layer of subnet-level access control.
- WAF managed rule groups mitigate common web exploits (OWASP Top 10).
- Systems Manager Session Manager provides auditable, IAM-authenticated shell access without exposing SSH.
- Secrets (DB credentials) should be stored in AWS Secrets Manager or SSM Parameter Store rather than hardcoded in IaC variables.

If you discover a security issue with this repository, please open an issue or, for sensitive reports, follow the process in `SECURITY.md`.

## Cost Considerations

This architecture provisions billable resources, including but not limited to: NAT Gateways (per-AZ), EC2 instances, an Application Load Balancer, RDS Multi-AZ (double the single-instance cost), and CloudFront data transfer. Review the [AWS Pricing Calculator](https://calculator.aws/) before deploying to production, and remember to tear down resources after testing to avoid ongoing charges.


