# UXE ERPNext DEV Platform

Technical README for the UXE ERPNext v16 DEV platform deployed on Azure
and K3s.

This repository architecture separates **application source/CI** from
**deployment state/CD**:

``` text
Developer / ERPNext upstream
        |
        v
GitHub: erpnext-app (dev)
        |
        | GitHub Actions
        v
Self-hosted runner: vm-uxe-hub-cicd-01
        |
        +--> Build ERPNext/Frappe image
        +--> Validate with bench version
        +--> Push immutable SHA image to ACR
        |
        v
GitHub: erpnext-gitops (dev)
        |
        +-------------------------------+
        |                               |
        v                               v
Argo CD: erpnext-dev           Argo CD: erpnext-dev-infra
        |                               |
        v                               v
Application/runtime             Persistent/base infrastructure
        |                               |
        +---------------+---------------+
                        |
                        v
                 DEV K3s cluster
                        |
                        v
Azure Application Gateway -> ingress-nginx -> ERPNext
```

## Platform Summary

  -----------------------------------------------------------------------------------
  Component                  DEV implementation
  -------------------------- --------------------------------------------------------
  ERPNext site               `dev-erpnext.uxe.ai`

  ERPNext                    `16.32.3`

  Frappe                     `16.31.0`

  K3s node                   `vm-uxe-dev-erpnext-01`

  K3s node IP                `10.31.1.16`

  CI/CD VM                   `vm-uxe-hub-cicd-01`

  Container registry         `cruxeplatformuaen01.azurecr.io`

  ERPNext image              `cruxeplatformuaen01.azurecr.io/erpnext/dev:<GIT_SHA>`

  Azure Key Vault            `kv-uxe-dev-uaen`

  Kubernetes namespace       `erpnext`

  Application delivery       GitHub Actions + ACR + GitOps + Argo CD

  North-south traffic        Azure Application Gateway -\> ingress-nginx -\> ERPNext

  ERPNext sites storage      Local PV on `/data-disk/erpnext-sites`

  MariaDB storage            Local PV on `/db-disk/mariadb`

  NFS                        Removed; not used by current ERPNext PVCs
  -----------------------------------------------------------------------------------

## Repositories

### `erpnext-app`

Application source and CI repository:

``` text
uxe-security-solutions/erpnext-app
branch: dev
```

Responsibilities:

-   ERPNext application source/customizations.
-   Tracking and merging ERPNext v16 upstream releases.
-   GitHub Actions workflow.
-   Building the custom ERPNext/Frappe image.
-   Validating the image.
-   Pushing the immutable image to ACR.
-   Updating the GitOps repository with the new image SHA.

DEV workflow:

``` text
.github/workflows/build-dev.yml
```

**Argo CD does not directly monitor `erpnext-app`.**

A source-code push first triggers GitHub Actions according to the
branch/path filters configured in `build-dev.yml`.

### `erpnext-gitops`

Deployment source-of-truth repository:

``` text
uxe-security-solutions/erpnext-gitops
branch: dev
```

Local path on the CI/CD VM:

``` bash
~/git/erpnext-gitops
```

Argo CD monitors this repository.

## CI/CD Flow for Source-Code Changes

A normal application change follows this path:

``` text
1. Developer pushes source change to erpnext-app/dev
2. GitHub Actions build-dev.yml starts when its event filters match
3. Self-hosted runner builds the ERPNext image
4. CI validates the image with bench version
5. CI pushes:
   cruxeplatformuaen01.azurecr.io/erpnext/dev:<GIT_SHA>
6. CI updates:
   erpnext-gitops/environments/dev/values.yaml
7. CI commits/pushes the GitOps change to dev
8. Argo CD detects the new GitOps desired state
9. erpnext-dev performs the application deployment
10. K3s converges to the new image
```

The important handoff is:

``` text
erpnext-app
    |
    | CI
    v
ACR
    |
    | update image.tag
    v
erpnext-gitops
    |
    | CD
    v
Argo CD
    |
    v
K3s
```

A push to `erpnext-app` is therefore a **CI event**, not a direct Argo
CD event.

## GitHub Actions

The DEV build workflow is:

``` text
erpnext-app/.github/workflows/build-dev.yml
```

The exact workflow YAML is authoritative for its `push`, `branches`,
`paths`, and `paths-ignore` filters.

The workflow runs on the self-hosted runner hosted by:

``` text
vm-uxe-hub-cicd-01
```

Runner location:

``` bash
/data/github-runners/erpnext-app
```

Common runner operations:

``` bash
cd /data/github-runners/erpnext-app

sudo ./svc.sh stop
sudo ./svc.sh start
sudo ./svc.sh status
```

Validate Docker access:

``` bash
sudo -u azureuser docker ps
ps -eo user,group,comm | grep Runner.Listener
```

### Image Build

The build follows the Frappe layered image design. The image is tagged
with the source Git commit SHA.

Example:

``` bash
docker build \
  --no-cache \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --secret=id=apps_json,src=apps.json \
  --tag=cruxeplatformuaen01.azurecr.io/erpnext/dev:<GIT_SHA> \
  --file=images/layered/Containerfile .
```

Validate before promotion:

``` bash
docker run --rm "${IMAGE}" bench version
docker push "${IMAGE}"
```

### Azure Authentication

The CI/CD VM uses Azure Managed Identity:

``` bash
az login --identity --output none
az acr login --name cruxeplatformuaen01
```

No Azure service-principal password is required for this runner flow.

## Argo CD Applications

The DEV platform has **two different Argo CD Applications**.

### `erpnext-dev`

Purpose: ERPNext application release and runtime resources.

``` text
Application: erpnext-dev
Project:     default
Repository:  git@github.com:uxe-security-solutions/erpnext-gitops.git
Branch:      dev
Path:        erpnext
Values:      ../environments/dev/values.yaml
Destination: https://10.31.1.16:6443
Namespace:   erpnext
Sync:        Automated + Prune
```

Typical ownership:

-   ERPNext Deployments.
-   Services.
-   Ingress.
-   Valkey.
-   Maintenance page and maintenance resources.
-   Migration hook.
-   Maintenance RBAC.
-   DEV application image tag.

Important trigger paths:

``` text
erpnext/templates/*.yaml
environments/dev/values.yaml
```

A source-code build eventually triggers this application **indirectly**
when GitHub Actions changes `environments/dev/values.yaml`.

### `erpnext-dev-infra`

Purpose: persistent/base infrastructure required by ERPNext.

``` text
Application: erpnext-dev-infra
Repository:  git@github.com:uxe-security-solutions/erpnext-gitops.git
Branch:      dev
Path:        environments/dev/manifests
Destination: https://10.31.1.16:6443
Namespace:   erpnext
```

Typical ownership:

-   MariaDB infrastructure.
-   MariaDB PV/PVC.
-   ERPNext sites PV/PVC.
-   Storage definitions.
-   Key Vault / Secrets Store CSI integration.

Important files include:

``` text
environments/dev/manifests/mariadb.yaml
environments/dev/manifests/mariadb-storage.yaml
environments/dev/manifests/erpnext-storage.yaml
environments/dev/manifests/erpnext-keyvault.yaml
```

### Argo CD Ownership Matrix

  ---------------------------------------------------------------------------------------
  Change                                          First automation  Argo CD application
  ----------------------------------------------- ----------------- ---------------------
  `erpnext-app` source                            GitHub Actions    `erpnext-dev`
                                                                    indirectly after
                                                                    GitOps update

  `erpnext-app/.github/workflows/build-dev.yml`   GitHub Actions    None directly
                                                  configuration     

  `environments/dev/values.yaml`                  Argo CD           `erpnext-dev`
                                                  reconciliation    

  `erpnext/templates/*.yaml`                      Argo CD           `erpnext-dev`
                                                  reconciliation    

  `environments/dev/manifests/*`                  Argo CD           `erpnext-dev-infra`
                                                  reconciliation    
  ---------------------------------------------------------------------------------------

## GitOps Image Update

After a successful build, CI updates:

``` text
erpnext-gitops/environments/dev/values.yaml
```

Example:

``` yaml
image:
  repository: cruxeplatformuaen01.azurecr.io/erpnext/dev
  tag: <NEW_GIT_SHA>
  pullPolicy: IfNotPresent
```

Once this commit reaches `erpnext-gitops/dev`, `erpnext-dev` sees a new
desired state and automated reconciliation can begin.

## Maintenance and Migration Deployment Sequence

The application deployment keeps users on the maintenance page while the
release is being applied. The backup is taken before the application
rollout, while database migration and cache clearing run after the new
ERPNext image has been reconciled.

``` text
PreSync -20
    |
    v
erpnext-maintenance-enable
    |
    +--> Route dev-erpnext ingress to maintenance service
    |
    v
PreSync -10
    |
    v
frappe-bench-erpnext-backup
    |
    +--> bench --site "${SITE}" backup --with-files
    |
    v
Sync 0
    |
    v
Reconcile ERPNext workloads / new image
    |
    +--> Deploy the new image from ACR
    +--> Reconcile ERPNext runtime resources
    |
    v
Sync 10
    |
    v
frappe-bench-erpnext-migrate
    |
    +--> bench --site "${SITE}" migrate
    +--> bench --site "${SITE}" clear-cache
    +--> bench --site "${SITE}" clear-website-cache
    |
    v
Sync 20
    |
    +--> Restore Git-managed ERPNext ingress backend
    +--> frappe-bench-erpnext:8080
    |
    v
Maintenance OFF
    |
    v
ERPNext available
```

The resulting release order is:

``` text
Maintenance ON -> Backup -> New image rollout -> Migrate -> Clear caches -> Restore ingress -> ERPNext available
```

The maintenance page is **deployment/health gated**, not based on a
fixed countdown. It remains available during the release process instead
of assuming maintenance will finish within a fixed number of minutes.

``` text
Scheduled Maintenance
ERPNext is currently being updated.
Maintenance in progress...
```

The old `00:00` countdown is not required.

### Backup Job

Template:

``` text
erpnext/templates/job-dev-backup.yaml
```

Argo CD ordering:

``` yaml
argocd.argoproj.io/hook: PreSync
argocd.argoproj.io/sync-wave: "-10"
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

Core command:

``` bash
bench --site "${SITE}" backup --with-files
```

This job runs after maintenance is enabled and before the new
application image is rolled out. A backup failure stops the release
before migration proceeds.

### Migration Job

Template:

``` text
erpnext/templates/job-dev-migrate.yaml
```

Argo CD ordering:

``` yaml
argocd.argoproj.io/hook: Sync
argocd.argoproj.io/sync-wave: "10"
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

Core commands:

``` bash
bench --site "${SITE}" migrate
bench --site "${SITE}" clear-cache
bench --site "${SITE}" clear-website-cache
```

The migration Job uses the image configured by
`environments/dev/values.yaml`, so when CI updates the immutable image
SHA the migration container is rendered from that same desired
application image.

Any command failure stops the hook because the script uses strict shell
error handling. The Git-managed ERPNext ingress is restored at Sync wave
`20`, after the migration wave has completed successfully.

### Deployment Wave Summary

  -----------------------------------------------------------------------------------------------------
  Phase                             Wave Resource / action                Purpose
  ---------------- --------------------- -------------------------------- -----------------------------
  PreSync                          `-20` `erpnext-maintenance-enable`     Route end users to the
                                                                          maintenance service

  PreSync                          `-10` `frappe-bench-erpnext-backup`    Create database,
                                                                          configuration, public-file
                                                                          and private-file backup

  Sync                               `0` ERPNext workloads                Reconcile/deploy the new
                                                                          ERPNext image

  Sync                              `10` `frappe-bench-erpnext-migrate`   Run migration and clear
                                                                          ERPNext caches using the
                                                                          configured image

  Sync                              `20` `dev-erpnext` Ingress            Restore the normal
                                                                          `frappe-bench-erpnext:8080`
                                                                          backend
  -----------------------------------------------------------------------------------------------------

Operational note: during a controlled release, verify that the wave-0
ERPNext workloads can reach the health state required for Argo CD to
continue to Sync wave `10`. If a future ERPNext release requires
database migration before the new workload can become healthy, the
ordering/health gating must be reviewed before forcing the sync forward.

## Maintenance Page

Relevant GitOps resources:

``` text
erpnext/templates/job-maintenance-enable.yaml
erpnext/templates/rbac-maintenance.yaml
erpnext/templates/configmap-maintenance-page.yaml
erpnext/templates/configmap-maintenance-state.yaml
erpnext/templates/deployment-maintenance.yaml
erpnext/templates/service-maintenance.yaml
```

The maintenance NGINX Deployment mounts the page from a ConfigMap.
Because the page uses a ConfigMap `subPath` mount, page changes require
the maintenance pod to roll. The Deployment should use a Helm checksum
annotation for the page ConfigMap so GitOps page updates automatically
produce a new pod-template hash.

## Backup

Every migration creates a full site backup with:

``` bash
bench --site dev-erpnext.uxe.ai backup --with-files
```

Inside the container:

``` text
/home/frappe/frappe-bench/sites/dev-erpnext.uxe.ai/private/backups
```

On the ERPNext VM:

``` text
/data-disk/erpnext-sites/dev-erpnext.uxe.ai/private/backups
```

Each backup contains:

  Component            Suffix
  -------------------- ----------------------------
  Site configuration   `-site_config_backup.json`
  Database             `-database.sql.gz`
  Public files         `-files.tar`
  Private files        `-private-files.tar`

List backups from the host:

``` bash
ls -lh /data-disk/erpnext-sites/dev-erpnext.uxe.ai/private/backups
```

## Storage

The current DEV environment uses local persistent volumes.

### MariaDB

``` text
PVC:          erpnext-mariadb
Capacity:     50Gi
Access mode:  RWO
StorageClass: mariadb-local
Host path:    /db-disk/mariadb
Reclaim:      Retain
```

### ERPNext Sites

``` text
PVC:          erpnext-sites
Capacity:     100Gi
Access mode:  RWO
StorageClass: erpnext-local
Host path:    /data-disk/erpnext-sites
Reclaim:      Retain
```

### NFS

The previously deployed NFS external provisioner was removed after
verification showed that no ERPNext PVC/PV used it.

The active design does **not** depend on NFS or RWX storage.

## ACR Authentication

K3s/containerd uses authenticated access to:

``` text
cruxeplatformuaen01.azurecr.io
```

K3s registry configuration:

``` text
/etc/rancher/k3s/registries.yaml
```

Conceptual configuration:

``` yaml
mirrors:
  "cruxeplatformuaen01.azurecr.io":
    endpoint:
      - "https://cruxeplatformuaen01.azurecr.io"

configs:
  "cruxeplatformuaen01.azurecr.io":
    auth:
      username: "k3s-erpnext-dev"
      password: "<retrieve securely from Key Vault>"
```

Never commit the registry password to Git.

## ACR Credential Expiry

The ACR credential is stored in:

``` text
Azure Key Vault: kv-uxe-dev-uaen
```

Relevant secrets include:

``` text
acr-k3s-erpnext-dev-password
acr-k3s-erpnext-dev-active-slot
```

The password secret has an expiry and a systemd-driven check on the
CI/CD VM monitors the remaining lifetime.

Credential rotation must update all consumers before the previous
credential/slot is invalidated. Otherwise new K3s image pulls can fail
even though already-running containers continue working.

## K3s Runtime Validation

Check workloads:

``` bash
kubectl get pods -n erpnext

kubectl get deployments -n erpnext \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image'
```

Check versions:

``` bash
kubectl exec -n erpnext \
  deployment/frappe-bench-erpnext-gunicorn \
  -- bench version
```

Expected validated versions:

``` text
erpnext 16.32.3
frappe 16.31.0
```

Check installed site applications:

``` bash
kubectl exec -n erpnext \
  deployment/frappe-bench-erpnext-gunicorn \
  -- bench --site dev-erpnext.uxe.ai list-apps
```

## Argo CD Operations

Check the application source, revision, sync and health:

``` bash
argocd app get erpnext-dev --hard-refresh
argocd app get erpnext-dev-infra --hard-refresh
```

Inspect live source configuration:

``` bash
argocd app get erpnext-dev -o json | jq '.spec.source, .spec.sources'
argocd app get erpnext-dev-infra -o json | jq '.spec.source, .spec.sources'
```

Watch application state:

``` bash
argocd app get erpnext-dev --watch
```

Check the current ERPNext ingress backend:

``` bash
KUBECONFIG=~/erpnext-dev-kubeconfig.yaml \
kubectl get ingress dev-erpnext \
  -n erpnext \
  -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}:{.spec.rules[0].http.paths[0].backend.service.port.number}{"\n"}'
```

Normal backend:

``` text
frappe-bench-erpnext:8080
```

During maintenance:

``` text
erpnext-maintenance:80
```

Check migration:

``` bash
KUBECONFIG=~/erpnext-dev-kubeconfig.yaml \
kubectl get job frappe-bench-erpnext-migrate -n erpnext

KUBECONFIG=~/erpnext-dev-kubeconfig.yaml \
kubectl logs -n erpnext job/frappe-bench-erpnext-migrate
```

## Updating ERPNext from Upstream

From the application repository:

``` bash
cd ~/git/erpnext-app

git fetch upstream --tags

git show upstream/version-16:erpnext/__init__.py \
  | grep '__version__'

git log --oneline \
  dev..upstream/version-16 \
  | head -30
```

Before merging, create a backup branch:

``` bash
git branch backup/dev-before-<VERSION>
```

Merge:

``` bash
git merge upstream/version-16
```

Validate:

``` bash
grep '__version__' erpnext/__init__.py
git status
git log -8 --oneline
```

Push only after reviewing the merge:

``` bash
git push origin dev
```

The push then follows the normal GitHub Actions -\> ACR -\> GitOps -\>
Argo CD path.

## Important Paths

  -----------------------------------------------------------------------------------------
  Purpose                   Path / name
  ------------------------- ---------------------------------------------------------------
  ERPNext app repository    `~/git/erpnext-app`

  GitOps repository         `~/git/erpnext-gitops`

  GitHub runner             `/data/github-runners/erpnext-app`

  DEV kubeconfig from CI/CD `~/erpnext-dev-kubeconfig.yaml`
  VM                        

  ERPNext host backups      `/data-disk/erpnext-sites/dev-erpnext.uxe.ai/private/backups`

  MariaDB host data         `/db-disk/mariadb`

  ERPNext sites host data   `/data-disk/erpnext-sites`

  K3s registry              `/etc/rancher/k3s/registries.yaml`
  configuration             

  GitOps SSH key            `~/.ssh/github_cicd`

  ACR                       `cruxeplatformuaen01.azurecr.io`

  Key Vault                 `kv-uxe-dev-uaen`
  -----------------------------------------------------------------------------------------

## Troubleshooting

  ---------------------------------------------------------------------
  Symptom                            Primary checks
  ---------------------------------- ----------------------------------
  `ImagePullBackOff` / ACR `401`     `registries.yaml`, ACR token, Key
                                     Vault rotation state, K3s restart

  GitHub Actions cannot access       `id azureuser`, Docker socket
  Docker                             ownership, runner service restart

  Argo CD shows old hook             `git show origin/dev`, Argo
                                     manifests, hard refresh

  UI broken after upgrade            Migration cache clear, workload
                                     rollout, browser cache

  Migration Job missing              Argo sync state, hook annotations,
                                     `BeforeHookCreation`

  Migration Job failed               Job logs, pod events, DB
                                     connectivity, backup result

  Wrong ERPNext version              Deployment image SHA,
                                     `bench version`, CI build logs

  PVC unavailable                    PV/PVC state, host disk mount and
                                     permissions
  ---------------------------------------------------------------------

## Security Notes

-   Do not commit passwords, registry credentials, Key Vault secret
    values, kubeconfig credentials, or private SSH keys.
-   Use Azure Managed Identity for Azure access where supported.
-   Store ACR pull credentials in Key Vault and monitor their expiry.
-   Keep container images immutable by using Git SHA tags.
-   Use Git as the deployment source of truth instead of making
    unmanaged live-cluster changes.
-   Review persistent-storage changes carefully because local PVs use
    `Retain` and contain stateful application/database data.

## References

-   Argo CD automated sync:
    https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
-   Argo CD application specification:
    https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/
-   Argo CD sync phases and waves:
    https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
-   Argo CD Helm integration:
    https://argo-cd.readthedocs.io/en/latest/user-guide/helm/
-   Argo CD CI automation:
    https://argo-cd.readthedocs.io/en/latest/user-guide/ci_automation/
-   GitHub Actions workflow syntax:
    https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
-   K3s private registry configuration:
    https://docs.k3s.io/installation/private-registry
-   Kubernetes ConfigMaps:
    https://kubernetes.io/docs/concepts/configuration/configmap/
-   Helm chart tips and tricks:
    https://helm.sh/docs/howto/charts_tips_and_tricks/
-   Frappe Bench backup:
    https://docs.frappe.io/framework/user/en/bench/reference/backup
-   Frappe Bench migrate:
    https://docs.frappe.io/framework/user/en/bench/reference/migrate

------------------------------------------------------------------------

This README is derived from the UXE ERPNext DEV technical/as-built
documentation and reflects the validated DEV implementation state
documented for August 2026.
