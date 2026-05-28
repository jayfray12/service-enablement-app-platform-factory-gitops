# ETX migration analysis

## Objective

Migrate the ETX bootstrap model from `etx_app_platform_bootstrap` into the Red Hat
Demo Platform GitOps framework used by `service-enablement-app-platform-factory-gitops`.

The target repository must keep the existing RHPDS contract:

- Imperative bootstrap logic lives in `ansible/`.
- Declarative cluster state is reconciled by Red Hat OpenShift GitOps.
- The existing factory GitOps base is treated as the infra baseline.
- ETX-specific platform services are layered on top as a control plane.

## Source baseline

The supported source entrypoint is:

```text
etx_app_platform_bootstrap/bootstrap-etx-workshop.sh
```

The source script performs these main phases:

1. Parse runtime parameters and feature flags.
2. Validate local tools, OpenShift login, cluster domain, API URL, and repository access.
3. Generate or receive runtime secrets for Argo CD and Vault OIDC.
4. Install or reconcile the OpenShift GitOps operator.
5. Wait for the default `openshift-gitops` Argo CD instance.
6. Patch default Argo CD RBAC and health checks.
7. Create optional Git repository credentials for private repository access.
8. Apply one root Argo CD `Application`: `etx-bootstrap-infra`.
9. Wait for Argo CD sync and component health.
10. Run post-deploy validation for Keycloak, GitLab, Quay, Vault, Web Terminal, and SSO guardrails.
11. Run optional Web Terminal post-configuration.
12. Print deployment summary and access details.

The supported source GitOps path is:

```text
bootstrap/
cluster/infra/bootstrap/
cluster/platform/bootstrap/
workloads_library/infra/
workloads_library/platform/
```

The older source path:

```text
gitops/infrastructure/standalone/
gitops/applications/standalone/
```

should be treated as reference material only unless a resource is missing from the supported
`cluster/` and `workloads_library/` model.

## Target baseline

The current target repository has:

```text
ansible/bootstrap.yml
ansible/requirements.yml
example-app/
```

The README defines `ansible/bootstrap.yml` as the location for imperative setup
that cannot be expressed declaratively, and ArgoCD as the reconciler for manifests
committed to the repository.

The target repository does not currently contain `etx-infra-app/` or
`etx-control-plane-app/`. These should become the two ETX migration boundaries.

## Recommended target structure

```text
ansible/
  bootstrap.yml
  requirements.yml
  group_vars/
    all.yml
  roles/
    etx_bootstrap_preflight/
    etx_gitops_bootstrap/
    etx_postflight_validation/

etx-infra-app/
  base/
  overlays/
    factory/
  applications/

etx-control-plane-app/
  base/
  overlays/
    factory/
  applications/

docs/
  etx-migration-analysis.md
```

Keep `example-app/` as sample application content unless the lab flow explicitly
replaces it.

## Imperative logic migration to Ansible

Move these behaviors from `bootstrap-etx-workshop.sh` to Ansible:

| Source behavior | Target Ansible location | Notes |
| --- | --- | --- |
| Argument parsing and defaults | `group_vars/all.yml` plus `-e` overrides | Feature flags become variables such as `etx_gitlab_enabled`. |
| CLI and login checks | `roles/etx_bootstrap_preflight` | Use `kubernetes.core.k8s_info` and `ansible.builtin.uri`; avoid shell except when no module exists. |
| Cluster domain/API/GUID discovery | `roles/etx_bootstrap_preflight` | Register facts consumed by Argo CD Application templates. |
| OIDC secret generation | `roles/etx_bootstrap_preflight` | Use `ansible.builtin.password` lookup or generated facts; avoid committing generated secrets. |
| OpenShift GitOps operator install | Prefer RHPDS base GitOps; fallback in `roles/etx_gitops_bootstrap` | Since target framework already carries base GitOps, do not duplicate unless the base is absent. |
| Default Argo CD patching | `roles/etx_gitops_bootstrap` or declarative infra overlay | If RHPDS owns default Argo CD, patch only the deltas needed for ETX health checks/RBAC. |
| Git repository credential secret | `roles/etx_gitops_bootstrap` | Use `kubernetes.core.k8s` with `no_log: true`. |
| Root Application creation | `roles/etx_gitops_bootstrap` | Apply only the root `Application` objects pointing at target repo paths. |
| Wait for sync/health | `roles/etx_postflight_validation` | Use `kubernetes.core.k8s_info` retries against `Application` status. |
| Keycloak/GitLab/Quay/Vault validation | `roles/etx_postflight_validation` | Keep as checks, not desired state. |
| Summary rendering | `roles/etx_postflight_validation` | Optional Ansible debug/template output. |
| Web Terminal post-config | Ansible role if still required | First confirm whether RHPDS base already configures it. |

The Ansible playbook should not embed large heredoc manifests. Store Kubernetes
resources as templates or files and apply them with `kubernetes.core.k8s`.

## GitOps migration boundaries

### `etx-infra-app/`

This layer should contain ETX deltas that belong below the control plane. Because
the target framework already includes a base GitOps capability equivalent to the
source infra bootstrap, this layer should be deliberately small.

Recommended contents:

- ETX-specific Argo CD projects if not already provided by the RHPDS base.
- ETX-specific Argo CD health checks and RBAC deltas.
- ETX GitOps admin plane only if the design still requires a dedicated
  `etx-gitops` instance separate from `openshift-gitops`.
- Operators not already provided by the RHPDS base:
  - Red Hat Build of Keycloak operator.
  - Red Hat Quay operator.
  - Vault Secrets Operator.
  - Web Terminal Operator only if not base-owned.
- Namespace and minimal RBAC needed by the ETX control-plane layer.

Source candidates:

```text
cluster/infra/bootstrap/
workloads_library/infra/
bootstrap/argocd-healthchecks-patch.yaml
```

Do not blindly migrate the source `gitopsOperator` workload if RHPDS already owns
OpenShift GitOps installation.

### `etx-control-plane-app/`

This layer should contain the ETX platform services deployed after the GitOps base
and infra deltas are available.

Recommended contents:

- ETX Keycloak/RHBK instance and `etx-lab` realm.
- GitLab CE deployment and OIDC integration.
- GitLab repository seeding automation.
- Quay registry and OIDC integration.
- Vault deployment, OIDC auth, and Kubernetes auth.
- Multi-cluster namespace setup if still part of the workshop.
- App-of-apps definitions or kustomize/helm composition for the above services.

Source candidates:

```text
cluster/platform/bootstrap/
workloads_library/platform/
```

## GitOps composition recommendation

Use a two-root model from Ansible:

```text
etx-infra
  -> path: etx-infra-app

etx-control-plane
  -> path: etx-control-plane-app
```

`etx-control-plane` should depend on `etx-infra` through Argo CD sync waves and
health checks, not through Ansible ordering beyond creating the root Applications.

If the target RHPDS framework already creates a root Application for the repo, an
alternative is to make `etx-infra-app/` and `etx-control-plane-app/` child paths
under that existing root instead of creating new roots. This should be validated
against the deployed RHPDS catalog item before implementation.

## Variable mapping

| Source variable/flag | Target Ansible variable | GitOps consumer |
| --- | --- | --- |
| `GIT_REVISION` / `--git-revision` | `etx_git_revision` | Root Applications `targetRevision` |
| `GITHUB_TOKEN` / `--github-token` | `etx_git_token` | Repository credential Secret, GitLab seed |
| `ARGOCD_OIDC_CLIENT_SECRET` | `etx_argocd_oidc_client_secret` | ETX GitOps or Argo CD OIDC config |
| `VAULT_OIDC_CLIENT_SECRET` | `etx_vault_oidc_client_secret` | Vault OIDC auth config |
| `--disable-gitlab` | `etx_gitlab_enabled: false` | Control-plane GitLab Application |
| `--disable-quay` | `etx_quay_enabled: false` | Infra Quay operator and control-plane Quay |
| `--disable-vault` | `etx_vault_enabled: false` | Infra Vault operator and control-plane Vault |
| `--disable-web-terminal` | `etx_web_terminal_enabled: false` | Infra Web Terminal operator |
| `--disable-multicluster` | `etx_multicluster_enabled: false` | Control-plane multi-cluster setup |

## Migration phases

1. Establish the target branch and documentation.
2. Convert the shell bootstrap phases into Ansible roles with idempotent module usage.
3. Add `etx-infra-app/` with only the deltas not provided by the RHPDS GitOps base.
4. Add `etx-control-plane-app/` with the ETX platform workloads.
5. Replace source repo URLs in migrated manifests with the target repository URL.
6. Replace source paths with target paths.
7. Convert Helm values that only existed as shell placeholder substitutions into Ansible-rendered `Application` templates or declarative values files.
8. Validate in a clean RHPDS cluster:
   - Ansible preflight succeeds.
   - Root Applications are created.
   - Argo CD syncs infra before control plane.
   - ETX SSO does not regress the cluster default SSO.
   - GitLab, Quay, Vault, Keycloak, and Web Terminal match the source acceptance checks.

## Key risks

- Duplicating the OpenShift GitOps operator or default Argo CD config already owned by RHPDS.
- Carrying private token values into committed manifests.
- Preserving source repository URLs and paths after migration.
- Running ETX platform workloads before the RHPDS GitOps base and required operators are ready.
- Keeping shell validation logic instead of converting it into Ansible facts and retries.
- Treating historical `gitops/standalone` assets as authoritative.

## Initial decision

Use the RHPDS GitOps base as the infra baseline. Migrate only ETX-specific infra
deltas into `etx-infra-app/`, then layer the ETX workshop control plane into
`etx-control-plane-app/`. Use Ansible to perform preflight, secret handling,
root Application creation, and postflight validation.
