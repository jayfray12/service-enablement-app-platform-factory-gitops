# Service Enablement: App Platform Software Factory — GitOps

This repository contains the GitOps configuration for the **Software Factory** cluster.
ArgoCD on the factory cluster watches this repo and deploys the resources declared here.

## Structure

```
example-app/   Kubernetes manifests for the example application
ansible/       Ansible playbook for tasks that require imperative configuration
```
