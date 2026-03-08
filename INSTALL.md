# Installation

> **Work in progress.** These instructions cover the repos that are currently implemented. Additional services (MCP server, pipeline API, agent, web UI) are not yet deployed.

Deploy in order: **infra → variant-pipeline → clinvar-pipeline**. Each repo depends on the one before it.

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
poetry install
poetry run poe bootstrap   # creates GCS state bucket, inits the dev stack — run once

poetry run poe preview     # review what will be created
poetry run poe up          # apply
```

> After the first `poe up`, wait 1–2 minutes for GCP API activation to propagate before running it again.

### Post-deploy

```bash
poetry run poe secrets-push    # push secret values from .env into Secret Manager
poetry run poe schema-apply    # create the variants and annotations tables in ClickHouse
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
DNS = 8.8.8.8

[Peer]
PublicKey = <server public key — from `poe vpn-status`>
Endpoint = <vpn_gateway_external_ip — from `poe outputs`>:51820
AllowedIPs = 10.128.0.0/20, 10.10.0.0/24
PersistentKeepalive = 25
```

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

```bash
poetry run poe trigger -- \
  --bucket genomic-variant-prototype-variant-processing \
  --clickhouse-host 10.128.0.3
```

Add `--force-download`, `--force-load`, or `--force-enrich` to re-run individual steps independently.
