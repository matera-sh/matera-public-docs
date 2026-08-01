# QR Code Installation Guide

## Table of Contents

- [Overview](#overview)
- [Systems Architecture](#systems-architecture)
- [Architecture](#architecture)
- [Before You Begin](#before-you-begin)
- [Development vs. Production](#development-vs-production)
- [Installation](#installation)
  - [Step 1 – Create a Namespace](#step-1--create-a-namespace)
  - [Step 2 – Configure the Application](#step-2--configure-the-application)
  - [Step 3 – Deploy the Application](#step-3--deploy-the-application)
  - [Step 4 – Validate the Deployment](#step-4--validate-the-deployment)
- [Configuration Reference](#configuration-reference)
- [Security Considerations](#security-considerations)
- [Additional Resources](#additional-resources)

## Overview

This guide explains how to deploy Matera's Payment QR Code software in a Kubernetes environment using Helm.

The software enables users to generate, validate, and process ANSI X9.150-compliant Payment QR Codes for account-to-account payment initiation.

It supports the complete payment request lifecycle, including Payment Payload and Payment Notification flows. It protects message integrity and authenticity using JSON Web Signatures (JWS) and X9 Financial PKI trust principles.

## Systems Architecture

The Payment QR Code software integrates with several infrastructure components to provide authentication, persistence, and certificate management.

```
Client Applications (banking apps, portals, or other client systems)
        │
        ▼
X9 QR Code Service — Generates, validates, and processes X9.150 Payment QR Codes
        │
        ├──────────────┐
        ▼              ▼
   Keycloak         MongoDB
(OAuth2 auth &   (Datastore for payment
 authorization)   requests and metadata)
        │
        ▼
X9 Certificates / PKI
(Signing certificates, truststore, and PKI validation)
```

**Legend:**
- Request / Response Flow
- Authentication Flow
- Data Persistence
- PKI / Certificate Flow

## Architecture

The software is deployed as a containerized Java service in Kubernetes and integrates with standard infrastructure components for authentication, persistence, and certificate management.

- Java / Spring Boot application
- Kubernetes deployment using Helm
- MongoDB datastore (production)
- Keycloak for OAuth2 authentication
- X9 certificate management for signing and validation

> **Note:** MySQL is supported only for development and testing. MongoDB is the recommended production datastore.

## Before You Begin

Ensure the following components are available before installing the application.

| Requirement | Purpose |
|---|---|
| Kubernetes cluster | Hosts the application |
| Helm 3.x | Deploys the application |
| Container registry access | Pulls the application image |
| MongoDB | Stores payment request data |
| Keycloak | OAuth2 authentication |
| X9 certificates | Signing and validation |
| Kubernetes namespace | Deployment target |
| Image pull secret | Registry authentication |

## Development vs. Production

The Helm chart supports both development and production deployments.

| Development | Production |
|---|---|
| MySQL (optional) | MongoDB |
| Security may be disabled | OAuth2 authentication required |
| Demo certificates | Production certificates |
| Local or Kubernetes Secrets | External secret management recommended |

The sample configuration in this guide is intended as a starting point for development environments. Production deployments should replace all sample credentials, certificates, and secrets.

## Installation

Installing X9 QR Code consists of four steps:

1. Prepare the Kubernetes environment.
2. Configure the application.
3. Deploy the Helm chart.
4. Validate the installation.

### Step 1 – Create a Namespace

Create (or select) the Kubernetes namespace where the application will be installed.

```bash
kubectl create namespace <namespace>
```

### Step 2 – Configure the Application

The application requires several configuration settings before it can start successfully.

**Container Image**

```yaml
image:
  repository: registry.company-domain.com/fednow/x9-qrcode
```

**Authentication (Keycloak)**

Configure the OAuth2 provider used to secure the application.

```yaml
application:
  keycloak:
    host: https://keycloak.company-domain.com
    realm: Matera
```

**Public Endpoint**

Specify the public URL used in X9 payment payloads.

```yaml
application:
  x9:
    public-endpoints:
      host: https://x9.company-domain.com
```

**Certificates**

Configure the X9 signing certificate and truststore.

```yaml
application:
  x9:
    certificate:
      private-keystore:
        location: /certificate/x9-matera.jks
      truststore:
        location: /certificate/x9-truststore.jks
```

**Secrets**

Sensitive values should be stored separately from the application configuration.

Typical secrets include:

- MongoDB connection string
- Keycloak client secret
- Keystore password
- Private key password
- Truststore password

Example:

```yaml
secrets:
  spring:
    data:
      mongodb:
        uri: mongodb+srv://...
```

### Step 3 – Deploy the Application

Install or upgrade the Helm release.

```bash
helm upgrade --install x9-qrcode <repo-name>/x9-qrcode \
  --namespace <namespace> \
  --values values.yaml
```

To validate the generated Kubernetes manifests before deployment:

```bash
helm template x9-qrcode <repo-name>/x9-qrcode \
  --values values.yaml
```

### Step 4 – Validate the Deployment

After deployment, verify that the application has started successfully and is ready to accept requests.

- Pods are running
- Readiness probes succeed
- Liveness probes succeed
- MongoDB connectivity is established
- Keycloak authentication is working

Confirm that the following API groups are available:

- Payment Request
- Payment Notification
- Signature Services
- Public X9 Endpoints

Complete API details are available in the project's OpenAPI specification.

## Configuration Reference

The Helm chart provides configuration options for:

- Naming
- Images
- Service Accounts
- Deployment Strategy
- Runtime Environment
- Networking
- Autoscaling
- Resource Limits
- Health Probes
- Scheduling
- Application Configuration
- Secrets

The complete default configuration can be exported using:

```bash
helm show values <repo-name>/x9-qrcode > values-full.yaml
```

## Security Considerations

For production deployments:

- Enable OAuth2 authentication.
- Use MongoDB with TLS enabled.
- Store secrets in a secure secret management solution.
- Replace all sample certificates.
- Replace all sample passwords.
- Validate certificate trust using the X9 Financial PKI.

## Additional Resources

For complete configuration details, refer to:

- Helm Values Reference
- OpenAPI Specification
- Matera Portal Documentation
