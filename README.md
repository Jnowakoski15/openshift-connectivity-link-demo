# OpenShift Connectivity Link Demo

A declarative GitOps project that bootstraps an OpenShift cluster with Red Hat Connectivity Link, Developer Hub, and Dev Spaces using OpenShift GitOps (Argo CD) and Kustomize. Supports both **ROSA (AWS)** and **ARO (Azure)** clusters.

## Architecture

This project uses a **two-phase bootstrap** model:

| Phase | What it does | How it's applied |
|-------|-------------|-----------------|
| **Phase 1** | Installs the OpenShift GitOps operator via OLM | Manual `oc apply` (one-time) |
| **Phase 2** | Deploys ApplicationSets that auto-discover features | Manual `oc apply` (one-time), then self-managed by Argo CD |

Once both phases are applied, Argo CD manages itself and all features from this Git repository. Phase 2 uses a cloud-specific overlay (`bootstrap/phase-2/rosa/` or `bootstrap/phase-2/aro/`) to deploy both a common-features ApplicationSet (cloud-agnostic operators) and a cloud-specific ApplicationSet (DNS, TLS, gateway config).

### Features Included

| Feature | Namespace | Operator Package | Channel |
|---------|-----------|-----------------|---------|
| [Red Hat OpenShift Service Mesh 3](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.0) | `openshift-operators` | `servicemeshoperator3` | `stable` |
| [Red Hat Connectivity Link](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/) | `kuadrant-system` | `rhcl-operator` | `stable` |
| [cert-manager](https://docs.openshift.com/container-platform/latest/security/cert_manager_operator/index.html) | `cert-manager-operator` | `openshift-cert-manager-operator` | `stable-v1` |
| [Red Hat Developer Hub](https://docs.redhat.com/en/documentation/red_hat_developer_hub/) | `rhdh-operator` | `rhdh` | `fast` |
| [Red Hat OpenShift Dev Spaces](https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/) | `openshift-devspaces` | `devspaces` | `stable` |
| API Gateway | `api-gateway` | — | — |
| Sample App (httpbin) | `sample-app` | — | — |

## Prerequisites

- OpenShift Container Platform **4.19+** cluster (ROSA or ARO)
- `oc` CLI installed and authenticated as **cluster-admin**
- A fork of this repository pushed to GitHub (Argo CD needs to pull from it)
- **ROSA**: An AWS account with Route53 access for DNS and TLS certificate management
- **ARO**: An Azure subscription with Azure DNS access for DNS and TLS certificate management
- A dedicated DNS zone for the gateway subdomain (see [DNS Setup](#dns-setup) below)

## Pre-Installation Setup

Before deploying, you must complete the following steps manually. These resources contain credentials and DNS configuration that cannot be stored in Git.

### DNS Setup

The API Gateway uses a dedicated DNS zone to avoid conflicts with the cluster's managed DNS zones.

#### ROSA (AWS)

1. **Create a dedicated public hosted zone** in Route53 for your gateway subdomain (e.g. `gw.<cluster-domain>`). This zone must be separate from any existing ROSA-managed zones (ROSA creates both public and private zones, which causes Kuadrant's DNS operator to fail with "multiple zones found").

2. **Add NS delegation records** in the parent public hosted zone pointing your gateway subdomain to the dedicated zone's nameservers.

3. **Update the manifests** with your zone ID and hostnames:
   - `clusters/aws/cert-manager/cluster-issuer.yaml` — set `hostedZoneID` to your dedicated zone ID
   - `clusters/aws/api-gateway/gateway.yaml` — set the listener hostname to `*.<your-gateway-subdomain>`
   - `clusters/aws/sample-app/httproute.yaml` — set the hostname to `sample-app.<your-gateway-subdomain>`

#### ARO (Azure)

1. **Create a dedicated Azure DNS public zone** for your gateway subdomain (e.g. `gw.apps.<cluster-domain>`).

2. **Add NS delegation records** in the parent zone pointing your gateway subdomain to the dedicated zone's nameservers.

3. **Update the manifests** with your zone name, Azure credentials, and hostnames:
   - `clusters/azure/cert-manager/cluster-issuer.yaml` — set `hostedZoneName`, `clientID`, `subscriptionID`, `tenantID`, `resourceGroupName`
   - `clusters/azure/api-gateway/gateway.yaml` — set the listener hostname to `*.<your-gateway-subdomain>`
   - `clusters/azure/sample-app/httproute.yaml` — set the hostname to `sample-app.<your-gateway-subdomain>`

### Credentials Secrets

#### ROSA (AWS)

Two Secrets must be created manually before Argo CD syncs.

**1. cert-manager Route53 credentials** (for DNS-01 ACME challenges):

```bash
oc create namespace cert-manager-operator

oc create secret generic aws-route53-credentials \
  --namespace=cert-manager-operator \
  --from-literal=access-key-id=<AWS_ACCESS_KEY_ID> \
  --from-literal=secret-access-key=<AWS_SECRET_ACCESS_KEY>
```

**2. DNSPolicy AWS credentials** (for Kuadrant DNS record management):

```bash
oc create namespace api-gateway

oc create secret generic aws-dns-credentials \
  --namespace=api-gateway \
  --type=kuadrant.io/aws \
  --from-literal=AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY> \
  --from-literal=AWS_REGION=<AWS_REGION>
```

> **Note:** The same IAM user can be used for both Secrets. The IAM user needs permissions for `route53:ChangeResourceRecordSets`, `route53:GetHostedZone`, `route53:ListHostedZones`, and `route53:ListResourceRecordSets` on the dedicated hosted zone.

#### ARO (Azure)

Two Secrets must be created manually before Argo CD syncs.

**1. cert-manager Azure DNS credentials** (for DNS-01 ACME challenges):

```bash
oc create namespace cert-manager-operator

oc create secret generic azure-dns-credentials \
  --namespace=cert-manager-operator \
  --from-literal=client-secret=<AZURE_CLIENT_SECRET>
```

**2. DNSPolicy Azure credentials** (for Kuadrant DNS record management):

```bash
oc create namespace api-gateway

oc create secret generic azure-dns-credentials \
  --namespace=api-gateway \
  --type=kuadrant.io/azure \
  --from-literal=AZURE_CLIENT_ID=<AZURE_CLIENT_ID> \
  --from-literal=AZURE_CLIENT_SECRET=<AZURE_CLIENT_SECRET> \
  --from-literal=AZURE_SUBSCRIPTION_ID=<AZURE_SUBSCRIPTION_ID> \
  --from-literal=AZURE_TENANT_ID=<AZURE_TENANT_ID>
```

> **Note:** The service principal needs the `DNS Zone Contributor` role on the dedicated Azure DNS zone.

## Deploy

### Step 1 — Fork and configure

Fork this repository, then update the ApplicationSets to point at your fork:

```bash
# Clone your fork
git clone https://github.com/<YOUR_ORG>/openshift-connectivity-link-demo.git
cd openshift-connectivity-link-demo

# Update the repo URLs in all ApplicationSets
find bootstrap/phase-2 -name '*.yaml' -exec \
  sed -i "s|Jnowakoski15|<your-github-org-or-user>|g" {} +

# Commit and push the change
git add bootstrap/phase-2/
git commit -m "Configure ApplicationSet repo URLs"
git push
```

### Step 2 — Install the GitOps operator (Phase 1)

```bash
oc apply -k bootstrap/phase-1/
```

Wait for the operator to be ready:

```bash
oc wait --for=condition=CatalogSourcesUnhealthy=False \
  subscription/openshift-gitops-operator \
  -n openshift-gitops-operator \
  --timeout=300s
```

Verify the GitOps operator pod is running and the default Argo CD instance is created:

```bash
oc get pods -n openshift-gitops-operator
oc get pods -n openshift-gitops
```

Wait until all pods in `openshift-gitops` are `Running` before proceeding. This typically takes 2-3 minutes.

### Step 3 — Deploy the ApplicationSets (Phase 2)

Choose the overlay matching your cloud platform:

```bash
# For ROSA (AWS):
oc apply -k bootstrap/phase-2/rosa/

# For ARO (Azure):
oc apply -k bootstrap/phase-2/aro/
```

This creates:
- A `ClusterRoleBinding` granting the Argo CD application controller cluster-admin access
- A `common-features` ApplicationSet that discovers cloud-agnostic features in `clusters/features/*`
- A cloud-specific ApplicationSet (`aws-features` or `azure-features`) that discovers features in `clusters/aws/*` or `clusters/azure/*`

### Step 4 — Verify

Check that Argo CD has created and synced all feature applications:

```bash
oc get applications -n openshift-gitops
```

You should see seven applications: `service-mesh`, `connectivity-link`, `developer-hub`, `dev-spaces`, and three cloud-prefixed applications (e.g., `aws-api-gateway`, `aws-cert-manager`, `aws-sample-app` on ROSA or `azure-api-gateway`, `azure-cert-manager`, `azure-sample-app` on ARO).

Watch them sync:

```bash
oc get applications -n openshift-gitops -w
```

All applications should reach `Synced` and `Healthy` status. The operators will take several minutes to install via OLM after the initial sync.

### Step 5 — Access the Argo CD console

```bash
# Get the Argo CD route
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'

# Get the admin password
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=- --keys=admin.password 2>/dev/null
```

Log in with username `admin` and the extracted password.

## Testing the Sample App API

The sample app deploys an httpbin instance behind the API Gateway with API key authentication and tiered rate limiting.

### Verify authentication is enforced

A request without an API key should return `401 Unauthorized`:

```bash
curl -s https://sample-app.<your-gateway-subdomain>/get
```

### Make an authenticated request

Pass the API key via the `Authorization` header with the `APIKEY` prefix:

```bash
curl -H "Authorization: APIKEY demo-api-key-sample-app-12345" \
  https://sample-app.<your-gateway-subdomain>/get
```

A successful response returns HTTP 200 with JSON containing request headers, origin, and URL.

### API key approval workflow

The sample app's APIProduct uses `approvalMode: manual`. The workflow is:

1. An `APIKey` and `APIKeyRequest` are created (declared in Git or via the dev portal)
2. An admin approves the request through the Kuadrant console plugin at **Connectivity Link → API Key Approvals**
3. Once approved, the APIKey status transitions to `APPROVED: True`

Check approval status:

```bash
oc get apikey -n sample-app
```

### Rate limit tiers

The PlanPolicy defines three tiers. The demo key uses the **bronze** tier:

| Tier | Daily Limit | Monthly Limit |
|------|-------------|---------------|
| Gold | 10,000 | 250,000 |
| Silver | 1,000 | 25,000 |
| Bronze | 100 | 2,000 |

## Adding a new feature

**Cloud-agnostic features** (operators, CRs with no cloud-specific config):
1. Create a new directory under `clusters/features/<feature-name>/`
2. Add a `kustomization.yaml` and your manifests
3. Commit and push — Argo CD will auto-discover it

**Cloud-specific features** (DNS, TLS, cloud credentials):
1. Create a base in `clusters/base/<feature-name>/` with shared resources
2. Create overlays in `clusters/aws/<feature-name>/` and `clusters/azure/<feature-name>/` referencing the base
3. Commit and push — the cloud-specific ApplicationSet will discover it

## Removing a feature

1. Delete the feature directory (and its base/overlay counterparts if cloud-specific)
2. Commit and push
3. Argo CD will prune the Application and its managed resources (automated sync with `prune: true` is enabled)

## Repository Structure

```
bootstrap/
  phase-1/                # GitOps operator install (Namespace, OperatorGroup, Subscription)
  phase-2/
    base/                 # ClusterRoleBinding + common-features ApplicationSet
    rosa/                 # Base + AWS-specific ApplicationSet
    aro/                  # Base + Azure-specific ApplicationSet
clusters/
  features/               # Cloud-agnostic features (auto-discovered by common ApplicationSet)
    service-mesh/         # OpenShift Service Mesh 3 (Istio) — required by Connectivity Link
    connectivity-link/    # Red Hat Connectivity Link operator
    developer-hub/        # Red Hat Developer Hub operator
    dev-spaces/           # Red Hat OpenShift Dev Spaces operator
  base/                   # Shared base configs for cloud-specific features
    api-gateway/          # Namespace, AuthPolicy, RateLimitPolicy, TLSPolicy
    cert-manager/         # Namespace, OperatorGroup, Subscription
    sample-app/           # Deployment, Service, AuthPolicy, PlanPolicy, API keys
  aws/                    # AWS/ROSA overlays (auto-discovered by aws-features ApplicationSet)
    api-gateway/          # Gateway (ROSA hostname) + DNSPolicy (Route53)
    cert-manager/         # CertManager (DNS workaround) + ClusterIssuer (Route53)
    sample-app/           # HTTPRoute (ROSA hostname)
  azure/                  # Azure/ARO overlays (auto-discovered by azure-features ApplicationSet)
    api-gateway/          # Gateway (ARO hostname) + DNSPolicy (Azure DNS)
    cert-manager/         # CertManager + ClusterIssuer (Azure DNS)
    sample-app/           # HTTPRoute (ARO hostname)
```
