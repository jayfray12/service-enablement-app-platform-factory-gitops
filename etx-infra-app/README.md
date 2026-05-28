# ETX infra app

This path is reserved for ETX-specific infrastructure deltas on top of the
Red Hat Demo Platform GitOps base.

Expected content:

- Argo CD project/RBAC/health-check deltas required by ETX.
- Optional dedicated `etx-gitops` instance if the final design keeps it.
- Operators not already owned by the RHPDS base.
- Namespaces and minimal RBAC required before the ETX control plane syncs.

Do not duplicate the base OpenShift GitOps installation unless the RHPDS catalog
item being used does not provide it.
