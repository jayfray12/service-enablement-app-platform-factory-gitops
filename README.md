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
  bootstrap.yml      Playbook for imperative setup steps (namespace creation, ConfigMaps, RBAC)
  requirements.yml   Ansible collection dependencies (kubernetes.core)
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

### 3. Run the Ansible bootstrap (optional)

The `ansible/bootstrap.yml` playbook handles tasks that cannot be expressed declaratively in GitOps — for example, creating namespaces with specific labels or configuring external integrations.

```bash
# Install required collections
ansible-galaxy collection install -r ansible/requirements.yml

# Run the bootstrap playbook
ansible-playbook ansible/bootstrap.yml
```

## example-app

The `example-app` directory contains a minimal OpenShift application (Red Hat UBI httpd) with all the resources needed to deploy and expose it. Use it as a starting point — replace the image, add environment variables, and extend the manifests as you build out the lab exercises.

## Promotion to Runtime

When the CI pipeline is ready to promote a build to the **Runtime** cluster, it commits updated manifests to the companion repo:
[service-enablement-app-platform-runtime-gitops](https://github.com/rhpds/service-enablement-app-platform-runtime-gitops)
