# Technical Guide

Comprehensive technical documentation for the Keycloak deployment.

## Table of Contents

- [Technical Guide](#technical-guide)
  - [Table of Contents](#table-of-contents)
  - [Image Configuration](#image-configuration)
    - [Why bitnamilegacy Images?](#why-bitnamilegacy-images)
    - [Why NOT Official Keycloak Images?](#why-not-official-keycloak-images)
    - [Why NOT Official PostgreSQL Images?](#why-not-official-postgresql-images)
    - [Security Flag Required](#security-flag-required)
    - [Current Image Versions](#current-image-versions)
  - [Secret Management](#secret-management)
    - [Auto-Generated Secrets](#auto-generated-secrets)
    - [Secret Mismatch Problem](#secret-mismatch-problem)
  - [Configuration Details](#configuration-details)
    - [Default Credentials](#default-credentials)
    - [Database Configuration](#database-configuration)
    - [Resource Allocation](#resource-allocation)
      - [Keycloak Pod](#keycloak-pod)
      - [PostgreSQL Pod](#postgresql-pod)
    - [Persistent Storage](#persistent-storage)
  - [Startup Sequence](#startup-sequence)
    - [PostgreSQL Startup](#postgresql-startup)
    - [Keycloak Startup](#keycloak-startup)
  - [Network Architecture](#network-architecture)
  - [Chart Dependencies](#chart-dependencies)
  - [Architecture Decisions](#architecture-decisions)
    - [StatefulSets vs Deployments (Kustomize)](#statefulsets-vs-deployments-kustomize)
      - [Why StatefulSets?](#why-statefulsets)
      - [Implementation Notes](#implementation-notes)
  - [Troubleshooting](#troubleshooting)
    - [Issue: Pods Stuck in ImagePullBackOff](#issue-pods-stuck-in-imagepullbackoff)
    - [Issue: Keycloak CrashLoopBackOff with Password Authentication Failed](#issue-keycloak-crashloopbackoff-with-password-authentication-failed)
    - [Issue: Keycloak Takes Long to Start](#issue-keycloak-takes-long-to-start)
    - [Issue: "Original containers have been substituted" Warning](#issue-original-containers-have-been-substituted-warning)
    - [Issue: Port Already in Use](#issue-port-already-in-use)
    - [Check Deployment Status](#check-deployment-status)
  - [Development vs Production](#development-vs-production)
    - [Development (values-dev.yaml)](#development-values-devyaml)
    - [Production Considerations](#production-considerations)
      - [1. Use External Database](#1-use-external-database)
      - [2. Enable TLS/SSL](#2-enable-tlsssl)
      - [3. Configure Ingress](#3-configure-ingress)
      - [4. Set Appropriate Resource Limits](#4-set-appropriate-resource-limits)
      - [5. Enable Monitoring](#5-enable-monitoring)
      - [6. High Availability](#6-high-availability)
      - [7. Secure Credentials](#7-secure-credentials)
      - [8. Configure Backup Strategy](#8-configure-backup-strategy)
      - [9. Logging and Auditing](#9-logging-and-auditing)
      - [10. Network Policies](#10-network-policies)
  - [Maintenance Tasks](#maintenance-tasks)
    - [Backup Database](#backup-database)
    - [Restore Database](#restore-database)
    - [Update Chart Dependencies](#update-chart-dependencies)
    - [View Helm Release History](#view-helm-release-history)
    - [Rollback to Previous Version](#rollback-to-previous-version)
    - [Export Current Configuration](#export-current-configuration)
  - [Summary](#summary)

## Image Configuration

### Why bitnamilegacy Images?

This deployment uses **`bitnamilegacy`** images instead of the standard `bitnami` images:

- **Keycloak**: `docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0`
- **PostgreSQL**: `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r0`
- **Config CLI**: `docker.io/bitnamilegacy/keycloak-config-cli:6.4.0-debian-12-r11`

**Reasons**:

1. **Bitnami Image Migration**: As of August 28, 2025, Bitnami moved older versioned images to the `bitnamilegacy` repository
2. **Chart Compatibility**: The Bitnami Helm chart is specifically designed for these image paths and their environment variables. Official Keycloak images use different configuration patterns that are incompatible with the chart's templates
3. **Production Stability**: These legacy images are stable, tested, and fully compatible with the chart's configuration. They receive security updates and are maintained by VMware/Broadcom
4. **Integration**: The chart expects specific container structure, environment variables, and initialization scripts that only Bitnami-packaged images provide

**Note**: Despite the "legacy" name, these images are actively maintained and receive security patches. They're legacy only in the sense that they're the older naming convention.

### Why NOT Official Keycloak Images?

Attempted using `quay.io/keycloak/keycloak` but encountered multiple compatibility issues:

1. **Path Incompatibility**: Official images use `/opt/keycloak` vs `/opt/bitnami/keycloak`
2. **Init Container Failures**: Chart's prepare-write-dirs init container fails with different directory structure
3. **Environment Variables**: Different variable names and formats (e.g., `KC_DB_URL` vs Bitnami's conventions)
4. **Volume Mounts**: Incompatible mount paths break the chart's volume configuration
5. **Database Config**: Different connection parameter formats and configuration methods

**Conclusion**: The Bitnami Helm chart is tightly coupled to Bitnami image structure. Using official images would require rewriting most of the chart templates.

### Why NOT Official PostgreSQL Images?

Similar issues with `postgres:17-alpine` and other official PostgreSQL images:

1. **Data Directory**: Uses `/var/lib/postgresql` vs `/bitnami/postgresql`
2. **Initialization Scripts**: Different script locations (`/docker-entrypoint-initdb.d` vs Bitnami's structure)
3. **Health Checks**: Incompatible probe commands (different `pg_isready` paths)
4. **User Management**: Different authentication configuration and user creation methods

**Recommendation**: Stick with `bitnamilegacy` images for both Keycloak and PostgreSQL to ensure full compatibility with the Bitnami Helm chart.

### Security Flag Required

Because we're using `bitnamilegacy` images, you must set this flag in your values file:

```yaml
global:
  security:
    allowInsecureImages: true  # Required for bitnamilegacy images
```

**What this does**:
- Tells the Bitnami chart to skip image verification
- `bitnamilegacy` images are not recognized as official Bitnami images by the chart's security checks
- This is **safe** when using documented `bitnamilegacy` images from Docker Hub

**Security implications**:
- The images themselves are secure and maintained
- The flag only bypasses the chart's repository naming check
- Always verify image sources: `docker.io/bitnamilegacy/*`

### Current Image Versions

| Component | Image | Version |
|-----------|-------|----------|
| Keycloak | `bitnamilegacy/keycloak` | 26.3.3-debian-12-r0 |
| PostgreSQL | `bitnamilegacy/postgresql` | 17.6.0-debian-12-r0 |
| Config CLI | `bitnamilegacy/keycloak-config-cli` | 6.4.0-debian-12-r11 |

## Secret Management

### Auto-Generated Secrets

The Helm chart automatically creates two secrets during installation:

- **`my-keycloak`**: Contains Keycloak admin password
- **`my-keycloak-postgresql`**: Contains PostgreSQL database passwords

**Critical**: Both secrets must be created in the same deployment to ensure password synchronization between Keycloak and PostgreSQL.

**Retrieve admin password**:
```bash
kubectl get secret my-keycloak -n keycloak -o jsonpath="{.data.admin-password}" | base64 -d
echo
```

### Secret Mismatch Problem

A common issue occurs when secrets and PVCs get out of sync:

**Scenario**:
1. Deploy → Creates secret A with password X
2. PostgreSQL initializes database with password X
3. Uninstall (but PVC persists with old data)
4. Redeploy → Creates secret B with password Y
5. PostgreSQL uses OLD database (password X) but Keycloak has NEW password Y
6. **Result**: `FATAL: password authentication failed for user "bn_keycloak"`

**Solution**: Always delete PVCs and secrets together:
```bash
# Complete cleanup
helm uninstall my-keycloak -n keycloak
kubectl delete pvc --all -n keycloak
kubectl delete secret --all -n keycloak

# Wait for cleanup to complete
kubectl get all -n keycloak  # Should show no resources

# Fresh install
helm install my-keycloak ./keycloak --namespace keycloak
```

## Configuration Details

### Default Credentials

- **Admin Username**: `user`
- **Admin Password**: Auto-generated (retrieve using command below)

```bash
kubectl get secret my-keycloak -n keycloak -o jsonpath="{.data.admin-password}" | base64 -d
echo
```

**For Development** (values-dev.yaml):
- **Username**: `admin`
- **Password**: `admin123`

### Database Configuration

- **Database**: PostgreSQL 17.6.0
- **Database Name**: `bitnami_keycloak`
- **Database User**: `bn_keycloak`
- **Database Password**: Auto-generated and auto-configured
- **Connection**: Internal service (`my-keycloak-postgresql:5432`)
- **Persistence**: 8Gi PVC for data storage

### Resource Allocation

#### Keycloak Pod
- **CPU Requests**: 500m
- **CPU Limits**: 750m
- **Memory Requests**: 512Mi
- **Memory Limits**: 768Mi

#### PostgreSQL Pod
- **CPU Requests**: 100m
- **CPU Limits**: 150m
- **Memory Requests**: 128Mi
- **Memory Limits**: 192Mi

### Persistent Storage

- **PostgreSQL Data**: 8Gi PVC (auto-provisioned by kind)
- **Storage Class**: `standard` (kind's default)
- **Access Mode**: ReadWriteOnce

## Startup Sequence

Understanding the startup sequence helps set realistic expectations for deployment times.

### PostgreSQL Startup

**Timeline for `my-keycloak-postgresql-0`**:

1. **Image pull**: 10-15 minutes (first time only, ~800MB image)
2. **Init container**: ~5 seconds
3. **PostgreSQL start**: ~10 seconds
4. **Database initialization**: ~20 seconds
5. **Ready state**: ~1 minute after image is pulled

**Total**: ~1 minute (after image cached) or ~15 minutes (first deployment)

### Keycloak Startup

**Timeline for `my-keycloak-0`**:

1. **Image pull**: 10-15 minutes (first time only, ~800MB image)
2. **Init container** (prepare-write-dirs): ~5 seconds
3. **Wait for PostgreSQL**: Variable (depends on PostgreSQL readiness)
4. **Database schema creation**: ~60 seconds
5. **JGroups clustering setup**: ~30 seconds
6. **Quarkus application startup**: ~30 seconds
7. **Master realm initialization**: ~30 seconds
8. **Ready state**: ~3 minutes after PostgreSQL is ready

**Total First Deploy**: ~18 minutes (includes image pulls)  
**Subsequent Deploys**: ~3 minutes (images cached)

**What to expect**:
```
0:00  - Deployment created
0:30  - Images pulling (can take 10-15 min)
12:00 - PostgreSQL container starting
13:00 - PostgreSQL ready (1/1 Running)
13:30 - Keycloak container starting
16:00 - Keycloak ready (1/1 Running)
```

## Network Architecture

The following diagram shows the network flow and service architecture:

```
┌─────────────────────────────────────────────┐
│  External Access                            │
│  (kubectl port-forward or Ingress)          │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
         ┌─────────────────────┐
         │  my-keycloak        │ Service (ClusterIP)
         │  Port: 80           │
         └────────┬────────────┘
                  │
                  ▼
         ┌─────────────────────┐
         │  my-keycloak-0      │ Pod (StatefulSet)
         │  Port: 8080         │
         └────────┬────────────┘
                  │
                  │ JDBC Connection
                  ▼
         ┌──────────────────────────┐
         │  my-keycloak-postgresql  │ Service (ClusterIP)
         │  Port: 5432              │
         └────────┬─────────────────┘
                  │
                  ▼
         ┌───────────────────────────┐
         │  my-keycloak-postgresql-0 │ Pod (StatefulSet)
         │  Port: 5432               │
         │  PVC: data-my-keycloak-   │
         │       postgresql-0        │
         └───────────────────────────┘
```

**Key Points**:
- Keycloak connects to PostgreSQL via internal ClusterIP service
- External access requires port-forward (dev) or Ingress (prod)
- StatefulSets provide stable pod names and network identities
- PostgreSQL data persists in a PVC

## Chart Dependencies

The Keycloak Helm chart depends on two sub-charts from Bitnami:

**From `Chart.yaml`**:
```yaml
dependencies:
  - name: postgresql
    version: 16.x.x
    repository: oci://registry-1.docker.io/bitnamicharts
    condition: postgresql.enabled

  - name: common
    version: 2.x.x
    repository: oci://registry-1.docker.io/bitnamicharts
```

**What each provides**:
- **postgresql**: Bundled PostgreSQL database (can be disabled for external DB)
- **common**: Bitnami common templates and helpers

**Important**: `Chart.lock` pins exact versions for reproducible builds.

**Update dependencies**:
```bash
cd keycloak-infra/keycloak
helm dependency update
```

This downloads the latest chart versions matching the version constraints in `Chart.yaml`.

## Architecture Decisions

### StatefulSets vs Deployments (Kustomize)

The Kustomize deployment uses **StatefulSets** for both Keycloak and PostgreSQL instead of standard Deployments. This is an intentional architectural choice that provides significant operational benefits.

#### Why StatefulSets?

**Stable Pod Names**
- Pods are named predictably: `keycloak-0`, `keycloak-1`, `keycloak-2`
- Names never change across restarts or updates
- Easy to identify and target specific instances
- Better for debugging and monitoring

**Stable Network Identities**
- Each pod gets a predictable DNS name
- Format: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`
- Example: `keycloak-0.keycloak.keycloak.svc.cluster.local`
- Enables direct pod-to-pod communication if needed

**Ordered Deployment**
- Pods are created, updated, and deleted in order (0 → 1 → 2)
- Pod `N` only starts after pod `N-1` is Running and Ready
- Prevents race conditions during initialization
- Controlled rollout with predictable behavior

**Better for Distributed Caching**
- Keycloak uses Infinispan for distributed session caching
- JDBC_PING discovery works better with stable pod identities
- Cache rebalancing is more efficient
- Reduces cache misses during pod restarts

**Persistent Storage Support**
- Each pod can have its own PVC (if needed)
- PVCs are stable and bound to specific pod ordinals
- Survives pod restarts and rescheduling
- Better for future stateful requirements

#### Implementation Notes

The Kustomize manifests use StatefulSets in both environments:

**Development** (`overlays/dev`):
- 1 replica: `dev-keycloak-0`
- Simple setup for local testing

**Production** (`overlays/prod`):
- 3 replicas: `prod-keycloak-0`, `prod-keycloak-1`, `prod-keycloak-2`
- High availability with cache clustering
- Sequential rollout ensures stability

See `keycloak-infra/kustomize/base/keycloak-statefulset.yaml` for the implementation.

## Troubleshooting

### Issue: Pods Stuck in ImagePullBackOff

**Symptom**: Pods show `ErrImagePull` or `ImagePullBackOff`

**Cause**: Large images (~800MB each) taking time to download

**Solution**: Be patient! First-time image pull takes 10-15 minutes. Monitor progress:

```bash
kubectl get events -n keycloak --sort-by='.lastTimestamp' | tail -20
```

Look for: `Pulling image "docker.io/bitnamilegacy/keycloak...`

**Timing expectations**:
- **First Deployment**: 15-20 minutes (image downloads + startup)
- **Subsequent Deploys**: 2-3 minutes (images cached locally)

### Issue: Keycloak CrashLoopBackOff with Password Authentication Failed

**Symptom**: 
```
FATAL: password authentication failed for user "bn_keycloak"
```

**Cause**: Secrets and PVCs from previous deployments have mismatched passwords. This happens when:
- PostgreSQL was initialized with one set of credentials
- The data persists in PVC
- A new deployment generates different credentials
- The new credentials don't match the database

**Solution**: Complete cleanup and fresh install:

```bash
# 1. Uninstall release
helm uninstall my-keycloak -n keycloak

# 2. Delete persistent data (important!)
kubectl delete pvc --all -n keycloak

# 3. Delete secrets
kubectl delete secret --all -n keycloak

# 4. Wait for cleanup
kubectl get all -n keycloak  # Should show no resources

# 5. Reinstall
helm install my-keycloak ./keycloak --namespace keycloak
```

### Issue: Keycloak Takes Long to Start

**Symptom**: Pod running but not ready, readiness probe failures

**Cause**: Normal behavior - Keycloak initialization takes 2-3 minutes

**What's happening**:
1. Database schema creation (~1 minute)
2. JGroups clustering setup (~30 seconds)
3. Master realm initialization (~30 seconds)
4. Quarkus application startup (~30 seconds)

**Check logs**:
```bash
kubectl logs my-keycloak-0 -n keycloak -f
```

Look for: `Keycloak 26.3.3 on JVM started in XXXs`

**Timeline**:
```
0:00 - Pod created
0:30 - Container started
1:00 - Database connection established
2:00 - Keycloak initializing
3:00 - Keycloak ready (1/1 Running)
```

### Issue: "Original containers have been substituted" Warning

**Symptom**: Helm install shows security warning about substituted images

```
Warning: original containers have been substituted
```

**Cause**: Using `bitnamilegacy` images triggers Bitnami chart's security check

**Solution**: This is expected! The warning is safe to ignore when:
- `global.security.allowInsecureImages: true` is set
- You're using the documented `bitnamilegacy` images from Docker Hub
- The images match the versions specified in this guide

### Issue: Port Already in Use

**Symptom**: Port forwarding fails with "bind: address already in use"

**Solution**: Use a different local port

```bash
# Instead of port 8080, use 9090
kubectl port-forward -n keycloak svc/my-keycloak 9090:80
# Access: http://localhost:9090

# Or use any available port
kubectl port-forward -n keycloak svc/my-keycloak 8888:80
```

### Check Deployment Status

**Quick status check**:
```bash
kubectl get pods -n keycloak
```

**Detailed pod information**:
```bash
kubectl describe pod my-keycloak-0 -n keycloak
kubectl describe pod my-keycloak-postgresql-0 -n keycloak
```

**View logs**:
```bash
# Keycloak logs
kubectl logs my-keycloak-0 -n keycloak
kubectl logs my-keycloak-0 -n keycloak -f  # Follow logs
kubectl logs my-keycloak-0 -n keycloak --tail=50  # Last 50 lines

# PostgreSQL logs
kubectl logs my-keycloak-postgresql-0 -n keycloak
```

**Check resources**:
```bash
# Check secrets
kubectl get secrets -n keycloak

# Check persistent volumes
kubectl get pvc -n keycloak

# Check services
kubectl get svc -n keycloak

# Check all resources
kubectl get all -n keycloak
```

**Check events**:
```bash
kubectl get events -n keycloak --sort-by='.lastTimestamp'
```

## Development vs Production

### Development (values-dev.yaml)

Use `values-dev.yaml` for local development:

```bash
helm install my-keycloak ./keycloak \
  -f values-dev.yaml \
  --namespace keycloak \
  --create-namespace
```

**Features**:
- Fixed admin credentials (`admin/admin123`)
- Reduced resource limits for local development
- Bundled PostgreSQL instance
- No TLS/SSL
- ClusterIP service
- Faster startup times

**Resource configuration**:
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 750m
    memory: 768Mi
```

### Production Considerations

**Before deploying to production, you must configure**:

#### 1. Use External Database

Do not use the bundled PostgreSQL in production:

```yaml
postgresql:
  enabled: false

externalDatabase:
  host: your-postgres-server.example.com
  port: 5432
  user: keycloak
  database: keycloak
  existingSecret: keycloak-db-secret
  existingSecretPasswordKey: password
```

Create the database secret:
```bash
kubectl create secret generic keycloak-db-secret \
  --from-literal=password='your-secure-password' \
  -n keycloak
```

#### 2. Enable TLS/SSL

Always use TLS in production:

```yaml
production: true

tls:
  enabled: true
  autoGenerated:
    enabled: true  # Or provide your own certificates
```

Or use your own certificates:
```yaml
tls:
  enabled: true
  existingSecret: keycloak-tls-secret
```

#### 3. Configure Ingress

Expose Keycloak through an Ingress controller:

```yaml
ingress:
  enabled: true
  hostname: keycloak.yourdomain.com
  ingressClassName: nginx  # or your ingress controller
  tls: true
  certManager: true  # If using cert-manager
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

#### 4. Set Appropriate Resource Limits

Production workloads need more resources:

```yaml
resources:
  limits:
    memory: 2Gi
    cpu: 2000m
  requests:
    memory: 1Gi
    cpu: 1000m
```

#### 5. Enable Monitoring

Set up Prometheus metrics:

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

#### 6. High Availability

Run multiple replicas:

```yaml
replicaCount: 3

podDisruptionBudget:
  enabled: true
  minAvailable: 2

podAntiAffinity:
  type: hard
```

#### 7. Secure Credentials

Never use default or simple passwords:

```yaml
auth:
  adminUser: admin
  existingSecret: keycloak-admin-secret
```

Create the admin secret:
```bash
kubectl create secret generic keycloak-admin-secret \
  --from-literal=admin-password='your-very-secure-random-password' \
  -n keycloak
```

#### 8. Configure Backup Strategy

- Set up regular database backups
- Test restore procedures
- Document recovery processes
- Consider using PostgreSQL replication

#### 9. Logging and Auditing

Enable detailed logging:

```yaml
logging:
  level: INFO  # Or WARN for production
```

Configure audit logging:
```yaml
extraEnvVars:
  - name: KC_LOG_LEVEL
    value: INFO
  - name: KC_LOG_CONSOLE_COLOR
    value: "false"
```

#### 10. Network Policies

Restrict network access:

```yaml
networkPolicy:
  enabled: true
  allowExternal: false
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: your-application
```

---

## Maintenance Tasks

### Backup Database

Regular backups are critical for production deployments:

```bash
# Dump the database to a local file
kubectl exec my-keycloak-postgresql-0 -n keycloak -- \
  pg_dump -U bn_keycloak bitnami_keycloak > backup-$(date +%Y%m%d).sql

# Verify backup file
ls -lh backup-*.sql
```

**Best practices**:
- Schedule automated backups (cron, Kubernetes CronJob)
- Store backups in secure off-cluster storage (S3, GCS, Azure Blob)
- Test restore procedures regularly
- Keep multiple backup versions (daily, weekly, monthly)

### Restore Database

Restore from a backup file:

```bash
# Restore from backup file
kubectl exec -i my-keycloak-postgresql-0 -n keycloak -- \
  psql -U bn_keycloak bitnami_keycloak < backup-20251110.sql
```

**Warning**: This overwrites all current data. Test in a non-production environment first.

### Update Chart Dependencies

Update PostgreSQL and common chart dependencies:

```bash
# Update dependencies to latest matching versions
cd keycloak-infra/keycloak
helm dependency update

# Review changes
cat Chart.lock

# Upgrade the release
helm upgrade my-keycloak . -n keycloak
```

### View Helm Release History

Track all deployments and changes:

```bash
# View release history
helm history my-keycloak -n keycloak

# Example output:
# REVISION  UPDATED                   STATUS      CHART           DESCRIPTION
# 1         Mon Oct 21 10:00:00 2024  superseded  keycloak-25.2.0 Install complete
# 2         Tue Oct 22 14:30:00 2024  deployed    keycloak-25.2.0 Upgrade complete
```

### Rollback to Previous Version

Rollback if an upgrade causes issues:

```bash
# Rollback to previous revision
helm rollback my-keycloak -n keycloak

# Or rollback to specific revision
helm rollback my-keycloak 1 -n keycloak

# Verify rollback
helm history my-keycloak -n keycloak
```

### Export Current Configuration

Save your current values for reference or migration:

```bash
# Get all current values
helm get values my-keycloak -n keycloak > current-values.yaml

# Get all values including defaults
helm get values my-keycloak -n keycloak --all > all-values.yaml
```

---

## Summary

This guide covers:
- ✅ Why we use bitnamilegacy images and why official images don't work
- ✅ Secret management and common password mismatch issues
- ✅ Detailed startup sequence and timing expectations
- ✅ Network architecture and service flow
- ✅ Chart dependencies and how to update them
- ✅ Comprehensive troubleshooting for common issues
- ✅ Configuration options and resource settings
- ✅ Development vs production deployment strategies
- ✅ Production hardening checklist
- ✅ Maintenance tasks (backup, restore, rollback)

For basic usage and getting started, see [README.md](../keycloak-infra/README.md).