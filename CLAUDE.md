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
| Developer Hub | `clusters/features/developer-hub/` | `rhdh-operator` | `rhdh` | `fast` |
| Dev Spaces | `clusters/features/dev-spaces/` | `openshift-devspaces` | `devspaces` | `stable` |
| API Gateway | `clusters/features/api-gateway/` | `api-gateway` | — | — |

Service Mesh 3 is a prerequisite for Connectivity Link. It installs into `openshift-operators` (all-namespaces mode, no OperatorGroup needed) and deploys Istio into `istio-system` and IstioCNI into `istio-cni`. The API Gateway feature has no operator of its own — it creates a Kubernetes Gateway API `Gateway` (ClusterIP, accessed via OpenShift passthrough Routes) with Kuadrant protection policies (AuthPolicy, RateLimitPolicy). HTTPS uses the existing ROSA `*.apps` wildcard TLS cert copied into the `api-gateway` namespace.

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
