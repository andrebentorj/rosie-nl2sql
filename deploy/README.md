# Rosie with Docker Compose

From nothing to a running Rosie on one server, without installing Node.js. Everything runs
from the Rosie image: a check before you start, the setup wizard, and the service.

You need:

- **Docker Engine with Docker Compose 2.24 or newer** (`docker compose version`). Docker
  Desktop on a laptop is fine for a trial.
- **Access to the Rosie image.** You were given either registry access or a file (see
  "Getting the image" at the end).
- **A database URL with an administrator account**, used once by the wizard to create
  Rosie's two accounts and its own database: `postgres://`, `mysql://` or `mssql://` (MariaDB
  is not supported). It is never written to Rosie's settings.
- **A model key**: `OPENAI_API_KEY`, and `AI_BASE_URL` when the provider is not OpenAI
  (docs/MODELS.md).

## 1. Put the kit in a folder

Copy this folder (`compose.yaml`, `Caddyfile`, `.env.example`) to the server, for example
to `/opt/rosie`, and create the two folders the wizard writes to:

```bash
cd /opt/rosie
mkdir -p context config
```

## 2. Linux only: let the container write those folders

The image runs as the user with uid 1000. On Linux, give it the two folders (Docker Desktop
on macOS and Windows does not need this):

```bash
sudo chown 1000:1000 context config
```

## 3. Check before installing

Put the administrator URL and the model key in the environment of this shell. A password
with special characters is percent-encoded (`#` is `%23`, `@` is `%40`).

```bash
export DB_URL='postgres://admin:password@db.example.com:5432/sales'
export OPENAI_API_KEY='sk-...'
# export AI_BASE_URL='https://gateway.example.com/v1'   # only for a provider other than OpenAI

docker compose run --rm preflight
```

It changes nothing. It checks that the database host answers, that the account logs in and
sees the tables, that it may create accounts and a database, that the key and base URL are
well formed, and that the two folders are writable. Each failed line prints its fix. Add
`--call-model` at the end to also make one tiny model call (a fraction of a cent).

**A database on the same server** is `host.docker.internal` in the URL, not `localhost` or
`127.0.0.1`: inside a container those mean the container itself. The kit makes the name work
on Linux too.

## 4. Run the setup wizard

```bash
docker compose run --rm init --name "Sales"
```

It reads the schema, writes the context to `./context` and Rosie's settings to
`./config/.env`, creates the read-only account `rosie_readonly` and the engine account
`rosie_engine`, and Rosie's own tables in a separate database, `rosie`. It prints what it
will write before it writes it. Read its report: it lists what a person should review,
personal-data columns first. The wizard's other options go after `init`
(`docker compose run --rm init --help`), for example `--ro-user`, `--engine-user` and
`--engine-db` when another installation already uses those names on the same server.

Then forget the administrator URL:

```bash
unset DB_URL
```

## 5. Start Rosie

```bash
docker compose up -d
docker compose ps                         # STATUS turns to "healthy" within a minute
curl -s http://127.0.0.1:3001/api/assistant/health
```

`healthy` means Rosie answers questions. `unhealthy` means it is up but refusing every
question, and the health answer says why (`refusing`). Run the full check, which prints the
fix next to each problem:

```bash
docker compose exec rosie npm run doctor
```

Rosie listens on `127.0.0.1:3001`, this machine only. To reach it from other machines, use
the HTTPS proxy below. (For a closed test network only: `ROSIE_BIND=0.0.0.0` in `.env`
publishes it unencrypted.)

## 6. HTTPS (optional)

The kit includes Caddy, which gets and renews a certificate by itself. Point a DNS name at
this server, open ports 80 and 443 to it, then:

```bash
cp .env.example .env
echo 'ROSIE_DOMAIN=rosie.example.com' >> .env
```

In `config/.env`, set `TRUST_PROXY=1` (so the quotas count per person, not per proxy) and
`ROSIE_CORS_ORIGIN` to the address of the pages that embed the widget, then:

```bash
docker compose --profile proxy up -d
curl -s https://rosie.example.com/api/assistant/health
```

**The API token never goes into a web page.** The proxy does not add it. Either your host
application's own proxy forwards `/api/assistant/` and adds the token
(docs/INSTALL.md, Step 6d), or people sign in with JWT (docs/IDENTITY.md). Embedding the
widget is docs/WIDGET.md.

## Day to day

| | |
|---|---|
| Logs | `docker compose logs -f rosie` (rotated: 5 files of 10 MB) |
| The audit file | in the `rosie-logs` volume, mounted at `/app/logs` |
| Restart after editing `context/` or `config/.env` | `docker compose up -d --force-recreate rosie` |
| Health for a monitor | `/api/assistant/health` (200 answers, 503 refuses every question), `/api/assistant/health/live` (the process is up) |
| Back up | the `context`, `config` and `exports` folders, and the `rosie-logs` volume |
| Connect an AI client (MCP) | `docker compose exec rosie npm run mcp:config` prints the client entry (`docker compose -f <this folder>/compose.yaml exec -T rosie node server/mcp/stdio-server.js`) |

The service restarts by itself after a crash or a reboot (`restart: unless-stopped`), runs
with a read-only file system and no extra privileges, and is limited to 2 CPUs and 1 GB of
memory (it uses about 400 MB under load, docs/CAPACITY.md). Change the limits in
`compose.yaml` if you need to.

## Compliance and other operator commands

The commands of docs/COMPLIANCE.md and docs/OPERATIONS.md run in a one-off container of the
`tools` service: the same image, `config/.env`, the context and the audit volume, a read-only
file system. Whatever they write goes to `/exports`, which is the `exports` folder next to
`compose.yaml` (create it once; on Linux `sudo chown 1000:1000 exports`):

```bash
mkdir -p exports

# A person's records (GDPR access and erasure requests)
docker compose run --rm tools subject -- --user ana@example.com
docker compose run --rm tools subject -- --user ana@example.com --export /exports/ana.json
docker compose run --rm tools subject -- --user ana@example.com --erase --dry-run

# An evidence bundle for a period, and its check (also works on a copy, anywhere)
docker compose run --rm tools audit-export -- --from 2026-11-01 --to 2026-11-30 --out /exports/evidence-2026-11
docker compose run --rm tools audit-verify -- /exports/evidence-2026-11

# Retention: the report first, then the removal
docker compose run --rm tools audit-prune
```

The commands that change records (`subject --erase`, `audit-prune --confirm`) need an
administrator connection to the engine database. Give it for that command only, from the shell,
never in `config/.env`; the `tools` service passes `ROSIE_ADMIN_DB_URL` through from the shell:

```bash
read -rs ROSIE_ADMIN_DB_URL && export ROSIE_ADMIN_DB_URL      # paste the URL; nothing is echoed
docker compose run --rm tools subject -- --user ana@example.com --erase
docker compose run --rm tools audit-prune -- --confirm
unset ROSIE_ADMIN_DB_URL
```

**Retention every night, from the host's cron.** Keep the database URL for the prune in a file
only root can read: `/opt/rosie/admin-url`, owned by root with mode 600
(`sudo install -o root -g root -m 600 /dev/null /opt/rosie/admin-url`, then write the URL into it).
It must stay outside every folder or volume mounted into the containers (`context`, `config`,
`exports`, the audit volume), so the running service can never read it. Prefer an account made
for the prune over a full administrator where your engine allows it: SELECT and DELETE on
`rosie_audit_log`, `rosie_answer_certificate` and `rosie_feedback`, and INSERT on
`rosie_audit_log` (the run records itself; on PostgreSQL also USAGE on `rosie_audit_log_id_seq`), nothing else; it must not be the server's account
(`DB_USER`), which the command refuses. Then in root's crontab:

```cron
15 3 * * *  cd /opt/rosie && ROSIE_ADMIN_DB_URL="$(cat /opt/rosie/admin-url)" docker compose run --rm -T tools audit-prune -- --confirm --by cron >> /var/log/rosie-prune.log 2>&1
```

`-T` because cron has no terminal. The period is `ROSIE_AUDIT_RETENTION_DAYS` in
`config/.env` (0, the default, keeps everything and the command removes nothing).

## Upgrade

```bash
echo 'ROSIE_IMAGE=ghcr.io/andrebentorj/rosie:<new version>' >> .env   # or edit the line
docker compose pull rosie
docker compose up -d
docker compose exec rosie npm run doctor
```

The context, the settings and the audit volume are outside the image and are kept. Read
"Upgrading" in docs/INSTALL.md for every version between yours and the new one: it says when a
release needs a new table or grant. To go back, set the previous version and run
`docker compose up -d` again.

## Remove

```bash
docker compose --profile proxy down       # stops and removes the containers
docker volume rm rosie_rosie-logs         # the audit file: keep a copy first if you must
```

Then delete the two accounts and the engine database with the statements the wizard printed
at the end of its report (also in docs/INSTALL.md, "Uninstall"), and the `context` and
`config` folders.

## Getting the image

With registry access, log in once with the token you created (scope `read:packages`):

```bash
echo "$TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USER --password-stdin
```

With a file, load it: `docker load < rosie-<version>.tar.gz`. Then set `ROSIE_IMAGE` in
`.env` to the name it printed. How to verify the signature: docs/SECURITY_OVERVIEW.md,
"Verify what you received".
