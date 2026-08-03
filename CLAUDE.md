# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Declarative GitOps project for OpenShift 4.19+ using Kustomize and Argo CD (OpenShift GitOps 1.21). Provisions a cluster with Service Mesh 3, Connectivity Link, Developer Hub, and Dev Spaces through a two-phase bootstrap. Supports both ROSA (AWS) and ARO (Azure) clusters via cloud-specific Kustomize overlays. Hosted on GitHub.

## Architecture

### Two-Phase Bootstrap

**Phase 1** (`bootstrap/phase-1/`): Installs the OpenShift GitOps operator via OLM into `openshift-gitops-operator` namespace. Applied manually once with `oc apply -k`. The operator auto-creates the `openshift-gitops` namespace with the default Argo CD instance.

**Phase 2** (`bootstrap/phase-2/`): Grants cluster-admin to the Argo CD application controller and deploys ApplicationSets. The base (`bootstrap/phase-2/base/`) contains the `common-features` ApplicationSet that discovers `clusters/features/*` (cloud-agnostic features). Cloud-specific overlays add a second ApplicationSet:
- **ROSA**: `oc apply -k bootstrap/phase-2/rosa/` adds `aws-features` ApplicationSet discovering `clusters/aws/*`
- **ARO**: `oc apply -k bootstrap/phase-2/aro/` adds `azure-features` ApplicationSet discovering `clusters/azure/*`

Applied manually once; self-managed by Argo CD afterward.

### Feature Directory Convention

Cloud-agnostic features live at `clusters/features/<feature-name>/`. Cloud-specific features use a base + overlay pattern: shared resources in `clusters/base/<feature-name>/`, cloud-specific resources in `clusters/aws/<feature-name>/` or `clusters/azure/<feature-name>/`. Each overlay's `kustomization.yaml` references its base and adds cloud-specific manifests (gateway hostname, DNS policy, TLS issuer). Standard pattern per feature: `namespace.yaml` + `operator-group.yaml` + `subscription.yaml` + CR. Resources use Argo CD sync wave annotations (wave 0: namespaces/operatorgroups, wave 1: subscriptions, wave 2: CRs with `SkipDryRunOnMissingResource`). The ApplicationSets create one Argo CD Application per directory with automated sync, prune, self-heal, and retry policy enabled.

### Current Features

**Cloud-Agnostic Features** (`clusters/features/`):

| Feature | Directory | Namespace | OLM Package | Channel |
|---------|-----------|-----------|-------------|---------|
| Service Mesh 3 | `clusters/features/service-mesh/` | `openshift-operators` | `servicemeshoperator3` | `stable` |
| Connectivity Link | `clusters/features/connectivity-link/` | `kuadrant-system` | `rhcl-operator` | `stable` |
| Developer Hub | `clusters/features/developer-hub/` | `rhdh-operator` | `rhdh` | `fast` |
| Dev Spaces | `clusters/features/dev-spaces/` | `openshift-devspaces` | `devspaces` | `stable` |

**Cloud-Specific Features** (base in `clusters/base/`, overlays in `clusters/aws/` or `clusters/azure/`):

| Feature | Base | AWS Overlay | Azure Overlay | Namespace |
|---------|------|-------------|---------------|-----------|
| cert-manager | `clusters/base/cert-manager/` | `clusters/aws/cert-manager/` | `clusters/azure/cert-manager/` | `cert-manager-operator` |
| API Gateway | `clusters/base/api-gateway/` | `clusters/aws/api-gateway/` | `clusters/azure/api-gateway/` | `api-gateway` |
| Sample App | `clusters/base/sample-app/` | `clusters/aws/sample-app/` | `clusters/azure/sample-app/` | `sample-app` |

Service Mesh 3 is a prerequisite for Connectivity Link. It installs into `openshift-operators` (all-namespaces mode, no OperatorGroup needed) and deploys Istio into `istio-system` and IstioCNI into `istio-cni`. The API Gateway feature has no operator of its own — it creates a Kubernetes Gateway API `Gateway` (HTTPS-only LoadBalancer) with Kuadrant protection policies (AuthPolicy, RateLimitPolicy, TLSPolicy, DNSPolicy). TLS certificates are provisioned by cert-manager via Let's Encrypt DNS-01 (Route53 on AWS, Azure DNS on Azure). The gateway uses a deny-by-default AuthPolicy (OPA `allow = false`) that individual apps override at the HTTPRoute level. The sample-app feature deploys httpbin (go-httpbin) with an HTTPRoute and an allow-all AuthPolicy override.

### DNS Architecture

Each cloud uses a dedicated DNS zone for the gateway hostname to avoid conflicts with the cluster's managed DNS zones.

**ROSA (AWS)**: ROSA clusters have both public and private Route53 hosted zones for the same domain, causing Kuadrant's DNS operator to fail with "multiple zones found". A **dedicated public Route53 hosted zone** (`gw.rosa.rosa-pfqsf.to0l.p3.openshiftapps.com`, zone `Z0688131ONQ3PCUBTGDM`) is used with NS delegation records in the parent public zone. The cert-manager CertManager CR includes `--dns01-recursive-nameservers-only` and `--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53` to avoid split-horizon DNS issues.

**ARO (Azure)**: A **dedicated Azure DNS public zone** (`gw.apps.wz405duy.eastus.aroapp.io`) in resource group `openenv-nfv5d` with NS delegation from the parent zone. ARO does not require the split-horizon DNS workaround.

### Out-of-Band Prerequisites

**ROSA (AWS)** — Two Secrets must be created manually:

1. `aws-route53-credentials` in `cert-manager` namespace (for the ClusterIssuer DNS-01 solver). Keys: `access-key-id`, `secret-access-key`.
2. `aws-dns-credentials` in `api-gateway` namespace (for DNSPolicy). Type: `kuadrant.io/aws`. Keys: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`.

Plus: Dedicated Route53 hosted zone and NS delegation in the parent zone.

**ARO (Azure)** — Two Secrets must be created manually:

1. `azure-dns-credentials` in `cert-manager` namespace (for the ClusterIssuer DNS-01 solver). Keys: `client-secret` (Azure service principal client secret).
2. `azure-dns-credentials` in `api-gateway` namespace (for DNSPolicy). Type: `kuadrant.io/azure`. Keys: `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`.

Plus: Dedicated Azure DNS public zone and NS delegation in the parent zone.

## Common Commands

```bash
# Apply Phase 1 bootstrap (requires cluster-admin, run once)
oc apply -k bootstrap/phase-1/

# Apply Phase 2 for ROSA (after GitOps operator is ready)
oc apply -k bootstrap/phase-2/rosa/

# Apply Phase 2 for ARO (after GitOps operator is ready)
oc apply -k bootstrap/phase-2/aro/

# Validate kustomize output locally
oc kustomize bootstrap/phase-2/rosa/
oc kustomize bootstrap/phase-2/aro/
oc kustomize clusters/features/<feature-name>/
oc kustomize clusters/aws/<feature-name>/
oc kustomize clusters/azure/<feature-name>/

# Check Argo CD application sync status
oc get applications -n openshift-gitops
```

## Key Constraints

- **Strictly declarative**: No shell scripts, no `oc create`, no imperative commands. Everything must be Argo CD-syncable.
- **Kustomize-native**: Every deployable directory needs a `kustomization.yaml`. Use `resources:` to list manifests.
- **OLM pattern**: Operators install via `Subscription` + `OperatorGroup`, not direct Deployments.
- **Namespace isolation**: Each feature declares its own namespace. Namespace-scoped resources must set their namespace explicitly.
- **All manifests target `redhat-operators` catalog** in `openshift-marketplace`.
