# OpenShift Connectivity Link Demo

A declarative GitOps project that bootstraps an OpenShift cluster with Red Hat Connectivity Link, Developer Hub, and Dev Spaces using OpenShift GitOps (Argo CD) and Kustomize.

## Architecture

This project uses a **two-phase bootstrap** model:

| Phase | What it does | How it's applied |
|-------|-------------|-----------------|
| **Phase 1** | Installs the OpenShift GitOps operator via OLM | Manual `oc apply` (one-time) |
| **Phase 2** | Deploys an ApplicationSet that auto-discovers features | Manual `oc apply` (one-time), then self-managed by Argo CD |

Once both phases are applied, Argo CD manages itself and all features from this Git repository. Adding a new feature is as simple as creating a new directory under `clusters/features/`.

### Features Included

| Feature | Namespace | Operator Package | Channel |
|---------|-----------|-----------------|---------|
| [Red Hat OpenShift Service Mesh 3](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.0) | `openshift-operators` | `servicemeshoperator3` | `stable` |
| [Red Hat Connectivity Link](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/) | `kuadrant-system` | `rhcl-operator` | `stable` |
| [Red Hat Developer Hub](https://docs.redhat.com/en/documentation/red_hat_developer_hub/) | `rhdh-operator` | `rhdh` | `fast` |
| [Red Hat OpenShift Dev Spaces](https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/) | `openshift-devspaces` | `devspaces` | `stable` |

## Prerequisites

- OpenShift Container Platform **4.19+** cluster
- `oc` CLI installed and authenticated as **cluster-admin**
- A fork of this repository pushed to GitHub (Argo CD needs to pull from it)

## Deploy

### Step 1 — Fork and configure

Fork this repository, then update the ApplicationSet to point at your fork:

```bash
# Clone your fork
git clone https://github.com/<YOUR_ORG>/openshift-connectivity-link-demo.git
cd openshift-connectivity-link-demo

# Update the repo URL in the ApplicationSet (two occurrences)
sed -i "s|YOUR_ORG|<your-github-org-or-user>|g" bootstrap/phase-2/applicationset.yaml

# Commit and push the change
git add bootstrap/phase-2/applicationset.yaml
git commit -m "Configure ApplicationSet repo URL"
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

### Step 3 — Deploy the ApplicationSet (Phase 2)

```bash
oc apply -k bootstrap/phase-2/
```

This creates:
- A `ClusterRoleBinding` granting the Argo CD application controller cluster-admin access
- An `ApplicationSet` that watches `clusters/features/*` in your repo and creates an Argo CD Application for each feature directory

### Step 4 — Verify

Check that Argo CD has created and synced all feature applications:

```bash
oc get applications -n openshift-gitops
```

You should see four applications: `service-mesh`, `connectivity-link`, `developer-hub`, and `dev-spaces`.

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

## Adding a new feature

1. Create a new directory under `clusters/features/<feature-name>/`
2. Add a `kustomization.yaml` and your manifests (Namespace, OperatorGroup, Subscription, CRs)
3. Commit and push to your repo
4. Argo CD will automatically detect the new directory and create an Application for it

## Removing a feature

1. Delete the feature directory from `clusters/features/`
2. Commit and push
3. Argo CD will prune the Application and its managed resources (automated sync with `prune: true` is enabled)

## Repository Structure

```
bootstrap/
  phase-1/            # GitOps operator install (Namespace, OperatorGroup, Subscription)
  phase-2/            # ApplicationSet + ClusterRoleBinding for Argo CD
clusters/
  features/
    service-mesh/      # OpenShift Service Mesh 3 (Istio) — required by Connectivity Link
    connectivity-link/ # Red Hat Connectivity Link operator
    developer-hub/     # Red Hat Developer Hub operator
    dev-spaces/        # Red Hat OpenShift Dev Spaces operator
```
