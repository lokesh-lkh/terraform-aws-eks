# Terraform AWS EKS — Full Stack Deployment

Production-grade Terraform infrastructure for deploying a complete **Amazon EKS cluster** running the RoboShop microservices application. Covers VPC networking, security groups, RDS, EKS control plane + managed node groups, Helm charts for 8 services, ALB ingress with TLS, and ACM certificates.

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │           AWS us-east-1                  │
                        │                                          │
  ┌──────────────┐      │  ┌──────────────────────────────────┐  │
  │ Terraform    │      │  │  EKS Cluster (roboshop)           │  │
  │ 00-vpc       │──────┼─►│  ├─ blue nodegroup (SPOT)           │  │
  │ 10-sg        │      │  │  ├─ green nodegroup (SPOT)          │  │
  │ 20-sg-rules  │      │  │  ├─ EBS/EFS CSI drivers            │  │
  │ 30-rds       │      │  │  │                                  │  │
  │ 60-eks       │──────┼─►│  │  Helm Charts:                     │  │
  │ 70-acm       │      │  │  │  cart, catalogue, user,           │  │
  │ 80-frontend  │      │  │  │  payment, shipping, frontend      │  │
  └──────────────┘      │  │                                  │  │
                        │  │  ALB Ingress (TLS via ACM)       │  │
                        │  └──────────────────────────────────┘  │
                        │                                          │
                        │  RDS (MySQL)  │  MongoDB  │  Redis     │
                        └─────────────────────────────────────────┘
```

## Directory Structure

```
terraform-aws-eks/
├── 00-vpc/              # VPC, subnets (public/private/db), IGW, NAT, route tables
├── 10-sg/               # EKS control plane + node security groups
├── 20-sg-rules/         # SG rules (ingress/egress) with sg_rules.yaml data
├── 30-rds/              # MySQL RDS instance + subnet group
├── 60-eks/              # EKS cluster + managed node groups + Helm app
│   ├── app/
│   │   ├── 01-storage-class/   # EBS StorageClass
│   │   ├── 02-databases/       # MongoDB, Redis, RabbitMQ manifests
│   │   ├── 03-backend/         # 5 microservice Helm charts
│   │   ├── 04-frontend/        # Frontend Helm chart + ALB ingress
│   │   └── 05-gateway/         # Gateway API resources
│   └── locals.tf, main.tf, variables.tf
├── 70-acm/              # ACM certificate for TLS
└── 80-frontend-alb/      # ALB + target group + listener rules
```

## Key Features

- **Blue-green deployment** — Two managed node groups (blue/green) using SPOT instances with auto-scaling (2-10 nodes). Rollback by swapping node group labels.
- **Multi-tier networking** — Separate public/private/database subnet tiers across 2 AZs, NAT gateway for private egress, security groups with least-privilege rules.
- **Helm charts with environment separation** — Each microservice has `values.yaml` (defaults), `values-dev.yaml`, `values-prod.yaml` for dev/prod image versions, replica counts, and config maps.
- **ALB ingress with TLS** — AWS Load Balancer Controller annotations, ACM certificate, HTTPS-only listener, `target-type: ip`.
- **Gateway API** — Kubernetes Gateway API v1.5.0 standard install with server-side apply.
- **EBS/EFS CSI drivers** — IAM policies attached to node groups for dynamic PV provisioning.

## Prerequisites

- Terraform >= 1.0, AWS Provider >= 4.0
- AWS credentials with EKS, VPC, RDS, ACM, EC2, IAM permissions
- `kubectl` and `helm` configured against the target cluster
- Existing VPC + private subnets (output from `00-vpc`)

## Quick Start

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

> **Note:** This repo has `errored.tfstate` files in some directories — these are state snapshots from failed runs. Run `terraform plan` from each directory individually to validate the config before applying.

## Related Practice

- `../terraform-aws-vpc/` — Standalone production-ready VPC module
- `../terraform-aws-workstation/` — EC2 workstation that auto-creates an EKS cluster
- `../terraform-aws-instance/` — Single EC2 provisioning
- `../terraform-aws-sg/` — Security group patterns
- `../roboshop-infra-dev/` — Parallel dev environment infrastructure
