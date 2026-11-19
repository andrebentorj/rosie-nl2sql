# Rosie on Google Cloud

Rosie runs in your Google Cloud project as one Cloud Run service, reached through an external
HTTPS load balancer on your domain, and reaching your database through a Serverless VPC Access
connector. The template is `deploy/cloud/gcp/` (Terraform). What all three clouds share, and why:
[deploy/cloud/README.md](../deploy/cloud/README.md).

What it creates: three service accounts (service, tools, setup), empty Secret Manager secrets in
your region, an Artifact Registry repository that proxies ghcr.io (Cloud Run cannot pull from
ghcr.io directly), three buckets (the context, the settings file, exports), a Filestore share for
the audit file, the connector, a firewall rule for your database VMs, four jobs, the service and
its load balancer with a Google-managed certificate.

## Before you start

- Terraform 1.7 or newer and the gcloud CLI, signed in with rights to create the resources above
  and enable APIs in the project.
- The VPC network of your database and an unused /28 range in it for the connector. Database VMs:
  their network tags. Cloud SQL with a private address: nothing more.
- A domain name for Rosie whose DNS you can change.
- A GitHub user with read access to the Rosie image and a token with the scope `read:packages`.
- Your model key, and an administrator URL for your database (`postgres://`, `mysql://` or
  `mssql://`, with a password percent-encoded: `#` is `%23`, `@` is `%40`).

**Cost to know about.** The audit file needs a real file system: the template creates a Filestore
Basic HDD instance, whose smallest size is 1 TiB and is billed whatever Rosie uses. If you already
have an NFS share in that network, set `logs_nfs` and no instance is created.

**The NFS share.** The Filestore share the template creates may be mounted only from the
connector's range (`connector_cidr`), read-write, with root not squashed (the prepare job, as root,
gives the audit folder to uid 1000 once). A share you bring with `logs_nfs` must be restricted the
same way: exported to the connector's range only. NFS is not encrypted in transit: the audit
records travel inside your VPC, between the connector and the share.

## 1. Create the base

```bash
cd deploy/cloud/gcp
cp terraform.tfvars.example terraform.tfvars      # fill it in; no secret goes in it; stage = "base"
cp backend.tf.example backend.tf                  # optional but recommended: shared state
terraform init
terraform apply
```

The state holds names, addresses and settings, never a secret value.

## 2. Put three secrets in Secret Manager

`read -rs` keeps the values out of your shell history. The names use the default prefix `rosie`.

```bash
read -rs V && printf %s "$V" | gcloud secrets versions add rosie-ghcr-token --data-file=-
read -rs V && printf %s "$V" | gcloud secrets versions add rosie-openai-api-key --data-file=-
read -rs V && printf %s "$V" | gcloud secrets versions add rosie-db-admin-url --data-file=-
unset V
```

## 3. Prepare, check, then run the setup wizard

Set `stage = "jobs"` and apply again, then run the three jobs in order (Terraform prints them):

```bash
terraform apply
eval "$(terraform output -raw run_prepare)"       # creates the audit folder on the share, once
eval "$(terraform output -raw run_preflight)"     # changes nothing; one tiny model call
eval "$(terraform output -raw run_setup)"         # writes to your database server
```

Each prints a link to its logs (or: the console, Cloud Run, Jobs, the job, Logs). Preflight
prints a fix next to each failed line. Setup creates the two accounts, Rosie's tables and the
context, stores the generated secrets and ends with `done`. Its report lists what a person should
review, personal-data columns first. To change the wizard's options, set `wizard_args` and apply
before running it.

Then remove the administrator URL (every version of it):

```bash
for v in $(gcloud secrets versions list rosie-db-admin-url --filter='state!=DESTROYED' --format='value(name)'); do
  gcloud secrets versions destroy "$v" --secret rosie-db-admin-url --quiet
done
```

## 4. Start Rosie

Set `stage = "service"` and apply again. Point your domain's A record at the address Terraform
prints; Google issues the certificate once the name resolves (it can take an hour).

```bash
terraform apply
terraform output load_balancer_ip
curl -s https://rosie.example.com/api/assistant/health
```

`"status":"ok"` means Rosie answers questions; a 503 says why it refuses (`refusing`). The full
check: `eval "$(terraform output -raw run_doctor)"`.

The service's own `run.app` address is switched off, and its ingress admits only its load
balancer and traffic from inside your networks. The load balancer is public by nature, so the
service stage refuses to plan until you say who may reach it: `allowed_cidrs` (a Cloud Armor
allowlist), or `public = true` to accept every address. The API token callers send is the secret
`rosie-rosie-api-token`. It never goes into a web page: docs/WIDGET.md and docs/IDENTITY.md show
the two ways to put the widget in front of people, and docs/CHAT_PAGE.md the sign-in page (add its
secrets with `extra_secrets` and its settings with `settings`).

## Review and edit the context

The context is a bucket (`terraform output context_bucket`): browse it in the console, or copy it:

```bash
B=$(terraform output -raw context_bucket)
gcloud storage rsync --recursive --exclude '^\.bootstrap/' "gs://$B" ./context
# edit, then:
gcloud storage rsync --recursive --exclude '^\.bootstrap/' ./context "gs://$B"
gcloud run services update rosie --region <region> --update-labels restarted=$(date +%s)   # restart to load it
```

The bucket keeps earlier versions of every file for 90 days. The exclusion leaves `.bootstrap/`
in the bucket: the wizard's report, its answers and `samples.json`, sample values read from your
database. The setup job's log also has the report.

## Day to day

| | |
|---|---|
| Output | Cloud Logging (the console, Cloud Run, rosie, Logs) |
| Health for a monitor | `https://<domain>/api/assistant/health` (200 answers, 503 refuses every question); Cloud Run probes `/api/assistant/health/live` |
| Restart | `gcloud run services update rosie --region <region> --update-labels restarted=$(date +%s)` |
| doctor | `terraform output -raw run_doctor` |
| Other operator commands | `gcloud run jobs execute rosie-tools --region <region> --wait --args=audit-export,--,--from,2026-11-01,--to,2026-11-30,--out,/exports/2026-11`; files land in the exports bucket. Commands that change records need an administrator URL, which this job does not have: run those from the Compose kit on a machine that reaches the database, or add the variable yourself for that run |
| Backups | Your database holds Rosie's tables; the context and settings buckets keep versions; schedule Filestore backups for the audit file (`gcloud filestore backups create`) |
| Rotate the two passwords | add `--rotate-passwords` to `wizard_args`, apply, run setup, restart |
| Several tenants (OEM) | after setup, in the order of deploy/cloud/README.md, "Several tenants (OEM)" |

## Upgrade

Change one line in `terraform.tfvars` and apply:

```hcl
rosie_image = "ghcr.io/andrebentorj/rosie:<new version>"
```

Cloud Run starts a new revision and moves traffic to it when it is live. Read "Upgrading" in
docs/INSTALL.md for every version between yours and the new one; it says when a release needs a
new table or grant. Then run doctor. To go back, set the previous version and apply.

## Remove

Set `deletion_protection = false`, apply, then `terraform destroy`. The buckets refuse to be
deleted while they hold files: empty them first if you mean it. The two database accounts and the
`rosie` database stay: remove them with the statements the wizard printed (docs/INSTALL.md,
"Uninstall").

## Assumptions specific to Google Cloud

- Cloud Run mounts buckets with Cloud Storage FUSE (second generation execution environment),
  as uid 1000 through the mount options; the setup wizard writes the context through it.
- The audit file is on NFS because it is appended per answer and rewritten in place, which a
  bucket mount does not do safely. In a rollout a new revision becomes live before the old one
  stops, so for a few seconds two processes may append to it.
- The Artifact Registry remote repository reads the GitHub token from Secret Manager
  (`versions/latest`); the Artifact Registry service agent may read that secret only.
- `TRUST_PROXY = 2`: Google's load balancer appends its own address after the client's.
- The prepare job runs `busybox` from Docker Hub as root, once, to give the audit folder to uid 1000.
- `gcloud run services update --update-labels` to restart is to be confirmed at the first real
  deployment; any change to the service creates a new revision.
