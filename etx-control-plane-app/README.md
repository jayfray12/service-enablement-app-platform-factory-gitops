# ETX control plane app

This path is reserved for ETX workshop platform services reconciled by Red Hat
OpenShift GitOps after the factory GitOps base and ETX infra deltas are ready.

Expected content:

- ETX Keycloak/RHBK instance and `etx-lab` realm.
- GitLab CE with OIDC integration.
- GitLab repository seeding automation.
- Red Hat Quay registry with OIDC integration.
- Vault deployment and auth configuration.
- Multi-cluster namespace setup, if retained.

Imperative validation and secret preparation should stay in `ansible/`; desired
cluster state should live here as declarative GitOps content.
