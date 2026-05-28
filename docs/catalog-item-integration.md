# Catalog item integration

## Purpose

The Red Hat Demo Platform catalog item should use this repository as the GitOps
payload and run `ansible/bootstrap.yml` as the bootstrap entrypoint. This replaces
the previous manual execution of:

```text
etx_app_platform_bootstrap/bootstrap-etx-workshop.sh
```

The catalog item remains responsible for provisioning the cluster and providing a
Linux Ansible control environment. The playbook in this repository is responsible
for wiring ETX-specific bootstrap state into the already-provisioned factory
cluster.

## Required catalog behavior

1. Provision the factory cluster with the Red Hat Demo Platform base GitOps
   capability.
2. Ensure `oc` is logged into the target cluster with privileges to create
   Argo CD Applications, Secrets, Operators, Namespaces, RBAC, and routes.
3. Install Ansible collection dependencies:

   ```bash
   ansible-galaxy collection install -r ansible/requirements.yml
   ```

4. Run the bootstrap playbook:

   ```bash
   ansible-playbook ansible/bootstrap.yml \
     -e etx_git_revision=<branch-or-main>
   ```

5. Pass private repository or seed credentials only as runtime variables or
   environment variables. Do not commit generated credentials.

## Catalog variables

| Catalog input | Ansible variable | Default | Notes |
| --- | --- | --- | --- |
| GitOps repository URL | `etx_git_repo_url` | `https://github.com/rhpds/service-enablement-app-platform-factory-gitops.git` | Repo watched by OpenShift GitOps. |
| Git revision | `etx_git_revision` | `main` | Use the PR branch for validation, `main` after merge. |
| Git token | `etx_git_token` or `GITHUB_TOKEN` | empty | Required for private repo access and GitLab seed flows. |
| Cluster domain | `etx_cluster_domain` | auto-discovered | Override only when discovery is unavailable. |
| Cluster API URL | `etx_cluster_api_url` | auto-discovered | Uses `oc whoami --show-server`. |
| Cluster GUID | `etx_cluster_guid` | derived from domain | Used in provision metadata. |
| Deploy GitLab | `etx_gitlab_enabled` | `true` | Migrated from `--disable-gitlab`. |
| Deploy Quay | `etx_quay_enabled` | `true` | Migrated from `--disable-quay`. |
| Deploy Vault | `etx_vault_enabled` | `true` | Migrated from `--disable-vault`. |
| Deploy Web Terminal | `etx_web_terminal_enabled` | `true` | Migrated from `--disable-web-terminal`. |
| Deploy multicluster namespace setup | `etx_multicluster_enabled` | `true` | Migrated from `--disable-multicluster`. |
| Install OpenShift GitOps operator | `etx_manage_gitops_operator` | `false` | Keep false when RHPDS already provides base GitOps. |
| Patch base Argo CD health/RBAC | `etx_patch_base_argocd` | `true` | Migrates the imperative Argo CD patch from the old shell bootstrap. |
| Validate cluster SSO isolation | `etx_validate_cluster_sso_isolation` | `true` | Ensures ETX Keycloak does not break existing cluster SSO. |
| Validate ETX workshop requirements | `etx_validate_workshop_requirements` | `true` | Waits for Applications, Keycloak realm, and OIDC discovery. |

## Bootstrap flow

The catalog item should expect this ordering:

1. `etx_bootstrap_preflight`
   - Discovers cluster facts.
   - Validates repository access.
   - Records the cluster default SSO baseline.
   - Generates runtime OIDC secrets when not provided.
2. `etx_gitops_bootstrap`
   - Waits for the base `openshift-gitops` Argo CD instance.
   - Applies ETX Argo CD RBAC and health checks.
   - Creates private repository credentials when a token is provided.
   - Creates the `etx-infra` root Application.
   - Waits for infra reconciliation before creating `etx-control-plane`.
3. `etx_postflight_validation`
   - Waits for root and child Applications.
   - Waits for Keycloak realm import.
   - Verifies ETX OIDC discovery.
   - Checks cluster default SSO isolation.
   - Prints user-facing endpoints.

## GitOps layers

The playbook creates two root Applications in `openshift-gitops`:

| Application | Path | Purpose |
| --- | --- | --- |
| `etx-infra` | `etx-infra-app/migrated-source/bootstrap` | ETX GitOps admin plane, AppProjects, infra operators, provision metadata. |
| `etx-control-plane` | `etx-control-plane-app/migrated-source/bootstrap` | Keycloak realm, GitLab, Quay, Vault, seed jobs, and multicluster setup. |

`etx-control-plane` creates child Applications in the dedicated `etx-gitops`
instance after infra has created that ETX GitOps plane.

## Validation commands

Run these before opening or updating the PR:

```bash
ansible-playbook --syntax-check ansible/bootstrap.yml
kustomize build etx-infra-app/overlays/factory
kustomize build etx-control-plane-app/overlays/factory
helm template etx-infra etx-infra-app/migrated-source/bootstrap \
  --set platform.enabled=false \
  --set etxGitops.enabled=true \
  --set keycloakOperator.enabled=true \
  --set quayOperator.enabled=true \
  --set vaultSecretsOperator.enabled=true \
  --set webTerminalOperator.enabled=true
helm template etx-control-plane etx-control-plane-app/migrated-source/bootstrap \
  --set keycloakEtxLab.enabled=true \
  --set gitlabEtxLab.enabled=true \
  --set quayEtxLab.enabled=true \
  --set vaultEtxLab.enabled=true \
  --set multiClusterSetup.enabled=true
```

## Notes for first catalog item test

- Use the migration branch as `etx_git_revision` until the PR is merged.
- Keep `etx_manage_gitops_operator=false` unless the catalog item does not
  install the base GitOps operator.
- If the repo is private, pass `GITHUB_TOKEN` or `-e etx_git_token=...`.
- Expect Quay to take longer than the other components on small clusters.
- If postflight fails on cluster default SSO isolation, stop and inspect the
  cluster SSO route before retrying.
