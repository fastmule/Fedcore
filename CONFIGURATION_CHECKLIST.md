# App Factory Platform - Configuration Checklist

This document tracks all configuration values that need to be provided before the platform can be fully deployed.

## Table of Contents
- [AWS Configuration](#aws-configuration)
- [Azure Configuration](#azure-configuration)
- [Splunk Configuration](#splunk-configuration)
- [GitHub Secrets](#github-secrets)
- [HashiCorp Vault](#hashicorp-vault)
- [Container Registry](#container-registry)
- [Tenant Management](#tenant-management)

---

## AWS Configuration

### Cluster: fedcore-prod-use1

**File:** `platform/clusters/fedcore-prod-use1/cluster.yaml`

| Field | Current Value | Status | Notes |
|-------|---------------|--------|-------|
| `aws.account_id` | `123456789012` | ⚠️ PLACEHOLDER | Replace with actual AWS account ID |

**How to get the values:**
```bash
# Get AWS Account ID
aws sts get-caller-identity --query Account --output text
```

### IAM Roles Required

**ACK IAM Controller Role:** `fedcore-prod-use1-ack-iam-controller`

This role must be created with:
- Trust relationship with Pod Identity (pods.eks.amazonaws.com service principal)
- ServiceAccount: `ack-iam-controller` in namespace `ack-system`
- Permissions to manage IAM policies, roles, and permission boundaries

**Reference:** `platform/components/ack-iam-controller/overlays/cloud/aws/values.yaml`

---

## Azure Configuration

### Cluster: fedcore-prod-azeus

**File:** `platform/clusters/fedcore-prod-azeus/cluster.yaml`

| Field | Current Value | Status | Notes |
|-------|---------------|--------|-------|
| `azure.subscription_id` | `00000000-0000-0000-0000-000000000000` | ⚠️ PLACEHOLDER | Replace with actual Azure subscription ID |
| `azure.tenant_id` | `87654321-4321-4321-4321-210987654321` | ⚠️ PLACEHOLDER | Replace with Azure AD tenant ID |
| `azure.oidc_issuer` | `https://eastus.oic.prod-aks.azure.com/...` | ⚠️ PLACEHOLDER | Get from AKS cluster details |

**How to get the values:**
```bash
# Get Azure Subscription ID
az account show --query id --output tsv

# Get Azure Tenant ID
az account show --query tenantId --output tsv

# Get AKS OIDC Issuer
az aks show --name fedcore-prod-aks --resource-group fedcore-prod-eastus-rg \
  --query "oidcIssuerProfile.issuerUrl" --output tsv
```

---

## Splunk Configuration

### Splunk HEC Endpoints

**Files:**
- `platform/components/splunk-connect/overlays/cloud/aws/splunk-hec-config.yaml`
- `platform/components/splunk-connect/overlays/cloud/azure/splunk-hec-config.yaml`
- `platform/components/splunk-connect/overlays/cloud/onprem/splunk-hec-config.yaml`

| Cloud | Current Value | Status | Notes |
|-------|---------------|--------|-------|
| AWS | `splunk-hec-aws.yourorg.gov` | ⚠️ PLACEHOLDER | Replace with actual AWS Splunk HEC endpoint |
| Azure | Not configured | ❌ MISSING | Add Azure Splunk HEC endpoint |
| On-prem | Not configured | ❌ MISSING | Add on-prem Splunk HEC endpoint |

### Splunk Index Routing Strategy

**Decision Required:** Choose between:

1. **Single index per cloud** (simpler)
   - Example: `k8s_fedcore_aws`, `k8s_fedcore_azure`, `k8s_fedcore_onprem`
   - Configure in each cloud overlay

2. **Tenant-specific indexes** (more granular)
   - Example: `k8s_fedcore_tenant_acme`, `k8s_fedcore_tenant_test`
   - Requires Splunk admin to pre-create indexes
   - Uses `indexFields` in base config for automatic routing

**Current configuration:** Tenant-specific (base config uses `indexFields`)

### Required Values

- `SPLUNK_HEC_HOST`: Full hostname with protocol (e.g., `https://splunk-hec.yourorg.gov`)
- `SPLUNK_HEC_TOKEN`: HEC token from Splunk admin (UUID format)
- Port: `8088` (default)
- Protocol: `https`
- SSL verification: `false` (set to `true` if proper certs)

---

## GitHub Secrets

### Repository Secrets (Actions → Settings → Secrets)

**For AWS Deployments:**

| Secret Name | Purpose | Where Used |
|-------------|---------|------------|
| `AWS_ACCESS_KEY_ID` | AWS credentials for kubectl access | `.github/workflows/build-and-publish.yaml:322` |
| `AWS_SECRET_ACCESS_KEY` | AWS credentials for kubectl access | `.github/workflows/build-and-publish.yaml:323` |

**For Azure Deployments:**

| Secret Name | Purpose | Where Used |
|-------------|---------|------------|
| `AZURE_CLIENT_ID` | Service principal client ID | `.github/workflows/build-and-publish.yaml:333` |
| `AZURE_CLIENT_SECRET` | Service principal secret | `.github/workflows/build-and-publish.yaml:334` |
| `AZURE_TENANT_ID` | Azure AD tenant ID | `.github/workflows/build-and-publish.yaml:335` |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID | `.github/workflows/build-and-publish.yaml:336` |

**For On-Prem Deployments:**

| Secret Name | Purpose | Where Used |
|-------------|---------|------------|
| `KUBECONFIG` | Base64-encoded kubeconfig file | `.github/workflows/build-and-publish.yaml:360` |

**How to create base64-encoded kubeconfig:**
```bash
cat ~/.kube/config | base64 -w 0
```

### GitHub Environment Secrets

**Splunk HEC Credentials** (per environment: fedcore-lab-01, fedcore-prod-azeus, fedcore-prod-use1)

| Secret Name | Purpose | Where Used |
|-------------|---------|------------|
| `SPLUNK_HEC_HOST` | Splunk HEC endpoint hostname | `.github/workflows/build-and-publish.yaml:386` |
| `SPLUNK_HEC_TOKEN` | Splunk HEC token | `.github/workflows/build-and-publish.yaml:387` |

**To configure:**
1. Go to repository Settings → Environments
2. Create/edit environment matching cluster name (e.g., `fedcore-prod-use1`)
3. Add environment secrets

---

## Tenant Management

### Existing Tenants

| Tenant | Cluster | File | Status |
|--------|---------|------|--------|
| `test-tenant` | fedcore-lab-01 | `platform/clusters/fedcore-lab-01/tenants/test-tenant-onboarding.yaml` | ✅ OK |
| `acme` | fedcore-prod-azeus | `platform/clusters/fedcore-prod-azeus/tenants/acme-onboarding.yaml` | ✅ OK |
| `acme` | fedcore-prod-use1 | ❌ MISSING | ⚠️ NEEDS CREATION |

### Action Required: Create acme tenant in fedcore-prod-use1

The `acme` tenant exists in Azure but is missing from AWS cluster.

**To create:**
```bash
cp platform/clusters/fedcore-prod-azeus/tenants/acme-onboarding.yaml \
   platform/clusters/fedcore-prod-use1/tenants/acme-onboarding.yaml

# Update metadata.labels.platform.fedcore.io/cluster to "fedcore-prod-use1"
```

---

## Validation

Once all configuration values are provided, validate with:

```bash
# Run full validation
./scripts/validate.sh

# Check specific cluster bootstrap
./scripts/bootstrap.sh platform/clusters/fedcore-prod-use1

# Check specific RGD artifact
./scripts/build.sh platform/rgds/webapps platform/clusters/fedcore-prod-use1
```

---

## Checklist Progress

- [ ] AWS Account ID configured
- [ ] AWS ACK IAM role created with Pod Identity trust
- [ ] Azure Subscription ID configured
- [ ] Azure Tenant ID configured
- [ ] Azure OIDC Issuer configured (for Workload Identity)
- [ ] Splunk HEC endpoints configured (AWS, Azure, on-prem)
- [ ] Splunk index routing strategy decided
- [ ] GitHub secret: AWS_ACCESS_KEY_ID
- [ ] GitHub secret: AWS_SECRET_ACCESS_KEY
- [ ] GitHub secret: AZURE_CLIENT_ID
- [ ] GitHub secret: AZURE_CLIENT_SECRET
- [ ] GitHub secret: AZURE_TENANT_ID
- [ ] GitHub secret: AZURE_SUBSCRIPTION_ID
- [ ] GitHub secret: KUBECONFIG (for on-prem)
- [ ] GitHub environment secrets: SPLUNK_HEC_HOST (per cluster)
- [ ] GitHub environment secrets: SPLUNK_HEC_TOKEN (per cluster)
- [ ] ACME tenant added to fedcore-prod-use1
- [ ] Full validation passed