# Rosie on Azure

Rosie runs in your Azure subscription as one container app in a Container Apps environment
inside your virtual network. The template is `deploy/cloud/azure/main.bicep`. What all three
clouds share, and why: [deploy/cloud/README.md](../deploy/cloud/README.md).

What it creates, in one resource group: a Log Analytics workspace, a Container Apps environment
in your subnet (internal by default, with its private DNS zone), two managed identities (one for
the service and the tools job, one for the setup jobs), two Key Vaults (the runtime secrets, and
a separate one for the administrator URL only), a storage account with four file shares (the
context, the settings file, the audit file, exports), three jobs and the container app.

## Before you start

- The Azure CLI (`az`, with Bicep 0.36 or newer: `az bicep upgrade`), signed in with rights to create
  resources and role assignments in the resource group (Owner, or Contributor plus User Access
  Administrator).
- A subnet of the virtual network that reaches your database: a /27 or larger, delegated to
  `Microsoft.App/environments`, with the service endpoints `Microsoft.Storage` and
  `Microsoft.KeyVault`. Your database's firewall or network security group must admit this
  subnet on the database port, and nothing else needs to.
- A GitHub user with read access to the Rosie image and a token with the scope `read:packages`.
- Your model key, and an administrator URL for your database (`postgres://`, `mysql://` or
  `mssql://`, with a password percent-encoded: `#` is `%23`, `@` is `%40`).
- Your public IP address (the vaults and the storage account refuse every other address).

## 1. Create the base

```bash
cd deploy/cloud/azure
cp main.example.bicepparam main.bicepparam        # fill it in; no secret goes in it; stage = 'base'
az group create -n rosie -l westeurope
az deployment group create -g rosie -f main.bicep -p main.bicepparam
```

The outputs name the two vaults (`mainVaultName`, `setupVaultName`) and the storage account.

## 2. Put three secrets in the vaults

`read -rs` keeps the values out of your shell history, and passing them on standard input
(`--file /dev/stdin`) keeps them out of the process list.

```bash
MAIN=<mainVaultName>; SETUP=<setupVaultName>
read -rs V && printf %s "$V" | az keyvault secret set --vault-name $MAIN -n ghcr-token --file /dev/stdin -o none
read -rs V && printf %s "$V" | az keyvault secret set --vault-name $MAIN -n openai-api-key --file /dev/stdin -o none
read -rs V && printf %s "$V" | az keyvault secret set --vault-name $SETUP -n db-admin-url --file /dev/stdin -o none
unset V
```

If a command is refused, check that `operatorPrincipalId` and `operatorIps` were set.

## 3. Check, then run the setup wizard

Set `stage = 'jobs'` in `main.bicepparam` and deploy again, then:

```bash
az deployment group create -g rosie -f main.bicep -p main.bicepparam
az containerapp job start -g rosie -n rosie-preflight    # changes nothing; one tiny model call
az containerapp job start -g rosie -n rosie-setup        # writes to your database server
```

Read each run in the portal: the job, **Execution history**, **Console logs** (or Log Analytics,
`ContainerAppConsoleLogs_CL`). Preflight prints a fix next to each failed line. Setup creates the
two accounts, Rosie's tables and the context, stores the generated secrets and ends with `done`.
Its report lists what a person should review, personal-data columns first. To change the
wizard's options, set `wizardArgs` and deploy before running it.

Then remove the administrator URL:

```bash
az keyvault secret delete --vault-name $SETUP -n db-admin-url
```

The vault keeps it soft-deleted, unreadable, for 7 days (purge protection is on). To run setup
again within those days, `az keyvault secret recover` it first, then set a new value.

## 4. Start Rosie

Set `stage = 'service'` and deploy again. The output `url` is Rosie's address, HTTPS only:

```bash
az deployment group create -g rosie -f main.bicep -p main.bicepparam --query properties.outputs.url.value -o tsv
curl -s https://<that address>/api/assistant/health
```

With `internal = true` the address resolves only inside your virtual network (and networks that
use its DNS). A public environment (`internal = false`) needs `allowedCidrs`, the addresses that
may reach it, or `allowPublic = true` to accept every address; otherwise the deployment fails.
HTTP is redirected to HTTPS. `"status":"ok"` means Rosie answers questions; a 503 says why it refuses
(`refusing`). The full check: `az containerapp job start -g rosie -n rosie-tools` (doctor).

The API token callers send is the vault secret `rosie-api-token`. It never goes into a web page:
docs/WIDGET.md and docs/IDENTITY.md show the two ways to put the widget in front of people, and
docs/CHAT_PAGE.md the sign-in page (add its secrets with `extraSecrets`, as vault secrets named in
lowercase with dashes, and its settings with `settings`).

**Your own name.** `az containerapp hostname add` then `az containerapp hostname bind` on the app.
A free managed certificate needs a public environment (`internal = false`); for an internal one,
upload your certificate to the environment (`az containerapp env certificate upload`).

## Review and edit the context

The context is the file share `context` of the storage account. Open it in the portal (the storage
account, **Storage browser**, **File shares**), or copy it:

```bash
ST=<storageAccountName>
az storage file download-batch --account-name $ST -s context -d ./context --pattern '[!.]*'
# edit, then:
az storage file upload-batch --account-name $ST -d context -s ./context
az containerapp revision restart -g rosie -n rosie --revision "$(az containerapp show -g rosie -n rosie --query properties.latestRevisionName -o tsv)"
```

The pattern leaves out `.bootstrap/`, the wizard's folder: its report, its answers and
`samples.json`, sample values read from your database. Keep it on the share. The setup job's
log also has the report.

## Day to day

| | |
|---|---|
| Output | Log Analytics, `ContainerAppConsoleLogs_CL` (kept `logRetentionDays`, 90 by default) |
| Health for a monitor | `https://<address>/api/assistant/health` (200 answers, 503 refuses every question); the platform probes `/api/assistant/health/live` |
| Restart | `az containerapp revision restart` (above) |
| doctor | `az containerapp job start -g rosie -n rosie-tools` |
| Other operator commands | change the tools job's arguments, run it, set them back: `az containerapp job update -g rosie -n rosie-tools --args audit-prune`, then `job start`, then `--args doctor`. Files they write go to the `exports` share |
| Backups | Your database holds Rosie's tables. Add the shares to an Azure Backup vault (Azure Files backup) for the context and the audit file; share soft delete keeps a deleted share 30 days |
| Rotate the two passwords | add `--rotate-passwords` to `wizardArgs`, deploy, run setup, restart |
| Several tenants (OEM) | after setup, in the order of deploy/cloud/README.md, "Several tenants (OEM)" |

## Upgrade

Change one line in `main.bicepparam` and deploy:

```bicep
param rosieImage = 'ghcr.io/andrebentorj/rosie:<new version>'
```

Container Apps starts a new revision and moves traffic to it when it is live. Read "Upgrading" in
docs/INSTALL.md for every version between yours and the new one; it says when a release needs a
new table or grant. Then run doctor. To go back, set the previous version and deploy.

## Remove

`az group delete -n rosie`. The vaults stay soft-deleted for their retention period (90 days for
the main one, 7 for the setup one) and cannot be purged early. The two database accounts and the
`rosie` database stay: remove them with the statements the wizard printed (docs/INSTALL.md,
"Uninstall").

## Assumptions specific to Azure

- The environment uses the Consumption workload profile, which is what allows a /27 subnet.
- Container Apps mounts the SMB shares with the storage account key (Azure Files has no other
  way for SMB); the key lives in the environment's storage definition, not in Rosie's settings.
  The shares are mounted as uid 1000 (`mountOptions`), the image's user.
- Container Apps reads Key Vault secrets through the vault firewall with the environment's subnet
  allowed (service endpoint). If your policy requires private endpoints instead, add them and set
  the vaults' and the storage account's public network access to Disabled.
- Container Apps has no read-only root file system option; the root file system is writable.
- In single revision mode a new revision becomes live before the old one stops, so for a few
  seconds of an upgrade two processes may append to the same audit file.
- The template uses Bicep's `fail()` to refuse a public environment without `allowedCidrs` and
  secrets in `settings`; it needs Bicep 0.36 or newer, and Azure Resource Manager evaluating only
  the branch of `if()` it takes.
- The ingress redirecting HTTP to HTTPS with `allowInsecure: false` (Microsoft documents a
  redirect), `az storage file download-batch --pattern '[!.]*'` leaving out `.bootstrap/`,
  `az keyvault secret set --file /dev/stdin`, `az containerapp job update --args` and the column names of `ContainerAppConsoleLogs_CL` for jobs
  are to be confirmed at the first real deployment.
