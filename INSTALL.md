# Installation

Deploy in order: **infra → variant-pipeline → clinvar-pipeline → workflow-service → variant-mcp-server → agent-service → sample-service → vap-ui**. Each repo depends on the one before it.

---

## Prerequisites

Install these once, shared across all repos:

- [Poetry](https://python-poetry.org/docs/#installation)
- [Pulumi CLI](https://www.pulumi.com/docs/install/)
- [gcloud CLI](https://cloud.google.com/sdk/docs/install) — authenticated via `gcloud auth login && gcloud auth application-default login`
- `wireguard-tools` — for key generation (`brew install wireguard-tools` on macOS)

---

## 1. infra

Provisions the GCP base layer: VPC, ClickHouse VM, WireGuard VPN gateway, IAM, Secret Manager, GCS buckets, Artifact Registry, and Firestore.

### Configure

```bash
cd infra
cp .env.example .env
```

Edit `.env`:

```
google-cloud-project=your-gcp-project-id
workstation_ip=1.2.3.4/32              # curl -s ifconfig.me, then append /32
PULUMI_CONFIG_PASSPHRASE_FILE=/path/to/passphrase-file
clickhouse_password=your-db-password
anthropic_api_key=sk-ant-...
wireguard_server_private_key=          # generated below
```

Generate a WireGuard server key (one-time):

```bash
wg genkey | tee /tmp/wg-server-private.key | wg pubkey > /tmp/wg-server-public.key
cat /tmp/wg-server-private.key   # paste into wireguard_server_private_key in .env
```

Set up the Pulumi passphrase file:

```bash
echo -n "your-passphrase" > ~/.pulumi/passphrase && chmod 600 ~/.pulumi/passphrase
```

### Bootstrap and deploy

```bash
poetry lock
poetry install
poetry run poe bootstrap   # creates GCS state bucket, inits the dev stack — run once

poetry run poe preview     # review what will be created
poetry run poe up          # apply
```

> After the first `poe up`, wait 1–2 minutes for GCP API activation to propagate before running it again.

### Post-deploy

```bash
poetry run poe secrets-push    # push secret values from .env into Secret Manager
```

> Wait 1–2 minutes after `poe up` for the ClickHouse VM startup script to complete before running `schema-apply`.

```bash
poetry run poe schema-apply    # create the variants and annotations tables in ClickHouse
```

```bash
poetry run poe load-samples    # load 1000 Genomes sample metadata into Firestore (idempotent)
```

### Connect via VPN

Generate a client key pair:

```bash
mkdir -p ~/.wireguard && chmod 700 ~/.wireguard
wg genkey | tee ~/.wireguard/client-private.key | wg pubkey > ~/.wireguard/client-public.key
chmod 600 ~/.wireguard/client-private.key
```

Register the client with the gateway:

```bash
poetry run poe vpn-add-peer
# Peer public key: <contents of ~/.wireguard/client-public.key>
# VPN IP: 10.10.0.2   (use .3, .4, ... for additional users)
```

Create `variant-platform.conf`:

```ini
[Interface]
Address = 10.10.0.2/32
PrivateKey = <contents of ~/.wireguard/client-private.key>
DNS = 10.10.0.1

[Peer]
PublicKey = <server public key — from `poe vpn-status`>
Endpoint = <vpn_gateway_external_ip — from `poe outputs`>:51820
AllowedIPs = 10.128.0.0/20, 10.10.0.0/24, 199.36.153.4/30
PersistentKeepalive = 25
```

`DNS = 10.10.0.1` routes DNS through dnsmasq on the VPN gateway → Cloud DNS private zone → `*.run.app` resolves to Google's restricted VIPs, making Cloud Run internal-ingress services reachable. `199.36.153.4/30` ensures HTTPS to those VIPs routes through the tunnel.

Import and activate: WireGuard app on macOS/Windows, or `sudo wg-quick up variant-platform.conf` on Linux. Verify with `ping 10.128.0.3`.

---

## 2. variant-pipeline

Downloads a VCF from 1000 Genomes, normalises it with bcftools (Cloud Batch), and loads it into the ClickHouse `variants` table.

**Requires:** infra fully deployed, ClickHouse running, schema applied, VPN connected.

### Install and deploy

```bash
cd variant-pipeline
poetry install

# First time only:
poetry run poe login        # authenticate Pulumi to the GCS backend
poetry run poe stack-init   # init the dev stack

# Build Docker images and deploy:
poetry run poe build-all
poetry run poe deploy
```

### Trigger a run

```bash
poetry run poe trigger -- \
  --individual HG00096 \
  --bucket genomic-variant-prototype-variant-processing \
  --clickhouse-host 10.128.0.3
```

Add `--force-normalize` or `--force-load` to bypass idempotency caches on a re-run.

---

## 3. clinvar-pipeline

Downloads ClinVar from NCBI, loads it into the ClickHouse `annotations` table, and enriches it with the `variant_summary` TSV. Runs monthly via Cloud Scheduler or on demand.

**Requires:** infra fully deployed, `annotations` table exists in ClickHouse, VPN connected.

### Install and deploy

```bash
cd clinvar-pipeline
cp .env.example .env
# Edit .env: set PULUMI_CONFIG_PASSPHRASE_FILE to the same passphrase file used in infra

poetry install

# First time only:
poetry run poe login
poetry run poe stack-init

# Build Docker images and deploy:
poetry run poe build-all
poetry run poe deploy
```

Deploys the download Cloud Function, the `clinvar-refresh` Cloud Workflow, and a Cloud Scheduler job (1st of each month, 06:00 UTC).

### Trigger manually

The easiest way is the **vap-ui Submit Pipeline → ClinVar Refresh** tab. Or via CLI:

```bash
poetry run poe trigger -- \
  --bucket genomic-variant-prototype-variant-processing \
  --clickhouse-host 10.128.0.3
```

The workflow always re-downloads from NCBI. If the VCF `##fileDate=` version matches the currently loaded version in Firestore, the pipeline returns early (`up_to_date`) without running the Batch jobs.

---

## 4. workflow-service

RESTful Pipeline API — submit and monitor VCF ingest and ClinVar refresh pipelines. Backed by Firestore for state and Cloud Workflows for execution.

**Requires:** infra deployed, variant-pipeline and clinvar-pipeline deployed, VPN connected.

### Install and deploy

```bash
cd workflow-service
cp .env.example .env
# Edit .env: set GCP_PROJECT, BUCKET, CLICKHOUSE_HOST
# Set PULUMI_CONFIG_PASSPHRASE_FILE to the same passphrase file used in infra/
# DOWNLOAD_VCF_URL is pulled automatically from the variant-pipeline stack output at deploy time

poetry install

# First time only:
poetry run poe login
poetry run poe stack-init

# Build Docker image and deploy:
poetry run poe build
poetry run poe deploy
```

### Run locally

```bash
poetry run uvicorn src.main:app --reload --port 8080
```

### Trigger a pipeline

```bash
# VCF ingest (submits to the running local or deployed service):
poetry run poe trigger -- --individual HG00096 --s3-vcf-uri s3://bucket/HG00096.vcf.gz

# ClinVar refresh:
curl -X POST http://localhost:8080/pipelines \
  -H 'Content-Type: application/json' \
  -d '{"type": "clinvar_refresh"}'
```

---

## 5. variant-mcp-server

MCP server that exposes six genomic query tools over Streamable HTTP. The agent-service connects to this to run ClickHouse queries.

**Requires:** infra deployed, ClickHouse loaded with variants and annotations, VPN connected.

### Install and deploy

```bash
cd variant-mcp-server
cp .env.example .env
# Edit .env: set GCP_PROJECT and PULUMI_CONFIG_PASSPHRASE_FILE

poetry install

# First time only:
poetry run poe login
poetry run poe stack-init

# Build Docker image and deploy:
poetry run poe build
poetry run poe deploy
```

### Run locally

```bash
poetry run uvicorn src.server:app --reload --port 8081
```

Health check: `curl http://localhost:8081/health`

### Cost management

The service has `min_instance_count=1` to eliminate cold starts. Scale to zero when not in use:

```bash
poetry run poe mcp-stop   # scale min instances to 0
poetry run poe mcp-start  # restore min instances to 1
```

---

## 6. agent-service

FastAPI service that runs a Claude reasoning loop over the MCP variant server. Accepts natural-language questions and returns answers as a Server-Sent Events stream.

**Requires:** variant-mcp-server deployed and reachable, `anthropic-api-key` secret in Secret Manager (set by `infra poe secrets-push`), VPN connected.

### Install and deploy

```bash
cd agent-service
cp .env.example .env
# Edit .env:
#   GCP_PROJECT=<your-project-id>
#   MCP_SERVER_URL=<Cloud Run URL of variant-mcp-server>
#   PULUMI_CONFIG_PASSPHRASE_FILE=<same passphrase file as infra/>

poetry install

# First time only:
poetry run poe login
poetry run poe stack-init

# Build Docker image and deploy:
poetry run poe build
poetry run poe deploy
```

### Run locally

```bash
# Requires VPN to reach the deployed MCP server
poetry run uvicorn src.main:app --reload --port 8080
```

Health check: `curl http://localhost:8080/health`
Returns `{"status":"ok","tools":<n>}` — the tool count confirms MCP connectivity.

### Query the agent

```bash
poetry run poe ask -- --question "What pathogenic variants does HG002 have in BRCA2?"
```

Or POST directly:

```bash
curl -X POST https://<agent-service-url>/query \
  -H 'Content-Type: application/json' \
  -d '{"question": "What pathogenic variants does HG002 have in BRCA2?"}' \
  --no-buffer
```

The response is an SSE stream. Events: `tool_call`, `tool_result`, `answer`, `error`, `done`.

---

## 7. sample-service

RESTful API for 1000 Genomes sample metadata. Reads from the Firestore `samples` collection populated by `infra`'s `poe load-samples`.

**Requires:** infra deployed (including `poe up` with `sample-service-sa`), `poe load-samples` run, VPN connected.

### Install and deploy

```bash
cd sample-service
cp .env.example .env
# Edit .env: set GCP_PROJECT and PULUMI_CONFIG_PASSPHRASE_FILE

poetry install

# First time only:
poetry run poe login
poetry run poe stack-init

# Build Docker image and deploy:
poetry run poe build
poetry run poe deploy
```

### Run locally

```bash
poetry run uvicorn src.main:app --reload --port 8080
```

Requires Application Default Credentials (`gcloud auth application-default login`).

Health check: `curl http://localhost:8080/health`

---

## 8. vap-ui

Browser-based internal tool — pipeline management, agent query, cohort dashboard, and sample browser. Proxies requests to workflow-service, agent-service, variant-mcp-server, and sample-service via Next.js App Router API routes (reads env vars at request time).

**Requires:** infra deployed (including `poe up` with the new `vap-ui-sa` service account), all upstream services deployed, VPN connected.

### One-time infra update

The `vap-ui-sa` service account was added to `infra/iam.py`. If infra was deployed before this repo existed, re-apply it:

```bash
cd infra && poetry run poe up
```

### Install and deploy

```bash
cd vap-ui
cp .env.example .env
# Edit .env: set GCP_PROJECT and PULUMI_CONFIG_PASSPHRASE_FILE
```

`.env` for deploy (no local dev values needed for deployment):

```
GCP_PROJECT=variant-processing
PULUMI_CONFIG_PASSPHRASE_FILE=/path/to/passphrase-file
```

```bash
poetry install

# First time only:
poetry run poe login
poetry run poe stack-init

# Build Docker image and deploy:
poetry run poe build
poetry run poe deploy
```

### Run locally

Requires the upstream services to be running (or on VPN pointing at deployed services). Set `.env.local` with the upstream URLs, then:

```bash
npm install
npm run dev
```

Open `http://localhost:3000`. The Next.js dev server proxies `/api/workflow/*`, `/api/agent/*`, and `/api/mcp/*` to the configured upstream URLs.

### View logs

```bash
poetry run poe logs
```

