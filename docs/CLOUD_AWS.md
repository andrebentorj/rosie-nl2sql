# Rosie on AWS

Rosie runs in your AWS account as one ECS Fargate task in your VPC, behind an Application Load
Balancer with your certificate. The template is `deploy/cloud/aws/` (Terraform). What all three
clouds share, and why: [deploy/cloud/README.md](../deploy/cloud/README.md).

What it creates: an ECS cluster and service, an internal load balancer (HTTPS only), an
encrypted EFS file system with backups (the context, the settings file, the audit file),
empty Secrets Manager secrets, a CloudWatch log group, one IAM role per job, three security
groups, one rule on your database's security group, and a private S3 bucket used only to hand
files over (emptied after 7 days).

## Before you start

- Terraform 1.7 or newer and the AWS CLI v2, signed in to the account with rights to create the
  resources above.
- The VPC of your database, two private subnets in different zones with a NAT gateway, your
  database's security group and port.
- An ACM certificate in the same region for the name people will use (for example
  `rosie.example.com`).
- A GitHub user with read access to the Rosie image and a token with the scope `read:packages`.
- Your model key, and an administrator URL for your database (`postgres://`, `mysql://` or
  `mssql://`, with a password percent-encoded: `#` is `%23`, `@` is `%40`).

## 1. Create the infrastructure

```bash
cd deploy/cloud/aws
cp terraform.tfvars.example terraform.tfvars      # fill it in; no secret goes in it
cp backend.tf.example backend.tf                  # optional but recommended: shared, encrypted state
terraform init
terraform apply                                   # start_service = false for now
```

The state holds names, addresses and settings, never a secret value.

## 2. Put three secrets in Secrets Manager

`read -rs` keeps the values out of your shell history, and passing them on standard input
(`file:///dev/stdin`) keeps them out of the process list. The names below use the default prefix
`rosie`; `terraform output secret_names` lists them.

```bash
read -rs GH_TOKEN
printf '{"username":"%s","password":"%s"}' YOUR_GITHUB_USER "$GH_TOKEN" |
  aws secretsmanager put-secret-value --secret-id rosie/ghcr-pull --secret-string file:///dev/stdin
read -rs VALUE && printf %s "$VALUE" | aws secretsmanager put-secret-value --secret-id rosie/OPENAI_API_KEY --secret-string file:///dev/stdin
read -rs VALUE && printf %s "$VALUE" | aws secretsmanager put-secret-value --secret-id rosie/DB_ADMIN_URL --secret-string file:///dev/stdin
unset GH_TOKEN VALUE
```

## 3. Check, then run the setup wizard

Both run once in your VPC as one-off tasks. Terraform prints the exact commands:

```bash
eval "$(terraform output -raw run_preflight)"     # changes nothing; makes one tiny model call
aws logs tail /ecs/rosie --follow                 # read the result; Ctrl-C when it ends
eval "$(terraform output -raw run_setup)"         # writes to your database server
aws logs tail /ecs/rosie --follow
```

Preflight prints a fix next to each failed line. Setup creates the two accounts, Rosie's tables
and the context, stores the generated secrets and ends with `done`. Its report lists what a
person should review, personal-data columns first. To change the wizard's options (a project
name, strict masking, other account names), set `wizard_args` and apply before running it.

Then remove the administrator URL:

```bash
aws secretsmanager delete-secret --secret-id rosie/DB_ADMIN_URL --force-delete-without-recovery
```

This removes every version of it. The next `terraform apply` creates the secret again, empty;
put a value in it only when you run setup again.

## 4. Start Rosie

Set `start_service = true` in `terraform.tfvars`, then:

```bash
terraform apply
terraform output load_balancer_dns_name           # point your DNS name at it (CNAME or alias)
curl -s https://rosie.example.com/api/assistant/health
```

`"status":"ok"` means Rosie answers questions. A 503 says why it refuses (`refusing`). The full
check, with the fix next to each problem:

```bash
eval "$(terraform output -raw run_doctor)" && aws logs tail /ecs/rosie --follow
```

The API token callers send is in `rosie/ROSIE_API_TOKEN`. It never goes into a web page:
docs/WIDGET.md and docs/IDENTITY.md show the two ways to put the widget in front of people, and
docs/CHAT_PAGE.md the sign-in page (add its secrets with `extra_secrets` and its settings with
`settings`, then apply).

## Review and edit the context

The context lives on EFS. To look at it and change it, copy it through the hand-over bucket:

```bash
eval "$(terraform output -raw run_transfer)"                      # context -> s3://<bucket>/context/
B=$(terraform output -raw transfer_bucket)
aws s3 sync "s3://$B/context/" ./context
# edit the files, then send back the ones you changed:
aws s3 sync ./context "s3://$B/context-upload/"
eval "$(terraform output -raw run_transfer) --overrides '{\"containerOverrides\":[{\"name\":\"transfer\",\"environment\":[{\"name\":\"ROSIE_TRANSFER\",\"value\":\"context-upload\"}]}]}'"
aws ecs update-service --cluster rosie --service rosie --force-new-deployment   # restart to load it
```

Upload adds and replaces files; it does not delete. Files in the bucket expire after 7 days.
The wizard's folder `.bootstrap/` (its report, its answers and `samples.json`, sample values
read from your database) is never copied to the bucket nor replaced from it; the report is also
in the setup task's log.

## Day to day

| | |
|---|---|
| Output | `aws logs tail /ecs/rosie --follow` (kept `log_retention_days`, 365 by default) |
| Health for a monitor | `https://<name>/api/assistant/health` (200 answers, 503 refuses every question); `/api/assistant/health/live` is what the load balancer and ECS probe |
| Restart | `aws ecs update-service --cluster rosie --service rosie --force-new-deployment` |
| doctor | `terraform output -raw run_doctor` |
| Other operator commands | the tools task with another command: add `--overrides '{"containerOverrides":[{"name":"tools","command":["npm","run","-s","audit-export","--","--from","2026-11-01","--to","2026-11-30","--out","/exports/2026-11"]}]}'`, then copy `/exports` out with `ROSIE_TRANSFER=exports-download`. Commands that change records need `tools_admin_url = true` and a value in `rosie/DB_ADMIN_URL` for that run only |
| Backups | EFS backups are on (AWS Backup default plan). Your database holds Rosie's tables |
| Rotate the two passwords | `wizard_args = [..., "--rotate-passwords"]`, apply, run setup, restart |
| Several tenants (OEM) | after setup, in the order of deploy/cloud/README.md, "Several tenants (OEM)" |

## Upgrade

Change one line in `terraform.tfvars` and apply:

```hcl
rosie_image = "ghcr.io/andrebentorj/rosie:<new version>"
```

ECS stops the old task and starts the new one (a short interruption: one replica, so two never
write the same audit file). Read "Upgrading" in docs/INSTALL.md for every version between yours
and the new one; it says when a release needs a new table or grant. Then run doctor. To go back,
set the previous version and apply.

## Remove

Set `deletion_protection = false`, apply, then `terraform destroy`. Secrets are recoverable for 7
days. The two database accounts and the `rosie` database stay: remove them with the statements the
wizard printed (docs/INSTALL.md, "Uninstall").

## Assumptions specific to AWS

- The two private subnets are in different Availability Zones (one EFS mount target per zone).
- Fargate platform 1.4 or newer (EFS volumes with IAM authorization, `dependsOn`, capability
  drop): `LATEST` is used.
- The image pull uses ECS private registry authentication with the `ghcr-pull` secret; the
  execution role may read only that secret, the service's secrets and the log group.
- ECS Exec is off (it does not work with a read-only root file system); doctor and the operator
  commands run as one-off tasks instead.
- Secrets use the AWS managed key. To use your own KMS key, add it to the secrets and grant the
  roles `kms:Decrypt`.
