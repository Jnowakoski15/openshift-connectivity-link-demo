# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Declarative GitOps project for OpenShift 4.19+ using Kustomize and Argo CD (OpenShift GitOps 1.21). Provisions a cluster with Service Mesh 3, Connectivity Link, Developer Hub, and Dev Spaces through a two-phase bootstrap. Hosted on GitHub.

## Architecture

### Two-Phase Bootstrap

**Phase 1** (`bootstrap/phase-1/`): Installs the OpenShift GitOps operator via OLM into `openshift-gitops-operator` namespace. Applied manually once with `oc apply -k`. The operator auto-creates the `openshift-gitops` namespace with the default Argo CD instance.

**Phase 2** (`bootstrap/phase-2/`): Grants cluster-admin to the Argo CD application controller and deploys an `ApplicationSet` (git directory generator, `goTemplate: true`) that auto-discovers `clusters/features/*`. Applied manually once; self-managed by Argo CD afterward.

### Feature Directory Convention

Each feature lives at `clusters/features/<feature-name>/` with a `kustomization.yaml` entry point. Standard pattern per feature: `namespace.yaml` + `operator-group.yaml` + `subscription.yaml` + CR. Resources use Argo CD sync wave annotations (wave 0: namespaces/operatorgroups, wave 1: subscriptions, wave 2: CRs with `SkipDryRunOnMissingResource`). The ApplicationSet creates one Argo CD Application per directory with automated sync, prune, self-heal, and retry policy enabled.

### Current Features

| Feature | Directory | Namespace | OLM Package | Channel |
|---------|-----------|-----------|-------------|---------|
| Service Mesh 3 | `clusters/features/service-mesh/` | `openshift-operators` | `servicemeshoperator3` | `stable` |
| Connectivity Link | `clusters/features/connectivity-link/` | `kuadrant-system` | `rhcl-operator` | `stable` |
| cert-manager | `clusters/features/cert-manager/` | `cert-manager-operator` | `openshift-cert-manager-operator` | `stable-v1` |
| Developer Hub | `clusters/features/developer-hub/` | `rhdh-operator` | `rhdh` | `fast` |
| Dev Spaces | `clusters/features/dev-spaces/` | `openshift-devspaces` | `devspaces` | `stable` |
| API Gateway | `clusters/features/api-gateway/` | `api-gateway` | — | — |
| Sample App | `clusters/features/sample-app/` | `sample-app` | — | — |

Service Mesh 3 is a prerequisite for Connectivity Link. It installs into `openshift-operators` (all-namespaces mode, no OperatorGroup needed) and deploys Istio into `istio-system` and IstioCNI into `istio-cni`. The API Gateway feature has no operator of its own — it creates a Kubernetes Gateway API `Gateway` (HTTPS-only LoadBalancer) with Kuadrant protection policies (AuthPolicy, RateLimitPolicy, TLSPolicy, DNSPolicy). TLS certificates are provisioned by cert-manager via Let's Encrypt (Route53 DNS-01). DNSPolicy manages Route53 records pointing `*.gw.rosa.rosa-pfqsf.to0l.p3.openshiftapps.com` to the gateway's NLB. The cert-manager CertManager CR is configured with `--dns01-recursive-nameservers-only` and `--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53` to avoid split-horizon DNS issues on ROSA clusters. The gateway uses a deny-by-default AuthPolicy (OPA `allow = false`) that individual apps override at the HTTPRoute level. The sample-app feature deploys httpbin (go-httpbin) with an HTTPRoute and an allow-all AuthPolicy override.

### DNS Architecture

ROSA clusters have both public and private Route53 hosted zones for the same domain, causing Kuadrant's DNS operator to fail with "multiple zones found". To work around this, a **dedicated public hosted zone** (`gw.rosa.rosa-pfqsf.to0l.p3.openshiftapps.com`, zone `Z0688131ONQ3PCUBTGDM`) is used with NS delegation records in the parent public zone. The gateway hostname `*.gw.rosa...` is one level below the zone apex, avoiding the "apex domain not allowed" restriction.

### Out-of-Band Prerequisites

Two AWS credentials Secrets must be created manually before Argo CD syncs:

1. `aws-route53-credentials` in `cert-manager-operator` namespace (for the ClusterIssuer DNS-01 solver). Keys: `access-key-id`, `secret-access-key`.
2. `aws-dns-credentials` in `api-gateway` namespace (for DNSPolicy). Type: `kuadrant.io/aws`. Keys: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`.

The dedicated Route53 hosted zone and NS delegation in the parent zone must also be created manually.

### ApplicationSet Repo URL

The `repoURL` in `bootstrap/phase-2/applicationset.yaml` contains a `YOUR_ORG` placeholder. Users must replace it with their GitHub org/user before deploying.

## Common Commands

```bash
# Apply Phase 1 bootstrap (requires cluster-admin, run once)
oc apply -k bootstrap/phase-1/

# Apply Phase 2 after GitOps operator is ready
oc apply -k bootstrap/phase-2/

# Validate kustomize output locally
oc kustomize bootstrap/phase-1/
oc kustomize clusters/features/<feature-name>/

# Dry-run a feature against the cluster
oc apply -k clusters/features/<feature-name>/ --dry-run=server

# Check Argo CD application sync status
oc get applications -n openshift-gitops
```

## Key Constraints

- **Strictly declarative**: No shell scripts, no `oc create`, no imperative commands. Everything must be Argo CD-syncable.
- **Kustomize-native**: Every deployable directory needs a `kustomization.yaml`. Use `resources:` to list manifests.
- **OLM pattern**: Operators install via `Subscription` + `OperatorGroup`, not direct Deployments.
- **Namespace isolation**: Each feature declares its own namespace. Namespace-scoped resources must set their namespace explicitly.
- **All manifests target `redhat-operators` catalog** in `openshift-marketplace`.
