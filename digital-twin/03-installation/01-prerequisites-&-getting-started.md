# Getting Started With Digital Twin

## Digital Twin Starter Kit

When installing Digital Twin for the first time, Matera provides a Digital Twin Starter Kit to simplify deployment and help your team get up and running quickly.

The Starter Kit is a pre-built Infrastructure-as-Code (IaC) package that automates the deployment of a complete Digital Twin development environment in your AWS account.

Using the Starter Kit, your infrastructure team can deploy a fully functional Digital Twin environment in approximately two hours. It provisions all required AWS infrastructure including networking, Kubernetes, Kafka, databases, security services, and Digital Twin components (using Terraform).

The environment is intended for development, testing, and proof-of-concept activities. Once deployed, your team can adapt the infrastructure to align with your organization's networking, security, compliance, and operational standards before moving toward production.

---

## Deployment Approach

### Development Environment

The Starter Kit provides a proven foundation for evaluating Digital Twin while allowing your team to incorporate existing enterprise standards such as WAF, API Gateways, IAM, network policies, and other governance controls.

### Recommended Initial Deployment

For the initial deployment, we recommend using Amazon MSK (Managed Streaming for Apache Kafka), which is provisioned by the Starter Kit. This minimizes complexity and allows your team to deploy a working environment quickly.

If your organization standardizes on Confluent Kafka, you can migrate after the initial deployment has been validated.

---

## Prerequisites

Before deploying the Starter Kit, your organization should provide:

- A Linux workstation or Linux EC2 instance to run the deployment
- An AWS account with permissions to provision the required infrastructure
- A Route 53 hosted zone for the environment
- Access to the Digital Twin container images and Helm charts in Amazon ECR

The deployment typically takes 90–120 minutes, most of which is spent waiting for AWS services such as Amazon EKS, Amazon RDS, and Amazon MSK to provision.

---

## AWS Permissions

The AWS account used for deployment must be authorized to provision the following services:

| Area | Services |
|---|---|
| Networking | VPC, subnets, NAT / Internet Gateways, Route 53 |
| Security | ACM (TLS certificates), Secrets Manager, IAM |
| Compute | EKS (Kubernetes), EC2 (worker nodes), Lambda |
| Data | RDS Aurora (MySQL-compatible), MSK (Kafka) |
| State & registry | S3 + DynamoDB (Terraform backend), ECR (read access) |

> **Note:** Deploying the Starter Kit provisions AWS resources — including Amazon EKS, Amazon RDS, Amazon MSK, NAT Gateways, and Load Balancers — which will incur AWS charges while they are running.

---

## Required Tools

The following tools are required on the control machine:

| Tool | Version |
|---|---|
| Terraform | 1.8.4 (pinned) |
| Terragrunt | 0.55.1 (pinned) |
| AWS CLI | ≥ 2.0 |
| kubectl | 1.33 |
| just | ≥ 1.0 |
| jq | latest |
| Python | 3.12 |
| uv | latest |

The required tools can be installed automatically using the included `mise` configuration.

---

## What the Starter Kit Deploys

Once deployed, the Starter Kit provisions the following AWS infrastructure and Digital Twin components:

- Networking (VPC, subnets, Route 53)
- Amazon EKS
- Amazon MSK
- Amazon RDS Aurora (MySQL-compatible)
- ACM certificates
- Secrets Manager
- Keycloak
- Digital Twin application services
- Schema Registry
- Optional administration tools (KafkaUI and phpMyAdmin)
