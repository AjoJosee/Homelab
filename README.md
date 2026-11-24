# Homelab

This repository contains a small, self-hosted application platform built with
Docker Compose. It brings together workflow automation, a PostgreSQL database,
local AI inference, object storage, DNS filtering, and reverse-proxy management
in one deployable stack.

The project is intended for a personal server, home lab, or private network.
It is a starting point rather than a production-hardened distribution: review
the configuration, pin image versions, protect the host, and establish backups
before exposing services to the internet.

## What is included?

| Service | Purpose | Default access |
| --- | --- | --- |
| [n8n](https://n8n.io/) | Workflow automation and integrations | Through the configured hostname and reverse proxy |
| [PostgreSQL](https://www.postgresql.org/) 17 | Persistent database used by n8n | Internal Docker network only |
| [Nginx Proxy Manager](https://nginxproxymanager.com/) | Reverse proxy, HTTPS certificates, and host routing | `http://<server-ip>:81` for administration |
| [Ollama](https://ollama.com/) | Local large-language-model serving | Internal Docker network only |
| [MinIO](https://min.io/) | S3-compatible object storage | Through a configured proxy host |
| [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) | Network-wide DNS filtering and management | Through a configured proxy host |

All services are attached to the external Docker network named `proxy`. This
allows Nginx Proxy Manager to route requests to the other containers without
publishing every application port directly on the host. PostgreSQL and Ollama
are intentionally not published to the host by the Compose file.

## Repository layout

```text
.
├── compose.yml                 # Service definitions and persistent mounts
├── .env.example                # Environment variable template
├── workflows/
│   ├── workflow1.json          # Manual n8n-to-Ollama example
│   ├── workflow2.json          # MinIO/S3 file extraction and Ollama example
│   └── mail_labelling.json     # Gmail labeling agent using Ollama
├── data/                       # Runtime data, created locally and ignored by Git
└── .env                        # Local secrets, created locally and ignored by Git
```

The `data/` directory is not stored in the repository. It contains databases,
application state, model files, certificates, and other runtime data, so it
must be included in your backup plan.

## Architecture

```text
Internet / LAN clients
          |
          v
  Nginx Proxy Manager
     ports 80, 81, 443
          |
          v
   Docker network: proxy
    |       |       |       |       |
   n8n   MinIO  Ollama  AdGuard  PostgreSQL
    |
    +-- PostgreSQL database
    +-- Ollama model service
    +-- imported n8n workflows
```

The Compose file uses bind mounts under `./data`:

| Path | Service data |
| --- | --- |
| `data/n8n` | n8n configuration, credentials, and application state |
| `data/npm` | Nginx Proxy Manager configuration |
| `data/letsencrypt` | Certificates managed by Nginx Proxy Manager |
| `data/postgres` | PostgreSQL database files |
| `data/ollama` | Downloaded Ollama models |
| `data/minio` | MinIO object data |
| `data/adguard/work` | AdGuard Home working data |
| `data/adguard/conf` | AdGuard Home configuration |

Do not delete or move these directories while the corresponding containers
are running. For database-consistent backups, use PostgreSQL's backup tools
instead of relying only on a live filesystem copy.

## Requirements

Before deploying, install:

- Docker Engine with the Compose plugin (`docker compose`)
- A Linux host or another operating system capable of running Docker
- Enough disk space for PostgreSQL, workflow data, MinIO objects, and Ollama
  models
- A DNS name if you want HTTPS access from outside the local host
- A DNS resolver or router configuration if AdGuard Home will serve your LAN

If you plan to use local AI models, also account for their disk, memory, and
CPU/GPU requirements. Ollama can run without a GPU, but model performance will
vary substantially by hardware.

## Quick start

### 1. Clone the repository

```bash
git clone https://github.com/AjoJosee/Homelab.git
cd Homelab
```

### 2. Create the external Docker network

The Compose file declares `proxy` as an external network, so Docker Compose
will not create it automatically:

```bash
docker network create proxy
```

If the network already exists, Docker will report that fact; no action is
needed.

### 3. Create and edit the environment file

```bash
cp .env.example .env
chmod 600 .env
```

Replace every placeholder in `.env` with strong, unique values:

```dotenv
POSTGRES_USER=postgres
POSTGRES_PASSWORD=replace-with-a-long-random-password
POSTGRES_DB=postgres
TZ=Asia/Kolkata
MINIO_ROOT_USER=replace-with-a-unique-admin-name
MINIO_ROOT_PASSWORD=replace-with-a-long-random-password
N8N_HOST=n8n.example.com
```

`N8N_HOST` should be the hostname that users will use to reach n8n. Do not
include `https://`, a path, or a trailing slash. The Compose configuration
uses HTTPS for n8n and builds its webhook URL from this value.

Generate secrets rather than reusing passwords:

```bash
openssl rand -base64 32
```

Never commit `.env`, passwords, API keys, OAuth client secrets, or exported
workflow credentials. The repository ignores `.env` and all `data/` content,
but verify changes with `git status` before pushing.

### 4. Start the stack

```bash
docker compose up -d
```

Check container status and logs:

```bash
docker compose ps
docker compose logs -f --tail=100
```

n8n waits for PostgreSQL's health check before starting. On a first launch,
initialization and image downloads may take several minutes.

### 5. Confirm the services

```bash
docker compose ps
docker network inspect proxy
```

The expected containers are:

```text
n8n
nginx-proxy-manager
postgres
ollama
minio
adguardhome
```

Only Nginx Proxy Manager publishes ports on the host:

- `80/tcp`: HTTP
- `81/tcp`: Nginx Proxy Manager administration UI
- `443/tcp`: HTTPS

The other services communicate through the `proxy` network and are not directly
published by this Compose file.

## Nginx Proxy Manager setup

Nginx Proxy Manager is the intended entry point for the web applications.
Open its administration UI at:

```text
http://<server-ip>:81
```

Complete its initial setup, change the default administrator credentials, and
then create proxy hosts for the services you want to expose. Use Docker service
names as the upstream hostnames because all services share the `proxy` network.
For example:

| Public hostname | Forward hostname | Forward port |
| --- | --- | --- |
| `n8n.example.com` | `n8n` | `5678` |
| `minio.example.com` | `minio` | `9000` |
| `adguard.example.com` | `adguardhome` | See AdGuard's configured web port |

The exact ports and setup screens for MinIO and AdGuard Home can change with
their first-run configuration. Confirm the port from the service logs or
container configuration before creating the proxy host.

For HTTPS:

1. Point the DNS record at the server hosting Nginx Proxy Manager.
2. Ensure ports 80 and 443 reach that server.
3. Create the proxy host in Nginx Proxy Manager.
4. Request a Let's Encrypt certificate for the hostname.
5. Enable “Force SSL” after certificate issuance succeeds.

Do not expose administration panels to the public internet unless they are
protected with strong credentials, firewall rules, and preferably an additional
VPN or access-control layer. In particular, treat Nginx Proxy Manager's port
81, MinIO's administrative console, AdGuard Home, and n8n as administrative
interfaces.

## First-run service configuration

### n8n

Open n8n through the hostname configured in `N8N_HOST` and create the initial
owner account. n8n uses PostgreSQL for its database and stores its local
configuration under `data/n8n`.

When configuring webhooks or external integrations, use the HTTPS hostname
rather than the container name. OAuth providers generally require the exact
callback URL shown by n8n, including the correct scheme and hostname.

### PostgreSQL

PostgreSQL is configured with the values from `.env` and has a health check
used by n8n. It is not published to the host. To open a `psql` session:

```bash
docker compose exec postgres \
  psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

The command above reads shell variables. If they are not exported in your
current shell, use the values from `.env` explicitly or open a shell in the
container first.

Create a logical backup before upgrades or migrations:

```bash
docker compose exec -T postgres \
  pg_dump -U postgres -d postgres > postgres-$(date +%F).sql
```

Adjust the user and database names if you changed the defaults.

### Ollama

Ollama stores downloaded models in `data/ollama`. Pull a model after the
container is running:

```bash
docker compose exec ollama ollama pull <model-name>
docker compose exec ollama ollama list
```

Use the model name selected in the n8n workflow. The n8n Ollama nodes should
connect to the Docker service at `http://ollama:11434`; do not use
`localhost` from inside the n8n container.

### MinIO

MinIO stores objects in `data/minio` and is configured with
`MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`. Complete its initial setup before
using it as an S3-compatible backend. Create the bucket referenced by an
imported workflow, or update the workflow's bucket and object key to match your
own storage layout.

The included `workflow2.json` references the bucket `n8n-files` and the object
`random_anime_story.txt` as an example. Those names are not created
automatically by Compose.

### AdGuard Home

Complete AdGuard Home's setup wizard after the first start. Configure the
listening addresses and DNS port for your network, then update your router or
DHCP server if AdGuard Home is intended to be the LAN's DNS resolver. Do not
point your entire network at an unconfigured instance.

## Included n8n workflows

The JSON files under `workflows/` are export examples. They are inactive in
the repository and are not automatically imported or enabled when the
containers start.

### `workflow1.json`: manual Ollama prompt

This workflow starts with a manual trigger and sends a prompt to an Ollama
model. It is useful for confirming that n8n can reach the local Ollama
container and that the selected model is available.

### `workflow2.json`: S3 file analysis

This workflow:

1. Starts manually.
2. Downloads `random_anime_story.txt` from the `n8n-files` S3 bucket.
3. Extracts text from the file.
4. Sends the extracted content to an Ollama model.

Before running it, configure the S3/MinIO credential, endpoint, bucket, object
key, and Ollama model in n8n.

### `mail_labelling.json`: Gmail labeling agent

This workflow uses a Gmail trigger and Gmail tools with an n8n language-agent
node. It can read messages and labels, create labels, and add labels to
messages, with Ollama providing the chat model.

Treat this workflow as a template. Review the agent instructions and
permissions before connecting a real mailbox. Gmail OAuth credentials must be
created in Google Cloud and configured in n8n; no credentials are included in
this repository. Start with a test mailbox and leave the workflow inactive
until its behavior is verified.

### Importing a workflow

In n8n:

1. Open the **Workflows** view.
2. Choose **Import from File**.
3. Select a JSON file from `workflows/`.
4. Configure or replace missing credentials.
5. Update hostnames, bucket names, object keys, prompts, and model names.
6. Execute it manually and inspect each node's output.
7. Activate it only after testing.

Imported workflows may contain node settings that depend on the n8n version or
installed node packages. Treat exported JSON as configuration, not as a
portable backup of credentials.

## Operations

### View logs

```bash
docker compose logs -f n8n
docker compose logs -f postgres
docker compose logs -f ollama
```

### Restart one service

```bash
docker compose restart n8n
```

### Stop the stack without deleting data

```bash
docker compose down
```

### Update images

The Compose file currently uses several `latest` image tags. To update:

```bash
docker compose pull
docker compose up -d
docker image prune
```

Read the release notes for n8n, PostgreSQL, MinIO, AdGuard Home, Ollama, and
Nginx Proxy Manager before upgrading. For a safer and more reproducible
deployment, replace `latest` with tested version tags and update them
deliberately.

### Check the rendered configuration

Before starting after an edit:

```bash
docker compose config
```

This catches YAML and variable-substitution problems without starting
containers. It also makes it easier to confirm that `.env` values are being
loaded as expected.

## Backups and recovery

Back up both configuration and application data:

- `data/n8n`
- `data/npm`
- `data/letsencrypt`
- `data/postgres`
- `data/ollama` if model downloads are expensive to recreate
- `data/minio`
- `data/adguard`
- `.env`, stored securely and separately from public Git

For PostgreSQL, prefer a logical dump such as `pg_dump` and test restoring it.
For MinIO, back up objects using an S3-aware tool or a consistent filesystem
snapshot. A backup that has never been restored is not a verified recovery
plan.

Do not restore all directories over a running stack. Stop the affected
services, restore the data, verify ownership and permissions, and then start
the services again.

## Security checklist

- Replace every placeholder in `.env`.
- Use unique passwords for PostgreSQL, MinIO, n8n, and Nginx Proxy Manager.
- Keep `.env`, `data/`, OAuth credentials, and API keys out of Git.
- Restrict inbound traffic with a host firewall or router firewall.
- Expose only the ports that are required.
- Put administration interfaces behind a VPN or trusted network when possible.
- Enable HTTPS for public hostnames and renew certificates before expiry.
- Review n8n workflow credentials and Gmail permissions.
- Pin image versions after testing rather than relying on `latest`.
- Update containers and the host operating system regularly.
- Monitor disk usage, especially `data/postgres`, `data/minio`, and `data/ollama`.
- Back up data and periodically test restoration.

## Troubleshooting

### `network proxy declared as external, but could not be found`

Create the network once:

```bash
docker network create proxy
```

### n8n cannot connect to PostgreSQL

Check that PostgreSQL is healthy and that the credentials in `.env` match the
database configuration:

```bash
docker compose ps
docker compose logs postgres
docker compose logs n8n
```

The database hostname inside Compose is `postgres`, not `localhost`.

### A workflow cannot connect to Ollama

Confirm that a model is installed and use `http://ollama:11434` from n8n:

```bash
docker compose exec ollama ollama list
docker compose exec n8n sh -c 'wget -qO- http://ollama:11434/api/tags'
```

If the model is missing, pull it with `docker compose exec ollama ollama pull
<model-name>`.

### HTTPS or webhooks do not work

Confirm that:

- DNS resolves the hostname to the proxy host.
- Ports 80 and 443 reach Nginx Proxy Manager.
- The proxy host forwards to the correct Docker service name and port.
- `N8N_HOST` contains only the hostname.
- The n8n proxy host has a valid certificate.

### A container loses its data after recreation

The persistent data should be under `data/`. Check that the bind-mounted
directory exists and that the container has permission to write to it:

```bash
docker compose config
ls -la data
```

Do not use `docker compose down -v` for this project unless you intentionally
want to remove Docker-managed volumes. Also remember that the primary
persistence here is in bind-mounted directories, so filesystem deletion is
irreversible without a backup.

## Project status and limitations

This repository currently provides a Compose configuration and example
workflows. It does not include:

- Automated image update management
- Automated database or object-storage backups
- Production monitoring and alerting
- Hardware-specific GPU configuration for Ollama
- A pinned release matrix for all images
- Automated n8n workflow deployment

Those are appropriate next steps for a more reliable long-running deployment.

The workflows provided are just me trying out n8n. You can completely ignore that.
