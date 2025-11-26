# Data Analytics Platform - Infrastructure Documentation

**Last Updated**: 2025-11-26
**Repository**: https://github.com/NoahInCloud/data-analytics-k8s
**Cluster**: data-analytics-cluster (AWS EKS, eu-central-1)

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Infrastructure Components](#infrastructure-components)
- [Deployed Services](#deployed-services)
- [Access Information](#access-information)
- [GitOps Workflow](#gitops-workflow)
- [Troubleshooting](#troubleshooting)
- [Maintenance](#maintenance)

---

## 🏗️ Architecture Overview

### Cluster Information

| Component | Details |
|-----------|---------|
| **Cloud Provider** | AWS (eu-central-1) |
| **Cluster Name** | data-analytics-cluster |
| **Cluster Type** | EKS Standard (not Auto Mode) |
| **Kubernetes Version** | v1.34.2-eks-ecaa3a6 |
| **Node Group** | managed-nodes |
| **Instance Type** | t3.small |
| **Node Count** | 3 (min: 2, max: 3) |
| **Volume Size** | 20GB per node |
| **AWS Account** | 050752617334 |

### Networking

- **VPC ID**: vpc-0468bafe044fc453d
- **Subnets**:
  - eu-central-1a: subnet-056ba4cb1480cc6a2
  - eu-central-1b: subnet-098a44da636571e62
  - eu-central-1c: subnet-0106ab6ae252b246c

### Storage

- **EBS CSI Driver**: v1.53.0-eksbuild.1 (ACTIVE)
- **Default Storage Class**: gp2
- **OIDC Provider**: oidc.eks.eu-central-1.amazonaws.com/id/896D104746BFF6BC7A453909CC6BC7DB

### IAM Roles

**DataAnalyticsEKSNodeRole** (Worker Nodes):
- AmazonEKSWorkerNodePolicy
- AmazonEKS_CNI_Policy
- AmazonEC2ContainerRegistryReadOnly
- AmazonSSMManagedInstanceCore

**DataAnalyticsEBSCSIDriverRole** (Storage):
- AmazonEBSCSIDriverPolicy
- Uses IRSA (IAM Roles for Service Accounts)

---

## 🔧 Prerequisites

### Required Tools

```bash
# AWS CLI
brew install awscli
aws configure

# kubectl
brew install kubectl

# eksctl (optional, but recommended)
brew install eksctl

# GitHub CLI
brew install gh
gh auth login
```

### Cluster Access

```bash
# Configure kubectl to access the cluster
aws eks update-kubeconfig \
  --name data-analytics-cluster \
  --region eu-central-1

# Verify access
kubectl get nodes
kubectl get namespaces
```

---

## 📦 Infrastructure Components

### 1. ArgoCD (GitOps Controller)

ArgoCD is the central GitOps controller managing all application deployments.

**Namespace**: `argocd`

**Components**:
- Application Controller (StatefulSet)
- Repo Server
- API Server
- Dex (SSO/OAuth)
- Redis
- Notifications Controller
- ApplicationSet Controller

**Web UI Access**:
```bash
# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Open browser to: https://localhost:8080
# Username: admin
# Password: 84DiV6o62IVgZ9KC
```

**CLI Access**:
```bash
# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password 84DiV6o62IVgZ9KC \
  --insecure

# List applications
argocd app list

# Sync application
argocd app sync <app-name>
```

**Managed Applications**:
- postgres-operator
- postgres-database
- pgadmin
- grafana-internal
- grafana-public

### 2. PostgreSQL Operator (PGO)

CrunchyData PostgreSQL Operator v5.8.5 manages PostgreSQL lifecycle.

**Namespace**: `data-analytics-test`

**Operator Pod**: `pgo-759bc8bfd8-bw8d8`

**Features**:
- Automated backups with pgBackRest
- High availability configuration
- Declarative PostgreSQL clusters
- Custom Resource Definitions (CRDs)

**Important Installation Note**:

The PostgresCluster CRD exceeds Kubernetes annotation size limits and required manual installation:

```bash
# This CRD must be applied with server-side apply
kubectl apply --server-side \
  -f kustomize/install/crd/bases/postgres-operator.crunchydata.com_postgresclusters.yaml
```

**Installed CRDs**:
- postgresclusters.postgres-operator.crunchydata.com
- crunchybridgeclusters.postgres-operator.crunchydata.com
- pgadmins.postgres-operator.crunchydata.com
- pgupgrades.postgres-operator.crunchydata.com

---

## 🗄️ Deployed Services

### PostgreSQL Database

**Cluster Name**: `hippo`
**Namespace**: `data-analytics-test`
**PostgreSQL Version**: 17
**Operator Version**: 5.8.5-0

#### Running Pods

| Pod Name | Status | Containers | Purpose |
|----------|--------|------------|---------|
| `hippo-instance1-n784-0` | Running | 4/4 | PostgreSQL primary instance |
| `hippo-repo-host-0` | Running | 2/2 | pgBackRest repository host |
| `hippo-backup-*` | Completed | 0/1 | Scheduled backup jobs |

#### Connection Information

```bash
# Internal Kubernetes Service
Host: hippo-primary.data-analytics-test.svc.cluster.local
Port: 5432
Database: zoo
Username: hippo
Password: .voP}5L{k3tb:5nwM76+}MXK

# Connection String
postgresql://hippo:.voP}5L{k3tb:5nwM76+}MXK@hippo-primary.data-analytics-test.svc.cluster.local:5432/zoo

# Short form (within data-analytics-test namespace)
postgresql://hippo:.voP}5L{k3tb:5nwM76+}MXK@hippo-primary:5432/zoo
```

#### Services

| Service Name | Type | Cluster IP | Port | Purpose |
|--------------|------|------------|------|---------|
| `hippo-primary` | ClusterIP | None | 5432 | Primary read-write endpoint |
| `hippo-replicas` | ClusterIP | 10.100.222.45 | 5432 | Read-only replicas |
| `hippo-ha` | ClusterIP | 10.100.87.54 | 5432 | High availability endpoint |
| `hippo-ha-config` | ClusterIP | None | - | Config sync |
| `hippo-pods` | ClusterIP | None | - | Pod-to-pod communication |

#### Storage

- **Data Volume**: 1Gi (gp2, ReadWriteOnce)
- **Backup Volume**: 1Gi (gp2, ReadWriteOnce)

#### Test Connection

```bash
# Run temporary psql pod
kubectl run -it --rm psql \
  --image=postgres:17 \
  --restart=Never \
  -n data-analytics-test \
  -- psql postgresql://hippo:.voP}5L{k3tb:5nwM76+}MXK@hippo-primary:5432/zoo

# Once connected
zoo=# \l       # List databases
zoo=# \dt      # List tables
zoo=# \q       # Quit
```

#### Backup Configuration

pgBackRest manages automated backups:

```bash
# Check backup status
kubectl get postgrescluster hippo -n data-analytics-test -o yaml

# View backup history
kubectl logs -n data-analytics-test hippo-repo-host-0 -c pgbackrest

# List completed backup jobs
kubectl get jobs -n data-analytics-test | grep backup
```

---

### pgAdmin

**Instance Name**: `rhino`
**Namespace**: `data-analytics-test`
**Version**: pgAdmin 4 v9.8
**Image**: crunchy-pgadmin4 (CrunchyData build)

#### Pod Information

- **Pod**: `pgadmin-98c4f0b3-4643-4985-9474-826f3c07dfd0-0`
- **Status**: Running (1/1)
- **Storage**: 1Gi (gp2)

#### Access Information

**External Access (LoadBalancer)**:
```
URL: http://a5794e22af859474bb379967f6f31a58-886910537.eu-central-1.elb.amazonaws.com
Email: rhino@example.com
Password: pgadmin
```

**Internal Access (within cluster)**:
```
Service: pgadmin.data-analytics-test.svc.cluster.local
Port: 80
```

#### Features

- **Auto-registered servers**: PostgreSQL cluster `hippo` is automatically added
- **Server groups**: "supply" group contains all PostgresClusters in namespace
- **Persistent storage**: 1Gi for session data and settings
- **Authentication**: Internal (email/password)

#### Configuration

Storage configuration in `kustomize/pgadmin/pgadmin.yaml`:
```yaml
dataVolumeClaimSpec:
  storageClassName: gp2
  accessModes:
  - "ReadWriteOnce"
  resources:
    requests:
      storage: 1Gi
```

---

### Grafana Internal

**Purpose**: Internal monitoring dashboards for team use only

**Namespace**: `grafana-internal`
**Image**: grafana/grafana:11.1.13
**Service Type**: ClusterIP (not exposed externally)

#### Pod Information

- **Pod**: `grafana-internal-85f9c84dbb-qcvpk`
- **Status**: Running (1/1)
- **Storage**: 5Gi (gp2)

#### Access Information

```bash
# Port Forward (required for access)
kubectl port-forward -n grafana-internal svc/grafana-internal 3000:3000

# Then visit: http://localhost:3000
# Username: admin
# Password: grafana-internal-pass
```

**Internal Cluster URL**:
```
http://grafana-internal.grafana-internal.svc.cluster.local:3000
```

#### Configuration

- **Authentication**: Admin user/password required
- **Anonymous Access**: Disabled
- **Network**: ClusterIP only (internal traffic)
- **Use Case**: Team dashboards, internal metrics, sensitive data

#### Environment Variables

```yaml
- GF_SERVER_ROOT_URL: http://grafana-internal:3000
- GF_PATHS_DATA: /data/grafana/data
- GF_SECURITY_ADMIN_USER__FILE: /conf/admin/username
- GF_SECURITY_ADMIN_PASSWORD__FILE: /conf/admin/password
```

---

### Grafana Public

**Purpose**: Public-facing dashboards with anonymous viewing enabled

**Namespace**: `grafana-public`
**Image**: grafana/grafana:11.1.13
**Service Type**: LoadBalancer (AWS ELB)

#### Pod Information

- **Pod**: `grafana-public-7757b74756-c4cq5`
- **Status**: Running (1/1)
- **Storage**: 5Gi (gp2)

#### Access Information

**External Access (LoadBalancer)**:
```
URL: http://ab20a054ba2e04f4784bbca1c10d01ec-448219164.eu-central-1.elb.amazonaws.com

Admin Login:
  Username: admin
  Password: grafana-public-pass

Anonymous Access: Enabled (Viewer role)
```

#### Configuration

- **Authentication**: Admin user/password for configuration
- **Anonymous Access**: Enabled with Viewer role (read-only)
- **Network**: LoadBalancer (publicly accessible)
- **Use Case**: Public dashboards, external monitoring, customer-facing metrics

#### Environment Variables

```yaml
- GF_SERVER_ROOT_URL: http://grafana-public:3000
- GF_PATHS_DATA: /data/grafana/data
- GF_SECURITY_ADMIN_USER__FILE: /conf/admin/username
- GF_SECURITY_ADMIN_PASSWORD__FILE: /conf/admin/password
- GF_AUTH_ANONYMOUS_ENABLED: "true"
- GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
```

#### Anonymous Access

Anonymous users can:
- ✅ View public dashboards
- ✅ Explore public data
- ✅ Refresh dashboards
- ❌ Edit dashboards
- ❌ Change settings
- ❌ Create/delete resources

---

## 🔐 Access Information

All credentials are stored in `.env` file (excluded from Git via `.gitignore`).

### Quick Reference

| Service | Access Method | Credentials |
|---------|---------------|-------------|
| **ArgoCD** | Port-forward 8080:443 | admin / 84DiV6o62IVgZ9KC |
| **PostgreSQL** | Internal service | hippo / .voP}5L{k3tb:5nwM76+}MXK |
| **pgAdmin** | LoadBalancer | rhino@example.com / pgadmin |
| **Grafana Internal** | Port-forward 3000:3000 | admin / grafana-internal-pass |
| **Grafana Public** | LoadBalancer | admin / grafana-public-pass |

### Detailed Credentials

#### ArgoCD
```bash
URL: https://localhost:8080 (kubectl port-forward)
Username: admin
Password: 84DiV6o62IVgZ9KC

# Port forward command
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

#### PostgreSQL
```bash
Host: hippo-primary.data-analytics-test.svc.cluster.local
Port: 5432
Database: zoo
Username: hippo
Password: .voP}5L{k3tb:5nwM76+}MXK

# Connection string
postgresql://hippo:.voP}5L{k3tb:5nwM76+}MXK@hippo-primary.data-analytics-test.svc.cluster.local:5432/zoo
```

#### pgAdmin
```bash
URL: http://a5794e22af859474bb379967f6f31a58-886910537.eu-central-1.elb.amazonaws.com
Email: rhino@example.com
Password: pgadmin
```

#### Grafana Internal
```bash
URL: http://localhost:3000 (kubectl port-forward)
Internal URL: http://grafana-internal.grafana-internal.svc.cluster.local:3000
Username: admin
Password: grafana-internal-pass

# Port forward command
kubectl port-forward -n grafana-internal svc/grafana-internal 3000:3000
```

#### Grafana Public
```bash
URL: http://ab20a054ba2e04f4784bbca1c10d01ec-448219164.eu-central-1.elb.amazonaws.com
Username: admin
Password: grafana-public-pass
Anonymous: Enabled (Viewer role)
```

---

## 🔄 GitOps Workflow

All infrastructure is managed through Git and ArgoCD using the GitOps methodology.

### Repository Structure

```
data-analytics-k8s/
├── .gitignore                    # Excludes .env and sensitive files
├── README.md                     # Original PGO examples README
├── DOCUMENTATION.md              # This file
├── .env                          # Credentials (NOT in Git)
│
├── argocd-apps/                  # ArgoCD Application definitions
│   ├── grafana-internal.yaml    # Internal Grafana app
│   ├── grafana-public.yaml      # Public Grafana app
│   ├── pgadmin.yaml              # pgAdmin app
│   ├── postgres-database.yaml   # PostgreSQL database app
│   └── postgres-operator.yaml   # PostgreSQL operator app
│
└── kustomize/                    # Kubernetes manifests (Kustomize)
    ├── grafana-internal/         # Internal Grafana instance
    │   ├── namespace.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── pvc.yaml
    │   ├── serviceaccount.yaml
    │   └── kustomization.yaml
    │
    ├── grafana-public/           # Public Grafana instance
    │   ├── namespace.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── pvc.yaml
    │   ├── serviceaccount.yaml
    │   └── kustomization.yaml
    │
    ├── install/                  # PostgreSQL Operator installation
    │   ├── crd/                  # Custom Resource Definitions
    │   ├── default/              # Default installation
    │   ├── namespace/            # Namespace configuration
    │   └── ...
    │
    ├── pgadmin/                  # pgAdmin configuration
    │   ├── pgadmin.yaml
    │   ├── pgadmin-service.yaml
    │   └── kustomization.yaml
    │
    └── postgres/                 # PostgreSQL database configuration
        ├── postgres.yaml
        └── kustomization.yaml
```

### GitOps Principles

1. **Declarative**: Entire system state defined in Git
2. **Versioned**: All changes tracked with Git history
3. **Immutable**: Never modify cluster directly
4. **Pulled**: ArgoCD pulls changes from Git
5. **Continuously Reconciled**: Cluster state matches Git automatically

### Making Changes

#### Step 1: Edit Manifests Locally

```bash
cd "/Users/noahokorie/data analytics"

# Example: Update PostgreSQL storage
vim kustomize/postgres/postgres.yaml

# Make your changes
```

#### Step 2: Test Locally (Optional)

```bash
# Validate with kubectl
kubectl apply --dry-run=client -k kustomize/postgres/

# Or validate with kustomize
kustomize build kustomize/postgres/
```

#### Step 3: Commit and Push

```bash
# Stage changes
git add kustomize/postgres/postgres.yaml

# Commit with descriptive message
git commit -m "Increase PostgreSQL storage from 1Gi to 5Gi"

# Push to GitHub
git push origin main
```

#### Step 4: ArgoCD Auto-Sync

ArgoCD automatically detects changes and syncs (usually within 3 minutes).

**Or manually trigger sync**:
```bash
# Via kubectl
kubectl patch application postgres-database -n argocd \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Via ArgoCD CLI
argocd app sync postgres-database
```

### ArgoCD Sync Policies

All applications use these sync policies:

```yaml
syncPolicy:
  automated:
    prune: true        # Remove resources deleted from Git
    selfHeal: true     # Revert manual changes to cluster
  syncOptions:
    - CreateNamespace=true  # Auto-create namespaces
```

**What this means**:
- ✅ Changes in Git are automatically applied to cluster
- ✅ Manual cluster changes are automatically reverted
- ✅ Deleted resources in Git are removed from cluster
- ✅ New namespaces are created automatically

### Deployment Flow

```
Developer → Edit Manifests → Git Commit → Git Push → GitHub
                                                         ↓
                                                   ArgoCD Polls
                                                         ↓
                                                   ArgoCD Syncs
                                                         ↓
                                              Kubernetes Cluster Updated
```

### ArgoCD Application Health States

| State | Meaning |
|-------|---------|
| **Healthy** | All resources running correctly |
| **Progressing** | Resources being created/updated |
| **Degraded** | Some resources have issues |
| **Suspended** | Application is paused |
| **Missing** | Resources not found in cluster |
| **Unknown** | Health cannot be determined |

### ArgoCD Sync States

| State | Meaning |
|-------|---------|
| **Synced** | Cluster matches Git |
| **OutOfSync** | Cluster differs from Git |
| **Unknown** | Sync status cannot be determined |

---

## 🛠️ Troubleshooting

### Common Issues and Solutions

#### 1. PostgresCluster CRD Installation Failure

**Symptom**:
```
Error: metadata.annotations: Too long: may not be more than 262144 bytes
```

**Cause**: The PostgresCluster CRD contains extensive OpenAPI validation schemas that exceed Kubernetes annotation size limits.

**Solution**:
```bash
# Use server-side apply
kubectl apply --server-side \
  -f kustomize/install/crd/bases/postgres-operator.crunchydata.com_postgresclusters.yaml
```

**Why this works**: Server-side apply stores the last-applied-configuration on the server instead of in annotations.

---

#### 2. Pod Stuck in Pending State

**Symptom**:
```
NAME                              READY   STATUS    RESTARTS   AGE
grafana-public-7757b74756-c4cq5   0/1     Pending   0          2m
```

**Diagnose**:
```bash
# Check pod events
kubectl describe pod -n grafana-public grafana-public-7757b74756-c4cq5

# Common errors:
# - "Insufficient cpu"
# - "Insufficient memory"
# - "Too many pods"
```

**Solution**:
```bash
# Scale up node group
aws eks update-nodegroup-config \
  --cluster-name data-analytics-cluster \
  --nodegroup-name managed-nodes \
  --scaling-config minSize=2,maxSize=4,desiredSize=3 \
  --region eu-central-1

# Wait for new node
kubectl get nodes -w
```

---

#### 3. PVC Stuck in Pending

**Symptom**:
```
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
grafanadata   Pending                                      gp2
```

**Diagnose**:
```bash
# Check PVC events
kubectl describe pvc -n grafana-internal grafanadata

# Check storage class
kubectl get storageclass gp2

# Verify EBS CSI driver
kubectl get pods -n kube-system | grep ebs-csi
```

**Common Causes**:
1. EBS CSI driver not installed
2. Missing storageClassName in PVC
3. IAM permissions missing

**Solution**:
```bash
# 1. Check EBS CSI driver addon
aws eks describe-addon \
  --cluster-name data-analytics-cluster \
  --addon-name aws-ebs-csi-driver \
  --region eu-central-1

# 2. Verify IAM role
aws iam get-role --role-name DataAnalyticsEBSCSIDriverRole

# 3. Add storageClassName if missing
kubectl edit pvc -n grafana-internal grafanadata
# Add: storageClassName: gp2
```

---

#### 4. ArgoCD Application OutOfSync

**Symptom**:
```
NAME                SYNC STATUS   HEALTH STATUS
postgres-operator   OutOfSync     Healthy
```

**Diagnose**:
```bash
# View diff
kubectl get application postgres-operator -n argocd -o yaml

# Check application details in UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Visit https://localhost:8080
```

**Causes**:
1. Manual changes to cluster resources
2. Git repository not up-to-date
3. CRD issues (like PostgresCluster)

**Solution**:
```bash
# Force sync
kubectl patch application postgres-operator -n argocd \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Or hard refresh
argocd app sync postgres-operator --force
```

---

#### 5. LoadBalancer Stuck in Pending

**Symptom**:
```
NAME             TYPE           EXTERNAL-IP   PORT(S)
grafana-public   LoadBalancer   <pending>     80:31516/TCP
```

**Diagnose**:
```bash
# Check service events
kubectl describe svc -n grafana-public grafana-public

# Check AWS Load Balancer Controller logs (if installed)
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

**Common Causes**:
1. AWS service quotas exceeded
2. Subnet configuration issues
3. Security group problems

**Solution**:
```bash
# Wait (can take 2-5 minutes)
kubectl get svc -n grafana-public grafana-public -w

# Check AWS console
# Services → EC2 → Load Balancers → Look for new CLB
```

---

#### 6. PostgreSQL Operator Not Watching CRDs

**Symptom**:
```
level=error msg="Could not wait for Cache to sync"
```

**Cause**: Operator started before CRD was fully registered

**Solution**:
```bash
# Restart operator
kubectl rollout restart deployment/pgo -n data-analytics-test

# Wait for ready
kubectl wait --for=condition=available \
  deployment/pgo -n data-analytics-test --timeout=60s
```

---

### Diagnostic Commands

#### Cluster Health

```bash
# Node status
kubectl get nodes -o wide

# Node resources (requires metrics-server)
kubectl top nodes

# Check critical pods
kubectl get pods -A | grep -v Running | grep -v Completed
```

#### Application Status

```bash
# All ArgoCD applications
kubectl get application -n argocd

# Specific application details
kubectl describe application <app-name> -n argocd

# Application logs
argocd app logs <app-name>
```

#### Pod Troubleshooting

```bash
# Get pods in all namespaces
kubectl get pods -A

# Describe pod (shows events)
kubectl describe pod -n <namespace> <pod-name>

# View pod logs
kubectl logs -n <namespace> <pod-name>

# Previous container logs (if crashed)
kubectl logs -n <namespace> <pod-name> --previous

# Execute into pod
kubectl exec -it -n <namespace> <pod-name> -- /bin/bash
```

#### Storage Troubleshooting

```bash
# List all PVCs
kubectl get pvc -A

# Describe PVC
kubectl describe pvc -n <namespace> <pvc-name>

# Check storage classes
kubectl get storageclass

# EBS CSI driver status
kubectl get pods -n kube-system | grep ebs-csi
kubectl get csidrivers
```

#### Network Troubleshooting

```bash
# List all services
kubectl get svc -A

# Test DNS resolution
kubectl run -it --rm debug \
  --image=busybox \
  --restart=Never \
  -- nslookup hippo-primary.data-analytics-test.svc.cluster.local

# Test connectivity
kubectl run -it --rm debug \
  --image=postgres:17 \
  --restart=Never \
  -- psql postgresql://hippo:.voP}5L{k3tb:5nwM76+}MXK@hippo-primary.data-analytics-test.svc.cluster.local:5432/zoo
```

#### Events

```bash
# All recent events
kubectl get events -A --sort-by='.lastTimestamp'

# Events in specific namespace
kubectl get events -n data-analytics-test --sort-by='.lastTimestamp'

# Watch events
kubectl get events -A --watch
```

---

## 🔧 Maintenance

### Regular Maintenance Tasks

#### 1. Monitor Resource Usage

```bash
# Node capacity
kubectl describe nodes

# Pod resource requests/limits
kubectl get pods -A -o json | jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, requests: .spec.containers[].resources.requests}'

# Check for pods near limits
kubectl top pods -A
```

#### 2. Review Backups

```bash
# PostgreSQL backup status
kubectl get postgrescluster hippo -n data-analytics-test -o yaml | grep -A 20 backups

# Backup job history
kubectl get jobs -n data-analytics-test | grep backup

# Backup logs
kubectl logs -n data-analytics-test hippo-repo-host-0 -c pgbackrest
```

#### 3. Update Container Images

```bash
# Update in Git repository
cd "/Users/noahokorie/data analytics"
vim kustomize/grafana-public/deployment.yaml
# Change: image: grafana/grafana:11.1.13 → grafana/grafana:11.2.0

# Commit and push
git add kustomize/grafana-public/deployment.yaml
git commit -m "Update Grafana to 11.2.0"
git push

# ArgoCD will auto-sync
```

#### 4. Certificate Rotation

ArgoCD generates certificates automatically. To rotate:

```bash
# Delete secrets to force regeneration
kubectl delete secret -n argocd argocd-server-tls

# Restart ArgoCD server
kubectl rollout restart deployment argocd-server -n argocd
```

#### 5. Clean Up Completed Jobs

```bash
# List completed jobs
kubectl get jobs -A --field-selector status.successful=1

# Delete old backup jobs (keep last 5)
kubectl delete job -n data-analytics-test \
  $(kubectl get jobs -n data-analytics-test -o name | grep backup | head -n -5)
```

### Scaling Operations

#### Scale Node Group

```bash
# Increase capacity
aws eks update-nodegroup-config \
  --cluster-name data-analytics-cluster \
  --nodegroup-name managed-nodes \
  --scaling-config minSize=3,maxSize=5,desiredSize=4 \
  --region eu-central-1

# Decrease capacity
aws eks update-nodegroup-config \
  --cluster-name data-analytics-cluster \
  --nodegroup-name managed-nodes \
  --scaling-config minSize=2,maxSize=3,desiredSize=2 \
  --region eu-central-1
```

#### Scale Deployments

```bash
# Scale Grafana replicas
kubectl scale deployment grafana-public -n grafana-public --replicas=2

# Or update in Git (recommended)
vim kustomize/grafana-public/deployment.yaml
# Change: replicas: 1 → replicas: 2
git add . && git commit -m "Scale Grafana to 2 replicas" && git push
```

### Backup and Recovery

#### Backup PostgreSQL Manually

```bash
# Trigger manual backup
kubectl annotate postgrescluster hippo -n data-analytics-test \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date +%Y-%m-%d-%H-%M-%S)"
```

#### Restore PostgreSQL

See CrunchyData documentation for detailed restore procedures:
https://access.crunchydata.com/documentation/postgres-operator/latest/tutorials/disaster-recovery/

#### Backup Grafana Dashboards

```bash
# Export from Grafana UI
# Settings → Import/Export → Export

# Or via API
GRAFANA_URL="http://localhost:3000"
GRAFANA_TOKEN="<admin-token>"

curl -H "Authorization: Bearer $GRAFANA_TOKEN" \
  "$GRAFANA_URL/api/dashboards/db/<dashboard-uid>" \
  > dashboard-backup.json
```

### Monitoring and Alerts

#### Set Up Monitoring Stack (Future)

Recommended additions:
- Prometheus for metrics
- Loki for logs
- Alertmanager for notifications
- Tempo for traces

```bash
# Can be added as ArgoCD applications
# Create manifests in kustomize/monitoring/
```

---

## 📊 Namespaces Overview

| Namespace | Purpose | Resources | Access |
|-----------|---------|-----------|--------|
| `argocd` | GitOps controller | ArgoCD components | Admin only |
| `data-analytics-test` | Data infrastructure | PostgreSQL, pgAdmin, PGO | Internal + External (pgAdmin) |
| `grafana-internal` | Internal monitoring | Grafana (ClusterIP) | Port-forward only |
| `grafana-public` | Public dashboards | Grafana (LoadBalancer) | Public + Admin |
| `kube-system` | Kubernetes system | DNS, CNI, CSI drivers | System |

---

## 🔒 Security Considerations

### Current Security Measures

✅ **Implemented**:
- IAM Roles for Service Accounts (IRSA) for AWS integration
- Kubernetes Secrets for sensitive data
- Credentials stored in `.env` (excluded from Git)
- PostgreSQL password generated by operator
- ArgoCD admin password rotation supported
- Namespace isolation for services

### Production Recommendations

⚠️ **To Implement**:

1. **TLS/SSL Encryption**
   - Configure SSL for LoadBalancers
   - Enable TLS for PostgreSQL connections
   - Use cert-manager for certificate management

2. **Network Policies**
   ```bash
   # Restrict namespace-to-namespace traffic
   # Create NetworkPolicy resources
   ```

3. **Secrets Management**
   - Integrate AWS Secrets Manager
   - Or use HashiCorp Vault
   - Rotate credentials regularly

4. **RBAC**
   - Implement fine-grained Role-Based Access Control
   - Create ServiceAccounts with minimal permissions
   - Use Pod Security Standards/Policies

5. **Image Security**
   - Scan container images for vulnerabilities
   - Use image pull policies
   - Implement image signing

6. **Audit Logging**
   - Enable Kubernetes audit logs
   - Monitor ArgoCD audit trail
   - Track database access logs

---

## 📚 Additional Resources

### Official Documentation

- [CrunchyData PostgreSQL Operator](https://access.crunchydata.com/documentation/postgres-operator/latest/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Grafana Documentation](https://grafana.com/docs/)
- [pgAdmin Documentation](https://www.pgadmin.org/docs/)
- [AWS EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

### Useful Links

- [Kustomize Documentation](https://kustomize.io/)
- [EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [eksctl Documentation](https://eksctl.io/)

### Community Support

- [CrunchyData Discord](https://discord.gg/BnsMEeaPBV)
- [ArgoCD Slack](https://argoproj.github.io/community/join-slack/)
- [Kubernetes Slack](https://slack.k8s.io/)

---

## 📝 Change Log

| Date | Change | Author |
|------|--------|--------|
| 2025-11-26 | Initial cluster setup and documentation | System |
| 2025-11-26 | Added PostgreSQL Operator and database | System |
| 2025-11-26 | Added pgAdmin with LoadBalancer | System |
| 2025-11-26 | Added Grafana Internal and Public instances | System |
| 2025-11-26 | Scaled cluster to 3 nodes | System |

---

**End of Documentation**

For questions or issues, please refer to the troubleshooting section or consult the official documentation links above.
