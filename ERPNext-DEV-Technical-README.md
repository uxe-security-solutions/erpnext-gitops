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

  ----------------------------------------------------------------------------
  Component           DEV implementation
  ------------------- --------------------------------------------------------
  ERPNext site        `dev-erpnext.uxe.ai`

  ERPNext             `16.32.3`

  Frappe              `16.31.0`

  K3s node            `vm-uxe-dev-erpnext-01`

  K3s node IP         `10.31.1.16`

  CI/CD VM            `vm-uxe-hub-cicd-01`

  Container registry  `cruxeplatformuaen01.azurecr.io`

  ERPNext image       `cruxeplatformuaen01.azurecr.io/erpnext/dev:<GIT_SHA>`

  Azure Key Vault     `kv-uxe-dev-uaen`

  Kubernetes          `erpnext`
  namespace           

  Application         GitHub Actions + ACR + GitOps + Argo CD
  delivery            

  North-south traffic Azure Application Gateway -\> ingress-nginx -\> ERPNext

  ERPNext sites       Local PV on `/data-disk/erpnext-sites`
  storage             

  MariaDB storage     Local PV on `/db-disk/mariadb`

  NFS                 Removed; not used by current ERPNext PVCs
  ----------------------------------------------------------------------------

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

  --------------------------------------------------------------------------------------
  Change                                          First automation Argo CD application
  ----------------------------------------------- ---------------- ---------------------
  `erpnext-app` source                            GitHub Actions   `erpnext-dev`
                                                                   indirectly after
                                                                   GitOps update

  `erpnext-app/.github/workflows/build-dev.yml`   GitHub Actions   None directly
                                                  configuration    

  `environments/dev/values.yaml`                  Argo CD          `erpnext-dev`
                                                  reconciliation   

  `erpnext/templates/*.yaml`                      Argo CD          `erpnext-dev`
                                                  reconciliation   

  `environments/dev/manifests/*`                  Argo CD          `erpnext-dev-infra`
                                                  reconciliation   
  --------------------------------------------------------------------------------------

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

  -------------------------------------------------------------------------------------
  Phase             Wave Resource / action                Purpose
  --------- ------------ -------------------------------- -----------------------------
  PreSync          `-20` `erpnext-maintenance-enable`     Route end users to the
                                                          maintenance service

  PreSync          `-10` `frappe-bench-erpnext-backup`    Create database,
                                                          configuration, public-file
                                                          and private-file backup

  Sync               `0` ERPNext workloads                Reconcile/deploy the new
                                                          ERPNext image

  Sync              `10` `frappe-bench-erpnext-migrate`   Run migration and clear
                                                          ERPNext caches using the
                                                          configured image

  Sync              `20` `dev-erpnext` Ingress            Restore the normal
                                                          `frappe-bench-erpnext:8080`
                                                          backend
  -------------------------------------------------------------------------------------

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

## ACR Credential Expiry Monitoring and Rotation

K3s requires authenticated access to Azure Container Registry (ACR) to
pull ERPNext images from:

``` text
cruxeplatformuaen01.azurecr.io
```

The DEV ACR pull credential lifecycle is monitored from the CI/CD VM and
stored in Azure Key Vault.

### Key Vault Configuration

Azure Key Vault:

``` text
kv-uxe-dev-uaen
```

Primary credential secret:

``` text
acr-k3s-erpnext-dev-password
```

Related active-slot state:

``` text
acr-k3s-erpnext-dev-active-slot
```

The password secret stores the managed ACR pull credential. The
active-slot value is used by the credential-rotation design to track the
active ACR token credential slot.

No password or token value should be committed to Git or printed into
shared logs.

### Credential Expiration

The ACR password secret is configured with an explicit expiration date.
The current lifecycle uses a 180-day credential validity period.

Example:

``` bash
EXPIRY=$(date -u -d '+180 days' +'%Y-%m-%dT%H:%M:%SZ')

az keyvault secret set-attributes \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-password \
  --expires "$EXPIRY"
```

Verify expiration:

``` bash
az keyvault secret show \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-password \
  --query 'attributes.expires || attributes.expiresOn' \
  -o tsv
```

Setting a Key Vault expiry attribute does **not** itself rotate the ACR
credential. Rotation is handled by the scripts described below.

### Expiry Check Script

Installed on:

``` text
vm-uxe-hub-cicd-01
```

Script:

``` text
/usr/local/sbin/check-erpnext-acr-token-expiry.sh
```

Verified implementation:

``` bash
#!/bin/bash
set -euo pipefail

KV="kv-uxe-dev-uaen"
SECRET="acr-k3s-erpnext-dev-password"
THRESHOLD_DAYS=30

echo "Checking ERPNext K3s ACR credential..."

timeout 30 az login --identity --output none

EXPIRY=$(timeout 30 az keyvault secret show \
  --vault-name "$KV" \
  --name "$SECRET" \
  --query 'attributes.expires || attributes.expiresOn' \
  -o tsv)

if [ -z "$EXPIRY" ]; then
  echo "ERROR: Key Vault secret expiry is not configured."
  exit 1
fi

EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date -u +%s)

DAYS_LEFT=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))

echo "Expiry: $EXPIRY"
echo "Days remaining: $DAYS_LEFT"

if [ "$DAYS_LEFT" -lt 0 ]; then
  echo "ERROR: ACR credential has expired."
  exit 1
fi

if [ "$DAYS_LEFT" -le "$THRESHOLD_DAYS" ]; then
  echo "Credential is within rotation window."
  /usr/local/sbin/rotate-erpnext-acr-token.sh
else
  echo "No rotation required."
fi
```

### Expiry Check Logic

``` text
Start
  |
  v
Authenticate to Azure using VM Managed Identity
  |
  v
Read Key Vault secret expiry
  |
  +--> Missing expiry -> ERROR / exit 1
  |
  v
Calculate remaining days
  |
  +--> Already expired -> ERROR / exit 1
  |
  +--> More than 30 days -> No rotation required
  |
  `--> 30 days or less -> Run rotate-erpnext-acr-token.sh
```

The configured rotation threshold is:

``` text
30 days
```

Azure CLI operations are wrapped with `timeout 30` to prevent the
systemd job from hanging indefinitely during Azure login or Key Vault
access.

### Managed Identity Authentication

The check does not use a stored Azure username/password or
service-principal secret.

It authenticates using:

``` bash
az login --identity --output none
```

### Rotation Script

When the remaining credential lifetime is 30 days or less, the
expiry-check script invokes:

``` text
/usr/local/sbin/rotate-erpnext-acr-token.sh
```

The rotation implementation uses the two password slots associated with
the ACR token:

``` text
password1
password2
```

Only one slot is treated as active at a time. The active slot is tracked
in Azure Key Vault:

``` text
acr-k3s-erpnext-dev-active-slot
```

The script always generates the replacement credential in the
**inactive** slot first.

Verified implementation:

``` bash
#!/bin/bash
set -euo pipefail

ACR="cruxeplatformuaen01"
TOKEN="k3s-erpnext-dev"

KV="kv-uxe-dev-uaen"
PASSWORD_SECRET="acr-k3s-erpnext-dev-password"
SLOT_SECRET="acr-k3s-erpnext-dev-active-slot"

ERP_HOST="vm-uxe-dev-erpnext-01"
ERP_USER="azureuser"
ERP_KEY="/home/azureuser/.ssh/ssh-uxe-dev-erpnext-uaen.pem"

TEST_IMAGE="cruxeplatformuaen01.azurecr.io/erpnext/dev:test-v16"

echo "Authenticating to Azure..."
timeout 30 az login --identity --output none

CURRENT_SLOT=$(az keyvault secret show \
  --vault-name "$KV" \
  --name "$SLOT_SECRET" \
  --query value \
  -o tsv)

case "$CURRENT_SLOT" in
  password1)
    NEW_SLOT="password2"
    SLOT_ARG="--password2"
    ;;
  password2)
    NEW_SLOT="password1"
    SLOT_ARG="--password1"
    ;;
  *)
    echo "ERROR: invalid current slot: $CURRENT_SLOT"
    exit 1
    ;;
esac

echo "Current credential: $CURRENT_SLOT"
echo "Rotating to: $NEW_SLOT"

NEW_PASSWORD=$(az acr token credential generate \
  --name "$TOKEN" \
  --registry "$ACR" \
  "$SLOT_ARG" \
  --days 180 \
  --query 'passwords[0].value' \
  --output tsv)

if [ -z "$NEW_PASSWORD" ]; then
  echo "ERROR: failed to generate new ACR password"
  exit 1
fi

echo "Waiting for ACR credential propagation..."
sleep 75

echo "Updating K3s registry credential..."

printf '%s\n' "$NEW_PASSWORD" | \
ssh \
  -i "$ERP_KEY" \
  -o IdentitiesOnly=yes \
  -o BatchMode=yes \
  "$ERP_USER@$ERP_HOST" \
  'read -r TOKEN_PWD

   sudo tee /etc/rancher/k3s/registries.yaml >/dev/null <<EOF2
configs:
  "cruxeplatformuaen01.azurecr.io":
    auth:
      username: "k3s-erpnext-dev"
      password: "${TOKEN_PWD}"
EOF2

   sudo chown root:root /etc/rancher/k3s/registries.yaml
   sudo chmod 600 /etc/rancher/k3s/registries.yaml

   sudo systemctl restart k3s

   for i in $(seq 1 30); do
     if kubectl get node vm-uxe-dev-erpnext-01 \
       -o jsonpath="{.status.conditions[?(@.type==\"Ready\")].status}" \
       | grep -q True; then
       break
     fi
     sleep 5
   done

   sudo k3s crictl pull \
     cruxeplatformuaen01.azurecr.io/erpnext/dev:test-v16
  '

echo "K3s ACR pull validation successful."

EXPIRY=$(date -u -d '+180 days' +'%Y-%m-%dT%H:%M:%SZ')

echo "Updating Key Vault..."

az keyvault secret set \
  --vault-name "$KV" \
  --name "$PASSWORD_SECRET" \
  --value "$NEW_PASSWORD" \
  --output none

az keyvault secret set-attributes \
  --vault-name "$KV" \
  --name "$PASSWORD_SECRET" \
  --expires "$EXPIRY" \
  --output none

az keyvault secret set \
  --vault-name "$KV" \
  --name "$SLOT_SECRET" \
  --value "$NEW_SLOT" \
  --output none

unset NEW_PASSWORD

echo "========================================"
echo "ACR credential rotation successful"
echo "Active slot: $NEW_SLOT"
echo "Next expiry: $EXPIRY"
echo "========================================"
```

### Dual-Slot Rotation Logic

The rotation process is intentionally designed so the current working
credential is not overwritten before the replacement credential has been
validated.

``` text
Read active slot from Key Vault
        |
        v
Current = password1 ? -> generate password2
Current = password2 ? -> generate password1
        |
        v
Generate new inactive-slot password
        |
        v
Wait 75 seconds for ACR propagation
        |
        v
SSH to vm-uxe-dev-erpnext-01
        |
        v
Update /etc/rancher/k3s/registries.yaml
        |
        v
Set owner root:root and mode 600
        |
        v
Restart K3s
        |
        v
Wait for node Ready
        |
        v
Test ACR pull using test-v16 image
        |
        +--> Failure -> script exits; Key Vault active slot is NOT changed
        |
        v
Update Key Vault password secret
        |
        v
Set new 180-day expiry
        |
        v
Update active-slot secret
        |
        v
Rotation complete
```

### ACR Token and Slot Selection

ACR:

``` text
cruxeplatformuaen01
```

Token:

``` text
k3s-erpnext-dev
```

The script selects the inactive slot according to:

  Current active slot   New generated slot
  --------------------- --------------------
  `password1`           `password2`
  `password2`           `password1`

Any other value causes the rotation to fail immediately:

``` text
ERROR: invalid current slot
```

This prevents the script from rotating against an unknown or
inconsistent slot state.

### New Credential Generation

The replacement password is generated with:

``` bash
az acr token credential generate \
  --name k3s-erpnext-dev \
  --registry cruxeplatformuaen01 \
  --password1-or-password2 \
  --days 180
```

The script verifies that a non-empty password was returned before
continuing.

The generated credential is held only in the shell variable:

``` text
NEW_PASSWORD
```

and is removed from the shell environment with:

``` bash
unset NEW_PASSWORD
```

after the Key Vault update completes.

### ACR Propagation Delay

After generating the new slot, the script waits:

``` text
75 seconds
```

before attempting to use it:

``` bash
sleep 75
```

This allows time for the new ACR token credential to propagate before
K3s is switched to it.

### K3s Credential Update

The CI/CD VM connects to the ERPNext VM using:

``` text
Host: vm-uxe-dev-erpnext-01
User: azureuser
SSH key: /home/azureuser/.ssh/ssh-uxe-dev-erpnext-uaen.pem
```

SSH is executed with:

``` text
IdentitiesOnly=yes
BatchMode=yes
```

The new password is passed through standard input rather than as a
command-line argument.

K3s registry configuration is rewritten at:

``` text
/etc/rancher/k3s/registries.yaml
```

Resulting structure:

``` yaml
configs:
  "cruxeplatformuaen01.azurecr.io":
    auth:
      username: "k3s-erpnext-dev"
      password: "<new active credential>"
```

The file is then secured with:

``` bash
sudo chown root:root /etc/rancher/k3s/registries.yaml
sudo chmod 600 /etc/rancher/k3s/registries.yaml
```

### K3s Restart and Readiness Check

After the registry credential is updated:

``` bash
sudo systemctl restart k3s
```

The script checks node readiness for up to 30 iterations:

``` text
30 checks x 5 seconds
= up to approximately 150 seconds
```

Readiness condition:

``` text
.status.conditions[?(@.type=="Ready")].status == True
```

### ACR Pull Validation

Before updating Key Vault state, the script validates the new credential
by pulling:

``` text
cruxeplatformuaen01.azurecr.io/erpnext/dev:test-v16
```

using:

``` bash
sudo k3s crictl pull \
  cruxeplatformuaen01.azurecr.io/erpnext/dev:test-v16
```

This is the most important safety gate in the rotation sequence.

If the image pull fails, `set -euo pipefail` causes the script to
terminate before:

-   overwriting the Key Vault password secret,
-   changing the Key Vault active-slot value,
-   declaring the new credential active.

### Key Vault Update Order

Only after the K3s pull succeeds does the script update Key Vault.

The order is:

``` text
1. Store the new password
2. Set password-secret expiry to +180 days
3. Change active-slot secret to the new slot
```

Commands:

``` bash
az keyvault secret set \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-password \
  --value "$NEW_PASSWORD"

az keyvault secret set-attributes \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-password \
  --expires "$EXPIRY"

az keyvault secret set \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-active-slot \
  --value "$NEW_SLOT"
```

The next expiry is calculated as:

``` text
current UTC time + 180 days
```

### Rotation Failure Safety

The implemented order is intentionally fail-safe:

``` text
Generate new credential
      |
      v
Update K3s
      |
      v
Test pull
      |
      +---- failure ----> STOP
      |                  Key Vault still points to previous active slot
      |
      v
Update Key Vault
```

This greatly reduces the risk of recording an unvalidated credential as
active.

However, if a failure occurs **after K3s has been updated but before Key
Vault has been updated**, K3s may already be using the new credential
while Key Vault still records the previous slot. In that situation,
operators must inspect both the live `registries.yaml` and ACR slot
state before retrying rotation.

### Post-Rotation Verification

Verify active slot:

``` bash
az keyvault secret show \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-active-slot \
  --query value \
  -o tsv
```

Verify password-secret expiry:

``` bash
az keyvault secret show \
  --vault-name kv-uxe-dev-uaen \
  --name acr-k3s-erpnext-dev-password \
  --query 'attributes.expires || attributes.expiresOn' \
  -o tsv
```

Verify K3s:

``` bash
ssh \
  -i /home/azureuser/.ssh/ssh-uxe-dev-erpnext-uaen.pem \
  -o IdentitiesOnly=yes \
  azureuser@vm-uxe-dev-erpnext-01 \
  'sudo systemctl is-active k3s && kubectl get node'
```

Verify ACR pull:

``` bash
ssh \
  -i /home/azureuser/.ssh/ssh-uxe-dev-erpnext-uaen.pem \
  -o IdentitiesOnly=yes \
  azureuser@vm-uxe-dev-erpnext-01 \
  'sudo k3s crictl pull cruxeplatformuaen01.azurecr.io/erpnext/dev:test-v16'
```

Do not display the password contained in
`/etc/rancher/k3s/registries.yaml` during routine verification.

### Systemd Service

Service unit:

``` text
/etc/systemd/system/erpnext-acr-token-check.service
```

Verified configuration:

``` ini
[Unit]
Description=Check ERPNext ACR token expiry
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=azureuser
ExecStart=/usr/local/sbin/check-erpnext-acr-token-expiry.sh
```

Important behavior:

-   Runs as `azureuser`.
-   Uses a `oneshot` service.
-   Waits for `network-online.target`.
-   Executes the expiry-check script once per timer activation.
-   Exits after the check/rotation action finishes.

Check service state:

``` bash
systemctl status erpnext-acr-token-check.service --no-pager
```

Review logs:

``` bash
journalctl \
  -u erpnext-acr-token-check.service \
  -n 50 \
  --no-pager
```

### Systemd Timer

Timer unit:

``` text
/etc/systemd/system/erpnext-acr-token-check.timer
```

Verified configuration:

``` ini
[Unit]
Description=Daily ERPNext ACR token expiry check

[Timer]
OnCalendar=*-*-* 08:00:00
Persistent=true
RandomizedDelaySec=10m
Unit=erpnext-acr-token-check.service

[Install]
WantedBy=timers.target
```

The expiry check runs daily at approximately:

``` text
08:00 to 08:10
```

because `RandomizedDelaySec=10m` adds up to ten minutes of randomized
delay.

`Persistent=true` means a missed timer activation can run after the
timer becomes active again.

Verify the timer:

``` bash
systemctl status erpnext-acr-token-check.timer --no-pager
systemctl list-timers --all | grep erpnext-acr
```

Enable it if required:

``` bash
sudo systemctl enable --now erpnext-acr-token-check.timer
```

### Manual Validation

Run the check as the same account used by systemd:

``` bash
sudo -u azureuser /usr/local/sbin/check-erpnext-acr-token-expiry.sh
```

Expected output when outside the rotation window:

``` text
Checking ERPNext K3s ACR credential...
Expiry: <expiration timestamp>
Days remaining: <number>
No rotation required.
```

Inside the rotation window:

``` text
Credential is within rotation window.
```

and the script invokes:

``` text
/usr/local/sbin/rotate-erpnext-acr-token.sh
```

### Failure Conditions

  ---------------------------------------------------------------------
  Condition                          Result
  ---------------------------------- ----------------------------------
  Managed Identity login takes more  Timeout / service failure
  than 30 seconds                    

  Key Vault query takes more than 30 Timeout / service failure
  seconds                            

  Secret expiry is missing           Exit `1`

  Credential is already expired      Exit `1`

  Rotation script fails              Check service fails because
                                     `set -e` is enabled
  ---------------------------------------------------------------------

Investigate failures with:

``` bash
systemctl status erpnext-acr-token-check.service --no-pager

journalctl \
  -u erpnext-acr-token-check.service \
  -n 100 \
  --no-pager
```

### Why Rotation Matters

Existing ERPNext pods can continue running with already-present images
even if the ACR credential later becomes invalid. The failure may only
appear during a future rollout:

``` text
New ERPNext image
      |
      v
K3s/containerd image pull
      |
      v
Expired/invalid ACR credential
      |
      v
401 Unauthorized
      |
      v
ErrImagePull / ImagePullBackOff
```

The daily expiry check therefore protects future deployments from an
unnoticed credential expiry.

### Security Requirements

-   Never commit ACR passwords or tokens to Git.
-   Never include secret values in this README.
-   Store the managed credential in `kv-uxe-dev-uaen`.
-   Use the CI/CD VM Managed Identity for Azure authentication.
-   Maintain an explicit expiration date.
-   Run the expiry check daily.
-   Begin rotation at 30 days or less remaining.
-   Validate the replacement credential before retiring the previous
    credential.
-   Monitor systemd failures.
-   Validate ACR image pulling after every rotation.

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

  ---------------------------------------------------------------------------------
  Purpose           Path / name
  ----------------- ---------------------------------------------------------------
  ERPNext app       `~/git/erpnext-app`
  repository        

  GitOps repository `~/git/erpnext-gitops`

  GitHub runner     `/data/github-runners/erpnext-app`

  DEV kubeconfig    `~/erpnext-dev-kubeconfig.yaml`
  from CI/CD VM     

  ERPNext host      `/data-disk/erpnext-sites/dev-erpnext.uxe.ai/private/backups`
  backups           

  MariaDB host data `/db-disk/mariadb`

  ERPNext sites     `/data-disk/erpnext-sites`
  host data         

  K3s registry      `/etc/rancher/k3s/registries.yaml`
  configuration     

  GitOps SSH key    `~/.ssh/github_cicd`

  ACR               `cruxeplatformuaen01.azurecr.io`

  Key Vault         `kv-uxe-dev-uaen`
  ---------------------------------------------------------------------------------

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
