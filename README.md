# Service Enablement: App Platform Software Factory — GitOps

This repository contains the GitOps configuration for the **Software Factory** cluster.
ArgoCD on the factory cluster watches this repo and syncs whatever is declared here onto the cluster automatically.

## How it works

```
Developer pushes code
      │
      ▼
GitLab CI pipeline runs
      │
      ▼
Pipeline updates manifests in this repo
      │
      ▼
ArgoCD detects the change and syncs to the factory cluster
```

## Repository Structure

```
example-app/
  namespace.yaml     Namespace for the example application
  deployment.yaml    Deployment using Red Hat UBI httpd image
  service.yaml       ClusterIP service
  route.yaml         OpenShift Route with TLS edge termination

ansible/
  bootstrap.yml      Playbook for imperative ETX bootstrap steps
  requirements.yml   Ansible collection dependencies (kubernetes.core)

etx-infra-app/
  migrated-source/   ETX infra GitOps content migrated from etx_app_platform_bootstrap
  base/              Reserved kustomize base for RHPDS-specific ETX infra deltas
  overlays/factory/  Factory overlay for ETX infra deltas

etx-control-plane-app/
  migrated-source/   ETX platform GitOps content migrated from etx_app_platform_bootstrap
  base/              Reserved kustomize base for ETX control-plane deltas
  overlays/factory/  Factory overlay for ETX control-plane deltas

docs/
  etx-migration-analysis.md
  catalog-item-integration.md
```

## Getting Started

### 1. Fork this repo into your GitLab instance

During the lab, you will mirror this repo into the GitLab instance running on your factory cluster so your CI pipeline can commit changes back to it.

### 2. Point ArgoCD at your fork

Create an ArgoCD `Application` that points to your GitLab fork:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://<your-gitlab>/service-enablement-app-platform-factory-gitops.git
    targetRevision: main
    path: example-app
  destination:
    server: https://kubernetes.default.svc
    namespace: example-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3. Run the Ansible bootstrap

The `ansible/bootstrap.yml` playbook is the ETX bootstrap entrypoint. It handles
the imperative work that should not be committed as static GitOps state:

- preflight discovery of cluster domain, API URL, and GUID
- cluster default SSO isolation guardrail
- base OpenShift GitOps readiness and ETX health-check/RBAC patching
- optional repository credential creation
- generated OIDC secrets for ETX Argo CD and Vault
- ordered creation of the ETX infra and control-plane root Argo CD Applications
- postflight checks for ETX Applications, Keycloak realm import, and OIDC discovery

```bash
# Install required collections
ansible-galaxy collection install -r ansible/requirements.yml

# Run the bootstrap playbook
ansible-playbook ansible/bootstrap.yml
```

Runtime overrides mirror the old `bootstrap-etx-workshop.sh` flags:

```bash
GITHUB_TOKEN=$(gh auth token) ansible-playbook ansible/bootstrap.yml \
  -e etx_git_revision=main \
  -e etx_quay_enabled=false \
  -e etx_multicluster_enabled=false
```

`etx_manage_gitops_operator` defaults to `false` because the Red Hat Demo
Platform framework already provides the base OpenShift GitOps capability. Set it
to `true` only for clusters where that base is absent.

For Red Hat Demo Platform catalog item integration, see
`docs/catalog-item-integration.md`.

## example-app

The `example-app` directory contains a minimal OpenShift application (Red Hat UBI httpd) with all the resources needed to deploy and expose it. Use it as a starting point — replace the image, add environment variables, and extend the manifests as you build out the lab exercises.

## Promotion to Runtime

When the CI pipeline is ready to promote a build to the **Runtime** cluster, it commits updated manifests to the companion repo:
[service-enablement-app-platform-runtime-gitops](https://github.com/rhpds/service-enablement-app-platform-runtime-gitops)
