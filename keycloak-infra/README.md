# Keycloak Deployment with Bitnami Helm Charts

Complete production-ready Keycloak deployment on Kubernetes using Bitnami Helm charts with PostgreSQL database.

## 📋 Table of Contents

- [Keycloak Deployment with Bitnami Helm Charts](#keycloak-deployment-with-bitnami-helm-charts)
  - [📋 Table of Contents](#-table-of-contents)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Project Structure](#project-structure)
  - [Deployment Options](#deployment-options)
    - [Option 1: Helm Deployment (Recommended)](#option-1-helm-deployment-recommended)
    - [Option 2: Kustomize Deployment](#option-2-kustomize-deployment)
  - [Quick Start (Helm)](#quick-start-helm)
    - [Customizing Helm Configuration](#customizing-helm-configuration)
  - [Verify It Works](#verify-it-works)
  - [Next Steps](#next-steps)
  - [Comparison Between Helm \& Kustomize](#comparison-between-helm--kustomize)
  - [Documentation](#documentation)

## Overview

This project provides a complete Keycloak Identity and Access Management (IAM) solution deployed on Kubernetes using:

- **Keycloak 26.3.3** - Modern authentication and authorization server
- **PostgreSQL 17.6.0** - Production-ready database backend
- **Bitnami Helm Charts** - Battle-tested, enterprise-ready Helm charts
- **Kind Cluster** - Local Kubernetes cluster for development and testing

⏱️ **Deployment Time**:

- First deployment: 15-20 minutes (includes image downloads)
- Subsequent deployments: 2-3 minutes (images cached)

## Prerequisites

- **Docker** - For running kind cluster
- **kubectl** (v1.23+)
- **Helm** (v3.8.0+)
- **kind** - Kubernetes in Docker (for local development)
- **4GB+ RAM** available for the cluster

## Project Structure

```plaintext
keycloak-infra/
├── README.md                          # Project overview and quick start
├── TECHNICAL_GUIDE.md                 # Detailed troubleshooting and configuration
└── keycloak-infra/
    ├── keycloak/                      # Helm deployment
    │   ├── Chart.yaml                 # Helm chart metadata
    │   ├── Chart.lock                 # Dependency lock file
    │   ├── values.yaml                # Default configuration
    │   ├── values-dev.yaml            # Development overrides
    │   ├── charts/                    # Dependency charts (PostgreSQL, common)
    │   └── templates/                 # Kubernetes manifests
    └── kustomize/                     # Kustomize deployment (alternative)
        ├── base/                      # Base manifests
        ├── overlays/dev/              # Development overlay
        ├── overlays/prod/             # Production overlay
        └── README.md                  # Kustomize documentation
```

## Deployment Options

Choose between **Helm** (recommended for production) or **Kustomize** (simpler, GitOps-friendly):

### Option 1: Helm Deployment (Recommended)

Best for: Production deployments, complex configurations, dependency management

### Option 2: Kustomize Deployment

Best for: Simple deployments, GitOps workflows, learning Kubernetes

See [kustomize/README.md](kustomize/README.md) for Kustomize instructions.

## Quick Start (Helm)

Get Keycloak running in 3 commands:

```bash
# 1. Create cluster
kind create cluster --name keycloak-demo

# 2. Deploy Keycloak
cd keycloak-infra
helm install my-keycloak ./keycloak --namespace keycloak --create-namespace

# 3. Wait and watch (pods will take 15-20 minutes on first run)
kubectl get pods -n keycloak -w
```

**Access Keycloak**:

```bash
# Get admin credentials
echo "Username: user"
kubectl get secret my-keycloak -n keycloak -o jsonpath="{.data.admin-password}" | base64 -d
echo

# Port forward to access
kubectl port-forward -n keycloak svc/my-keycloak 8080:80
```

Open browser: **http://localhost:8080**

### Customizing Helm Configuration

You can customize the default Helm configuration by using the `values-dev.yaml` file to override default values and set your own configuration.

**Usage Example**:

```bash
# Deploy with custom configuration
helm install my-keycloak ./keycloak \
  -f values-dev.yaml \
  --namespace keycloak \
  --create-namespace
```

**Common Customizations**:

```yaml
# Example values-dev.yaml customizations
auth:
  adminUser: myadmin
  adminPassword: "mySecurePassword123"

resources:
  limits:
    memory: 2Gi
    cpu: 1500m
  requests:
    memory: 1Gi
    cpu: 750m

service:
  type: LoadBalancer  # Instead of default ClusterIP
```

The `values-dev.yaml` file allows you to override any default configuration from `values.yaml` while keeping your customizations separate and maintainable.

**Clean Up**:

```bash
# Delete keycloak release from cluster
helm uninstall my-keycloak -n keycloak

# Delete cluster
kind delete cluster --name keycloak-demo
```

## Verify It Works

Once pods are running (1/1 Ready status):

1. Access the Keycloak admin console at **http://localhost:8080**
2. Click **"Administration Console"**
3. Login with credentials from above
4. You should see the Keycloak admin dashboard
5. **Test**: Navigate to **Clients** → You'll see default clients
6. **Test**: Navigate to **Users** → Click "Add user" to test user creation

## Next Steps

1. **Create a Realm**: Admin Console → Create Realm → "my-app"
2. **Add a Client**: Configure OAuth2/OIDC for your application
3. **Add Users**: Create test users in your realm
4. **Customize**: Change themes, configure login flows
5. **Development**: Use `values-dev.yaml` for fixed credentials (`admin/admin123`)

## Comparison Between Helm & Kustomize

| Feature | Helm | Kustomize |
|---------|------|-----------|
| Templating | Yes (Go templates) | No (patches & overlays) |
| Package Management | Yes | No |
| Dependencies | Yes | No |
| Learning Curve | Moderate | Low |
| Complexity | Higher | Lower |
| Best For | Complex apps with many configurations | Simple deployments, GitOps |

## Documentation

- **[TECHNICAL_GUIDE.md](TECHNICAL_GUIDE.md)** - Comprehensive troubleshooting, configuration details, and production deployment guide
- **[keycloak-infra/kustomize/README.md](kustomize/README.md)** - Kustomize deployment guide
- **[Bitnami Keycloak Chart](https://github.com/bitnami/charts/tree/main/bitnami/keycloak)** - Official Bitnami chart documentation

---

**Quick Troubleshooting**:

- Pods stuck? Images are large (~800MB), first pull takes 15-20 minutes
- CrashLoopBackOff? Clean up PVCs and secrets, then reinstall

For detailed troubleshooting, see [TECHNICAL_GUIDE.md](TECHNICAL_GUIDE.md).
