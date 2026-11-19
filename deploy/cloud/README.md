# Rosie in your own cloud account

Templates that run Rosie in your AWS, Azure or Google Cloud account, next to your database,
from the same image as the Docker Compose kit (`deploy/README.md`). Nothing runs in a Rosie
account: the image is pulled from the registry you were given access to, the secrets stay in
your cloud's secret store, and the model is called from your network.

| Cloud | What runs Rosie | Template | Guide |
|---|---|---|---|
| AWS | ECS Fargate behind an Application Load Balancer | `aws/` (Terraform) | [docs/CLOUD_AWS.md](../../docs/CLOUD_AWS.md) |
| Azure | Container Apps in your virtual network | `azure/` (Bicep) | [docs/CLOUD_AZURE.md](../../docs/CLOUD_AZURE.md) |
| Google Cloud | Cloud Run behind an external HTTPS load balancer | `gcp/` (Terraform) | [docs/CLOUD_GCP.md](../../docs/CLOUD_GCP.md) |

## The same shape in all three

- **One service, one replica, always on.** The audit file, the context reload and the review
  queue are per process (docs/CAPACITY.md), and scheduled answers need a running process. An
  upgrade replaces the one replica.
- **HTTPS only.** AWS: your ACM certificate on the load balancer, TLS 1.2 or newer, port 80
  only redirects. Azure: the Container Apps ingress, HTTP redirected to HTTPS. Google Cloud: a
  Google-managed certificate on your domain, TLS 1.2 or newer, port 80 only redirects.
- **Private by default where this template supports it, and never open by accident.** AWS: an
  internal load balancer unless `internet_facing = true`. Azure: an internal environment;
  `internal = false` needs `allowedCidrs`, or `allowPublic = true`, or the deployment fails.
  Google Cloud: the load balancer is public by nature, so it needs `allowed_cidrs` (Cloud
  Armor), or `public = true`, or the plan fails; the service's own `run.app` address is
  switched off and its ingress admits only the load balancer and traffic from inside your
  networks. In every case Rosie's own token or sign-in decides who may ask.
- **Rosie gets one narrow path to the database.** AWS: one rule on your database's security
  group, its port from Rosie's group only. Google Cloud: a firewall rule from the connector's
  /28 only, for database VMs (Cloud SQL with a private address needs none). Azure: the subnet
  you give is Rosie's alone; your database's own firewall or network security group admits it
  (the template does not change resources it does not own). None of this removes other paths
  to your database that already exist.
- **Secrets in the cloud's secret store, never in the template or its state.** The templates
  create empty secrets. You put three values in them (the GitHub token that pulls the image,
  the model key, the administrator database URL); the setup job puts the four it generates
  (the two account passwords, the API token, the handle secret). The running service gets them
  as environment variables from the platform.
- **The settings file.** The setup job writes Rosie's other settings, without any secret, to
  `/settings/rosie.env`; the image's own dotenv loads it before `server.js`
  (`NODE_OPTIONS=--require dotenv/config`). The template's `settings` and the secrets are
  environment variables and win over it.
- **Health.** The platform probes `/api/assistant/health/live` (the process answers). Point
  your monitor at `/api/assistant/health`: it answers 503 while every question is refused and
  says why (a security hold, the model key). It is not used for restarts on purpose, since a
  restart with the same configuration fixes none of its causes, and a load balancer that hid
  it would hide that answer too.
- **The setup wizard runs once as a job** in your network, against your database: preflight
  first (changes nothing), then setup. The job is `setup/rosie-cloud-setup.js`, which every
  template passes to the image as `node -e`; it needs no change to the image.
- **Upgrades** change one value (the image tag) and apply.
- **The wizard's own folder stays put.** The context folder holds `.bootstrap/`: the wizard's
  report, its answers and `samples.json`, sample values read from your database (personal-data
  columns excluded, but real values). The guides' copy commands leave it out, and it should not
  be copied off the cloud storage or into a repository.
- **The setup job only talks to the official endpoints.** Secrets Manager, Key Vault and Secret
  Manager over HTTPS, the platform's own credential address (the ECS rule of the AWS SDKs, the
  Container Apps loopback, Google's metadata server), and it never follows a redirect. No
  setting moves an endpoint, and the templates refuse settings named like the job's or the
  cloud identity's (`ROSIE_SETUP_*`, `ROSIE_CLOUD`, `AWS_*`, `GCE_*`, `IDENTITY_*`,
  `AZURE_CLIENT_ID`). A token that expires while the wizard runs is fetched again once.
- **Preflight has its own identity**: it may read the administrator URL and the model key, and
  mount the context folder writable so it can check the wizard could write there (it writes
  nothing); nothing else.

## What the setup job does

1. Reads the administrator database URL and the model key from the secret store, with the
   job's own cloud identity.
2. If an earlier setup stored its secrets, writes them to a private seed `.env` so the wizard
   keeps the API token, the handle secret and the two accounts whose password still works.
3. Runs `scripts/rosie-init.js --yes --create-ro-user --create-engine-tables` with your extra
   options: it reads the schema, writes the context to the mounted context folder, creates the
   read-only account `rosie_readonly`, the engine account `rosie_engine` and Rosie's tables in
   the database `rosie`, and prints its report (also in the context folder,
   `.bootstrap/report.txt`). On a re-run it keeps the context files that exist.
4. Puts the four generated secrets in the secret store (only those that changed) and writes the
   settings file without any secret.
5. Removes its private folder. The `.env` with the passwords only ever exists in the job's
   temporary directory.

Values are never printed. Nothing is stored if the wizard wrote no settings; if it wrote them and
then failed, what it wrote is stored and the job ends with the wizard's code.

## Several tenants (OEM)

Tenancy ([docs/OEM.md](../../docs/OEM.md)) works the same in these deployments. The setup job
needs nothing for it: the wizard writes an ordinary context and the engine tables, which already
have the audit `tenant_id` column. Turn tenancy on after setup, in this order:

1. Edit the context (the guide's "Review and edit the context"): declare `"tenancy"` in
   `project.json` and the row policies (one shared database), or put the tenants file in the
   context folder (`tenants.json`) with each tenant's database, and any tenant context directory
   as a subfolder of it (`"context": "tenants/acme"`, a path relative to the file). The context
   folder is mounted read-only into the service, which is where these belong.
2. Add the verified sign-in settings (`ROSIE_JWT_ISSUER`, `ROSIE_JWT_AUDIENCE`,
   `ROSIE_JWT_JWKS_URL`, docs/IDENTITY.md) and `ROSIE_TENANT_CLAIM`, plus
   `ROSIE_TENANTS_FILE = "/context/tenants.json"` when there is a file, to the template's
   `settings`, and apply.
3. Restart, and run doctor. A context that does not declare its tenancy, or a table with the
   tenant column and no policy, keeps the service from starting; its last lines in the log name
   the problem.

Do not set `ROSIE_TENANT_CLAIM` before the context declares its tenancy: the service would refuse
to start. Under tenancy the static API token reads no data, and Slack and Teams cannot be on. The
billing report runs in the tools job: `npm run usage -- --by tenant --from ... --to ...
--format csv --out /exports/usage.csv` (the guide's "Other operator commands").

## Assumptions

Each guide lists its own. Common to all three:

1. Your database is reachable from a private network in the same region (a VPC or a virtual
   network you already have), and MariaDB is not supported (as in every installation).
2. You were given access to the private image on ghcr.io (a GitHub account with the Read role)
   and created a token with the single scope `read:packages`.
3. The private network has outbound internet access (a NAT gateway or equivalent) for the
   model provider and ghcr.io, unless you point `AI_BASE_URL` at a model inside your network.
4. The image is built for x86-64; the templates ask for x86-64.
5. TLS to the database is a setting (`DB_SSL`, in `settings`), off unless you set it. Managed
   databases usually want `require`.
6. The account in the administrator URL may create accounts and a database (preflight checks).
7. `TRUST_PROXY` is set for the platform's proxy hops: 1 on AWS (the load balancer appends the
   client's address) and Azure (the ingress does), 2 on Google Cloud, where the load balancer
   appends the client's address and then its own. With the `run.app` address off, every request
   on Google Cloud comes through that load balancer, so the count holds. If every person shares
   one quota, check the client address in the audit records and adjust it in `settings`.
8. None of this was applied to a real cloud account before this release. The templates were
   checked offline only (below); the first deployment for a customer is also the first real run
   of these templates, and its findings go back into them.

## Checking the templates offline

No credentials and no call to a cloud API:

```bash
cd deploy/cloud/aws && terraform init -backend=false && terraform fmt -check && terraform validate && terraform test
cd deploy/cloud/gcp && terraform init -backend=false && terraform fmt -check && terraform validate && terraform test
cd deploy/cloud/azure && az bicep build --file main.bicep --stdout > /dev/null && az bicep lint --file main.bicep   # Bicep 0.36 or newer (fail())
node test/unit/cloud-setup.js     # the setup job against fake secret stores, and the templates' shape
```

`terraform test` applies each configuration to a mocked provider (`tests/plan.tftest.hcl`): it
runs the provider's own checks of every value and the assertions there (the stages, one replica,
the read-only context, secrets never given as settings), without creating anything.
