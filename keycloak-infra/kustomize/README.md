# Keycloak Kustomize Deployment

Alternative deployment method using Kustomize instead of Helm.

## Prerequisites

- kubectl (v1.23+)
- kustomize (v4.0+) or kubectl with kustomize support
- Kubernetes cluster (kind, minikube, or cloud provider)

## Quick Start

### Deploy Development Environment

```bash
# Deploy to cluster
cd keycloak-infra/kustomize
kubectl apply -k overlays/dev/

# Wait for pods to be ready
kubectl get pods -n keycloak -w

# Port forward to access (via service)
kubectl port-forward -n keycloak svc/dev-keycloak 8080:80

# Or connect directly to a specific pod (using stable StatefulSet name)
kubectl port-forward -n keycloak dev-keycloak-0 8080:8080
```

Access at: http://localhost:8080

- Username: `user`
- Password: `admin123` (dev environment)

### Deploy Production Environment

#### Step 1: Generate Secure Passwords

Generate strong passwords for production:

```bash
# Generate secure passwords (Linux/macOS)
echo "Keycloak Admin Password: $(openssl rand -base64 32)"
echo "PostgreSQL Password: $(openssl rand -base64 32)"

# Or use this alternative
echo "Keycloak Admin Password: $(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32)"
echo "PostgreSQL Password: $(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32)"
```

**Save these passwords securely** (e.g., in a password manager or secrets management system).

#### Step 2: Update Production Secrets

Edit `overlays/prod/kustomization.yaml` and replace the placeholder passwords:

```bash
# Open the file in your editor
vim overlays/prod/kustomization.yaml
# or
nano overlays/prod/kustomization.yaml
```

Find the secret patch section and update:

```yaml
patches:
  # Update secrets with production credentials (CHANGE THESE!)
  - patch: |-
      apiVersion: v1
      kind: Secret
      metadata:
        name: keycloak-secrets
        namespace: keycloak
      stringData:
        admin-password: YOUR_GENERATED_ADMIN_PASSWORD_HERE
        postgres-password: YOUR_GENERATED_DB_PASSWORD_HERE
    target:
      kind: Secret
      name: keycloak-secrets
```

**Important**: Replace `YOUR_GENERATED_ADMIN_PASSWORD_HERE` and `YOUR_GENERATED_DB_PASSWORD_HERE` with the passwords you generated in Step 1.

#### Step 3: Review Configuration

Check the production configuration before deploying:

```bash
# Preview all resources that will be created
kubectl kustomize overlays/prod/

# Save to a file for review
kubectl kustomize overlays/prod/ > prod-deployment-preview.yaml

# Verify secrets are properly set (passwords will be base64 encoded)
kubectl kustomize overlays/prod/ | grep -A 10 "kind: Secret"
```

#### Step 4: Deploy to Production

```bash
# Deploy to cluster
kubectl apply -k overlays/prod/

# Wait for all pods to be ready (this may take 2-3 minutes)
kubectl get pods -n keycloak -w

# Expected output:
# NAME                              READY   STATUS    RESTARTS   AGE
# prod-keycloak-0                   1/1     Running   0          2m
# prod-keycloak-1                   1/1     Running   0          2m
# prod-keycloak-2                   1/1     Running   0          2m
# prod-postgresql-0                 1/1     Running   0          2m
```

#### Step 5: Verify Deployment

```bash
# Check all resources
kubectl get all -n keycloak

# Verify all 3 Keycloak replicas are running
kubectl get statefulset -n keycloak prod-keycloak

# Check pod logs
kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak --tail=50

# Test connectivity (optional, for testing only)
kubectl port-forward -n keycloak svc/prod-keycloak 8080:80

# Or connect to a specific pod
kubectl port-forward -n keycloak prod-keycloak-0 8080:8080
```

Access at: http://localhost:8080

- Username: user
- Password: (the admin password you set in Step 2)

#### Step 6: Configure Ingress (Recommended for Production)

For production access, set up an Ingress with TLS instead of port-forwarding. See the [Add Ingress](#add-ingress) section below.

**CLean Up**:

```bash
# Delete dev environment
kubectl delete -k overlays/dev/

# Delete prod environment
kubectl delete -k overlays/prod/
```

## Structure

```plaintext
kustomize/
├── base/                          # Base manifests
│   ├── namespace.yaml
│   ├── postgres-configmap.yaml
│   ├── secrets.yaml
│   ├── postgres-pvc.yaml
│   ├── postgres-statefulset.yaml
│   ├── postgres-service.yaml
│   ├── keycloak-statefulset.yaml
│   ├── keycloak-service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/                       # Development overlay
    │   └── kustomization.yaml
    └── prod/                      # Production overlay
        └── kustomization.yaml
```

## Configuration

### Images Used

- **Keycloak**: `docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0`
- **PostgreSQL**: `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r0`

### Resource Allocation

#### Development

- **Keycloak**: 500m CPU, 512Mi RAM (requests) | 750m CPU, 768Mi RAM (limits)
- **PostgreSQL**: 100m CPU, 128Mi RAM (requests) | 150m CPU, 192Mi RAM (limits)
- **Replicas**: 1

#### Production

- **Keycloak**: 1000m CPU, 1Gi RAM (requests) | 2000m CPU, 2Gi RAM (limits)
- **PostgreSQL**: 100m CPU, 128Mi RAM (requests) | 150m CPU, 192Mi RAM (limits)
- **Replicas**: 3

## Customization

### Update Secrets

Secrets are configured using patches in the overlay files. To update:
**For Development:**
Edit `overlays/dev/kustomization.yaml` - secrets are already set to simple values for testing.
**For Production:**
Edit `overlays/prod/kustomization.yaml` and update the secret patch:

```yaml
patches:
 - patch: |-
 apiVersion: v1
 kind: Secret
 metadata:
 name: keycloak-secrets
 namespace: keycloak
 stringData:
 admin-password: YOUR_SECURE_PASSWORD
 postgres-password: YOUR_SECURE_DB_PASSWORD
 target:
 kind: Secret
 name: keycloak-secrets
```

**Alternative: Use External Secrets**
For enhanced security, consider using external secret management:

- [External Secrets Operator](https://external-secrets.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- Cloud provider secret managers (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager)

### Change Replicas

Edit `overlays/prod/kustomization.yaml`:

```yaml
replicas:
  - name: keycloak
    count: 5  # Increase replicas
```

### Add Ingress

Create `overlays/prod/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
  namespace: keycloak
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - keycloak.yourdomain.com
    secretName: keycloak-tls
  rules:
  - host: keycloak.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prod-keycloak
            port:
              number: 80
```

Add to `overlays/prod/kustomization.yaml`:

```yaml
resources:
  - ingress.yaml
```

## Commands

### Preview Changes

```bash
# See what will be applied
kubectl kustomize overlays/dev/
kubectl kustomize overlays/prod/
```

### Deploy

```bash
# Development
kubectl apply -k overlays/dev/

# Production
kubectl apply -k overlays/prod/
```

### Update

```bash
# Update deployment
kubectl apply -k overlays/dev/

# Or use kubectl diff to see changes first
kubectl diff -k overlays/dev/
```

### Delete

```bash
# Delete dev environment
kubectl delete -k overlays/dev/

# Delete prod environment
kubectl delete -k overlays/prod/
```

### Troubleshooting

```bash
# Check pods
kubectl get pods -n keycloak

# View logs
kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak
kubectl logs -n keycloak -l app.kubernetes.io/name=postgresql

# Describe resources
kubectl describe statefulset -n keycloak dev-keycloak
kubectl describe statefulset -n keycloak dev-postgresql

# Check secrets
kubectl get secrets -n keycloak
```

## Common Issues

### Pods Stuck in ImagePullBackOff

First deployment takes 15-20 minutes to pull images (~800MB each). Be patient!

```bash
kubectl get events -n keycloak --sort-by='.lastTimestamp' | tail -20
```

### Password Authentication Failed

Clean up and redeploy:

```bash
kubectl delete -k overlays/dev/
kubectl delete pvc -n keycloak --all
kubectl apply -k overlays/dev/
```

### Port Already in Use

Use a different port:

```bash
kubectl port-forward -n keycloak svc/dev-keycloak 9090:80
```

## Next Steps

1. **Test deployment**: Verify Keycloak is accessible
2. **Create a realm**: Admin Console → Create Realm
3. **Add clients**: Configure OAuth2/OIDC applications
4. **Customize**: Themes, login flows, authentication
5. **Production**: Review production checklist above

For more details, see:

- [Main README](../README.md) - Project overview
- [TECHNICAL_GUIDE](../TECHNICAL_GUIDE.md) - Detailed troubleshooting
