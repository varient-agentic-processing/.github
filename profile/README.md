# Genomic Variant Platform

A cloud-native genomic variant analysis platform with an agentic natural language query interface. Built on GCP, ClickHouse, and the Model Context Protocol (MCP).

---

## What This Is

This platform ingests whole-genome sequencing VCF files, joins them with ClinVar clinical annotations, stores them in a columnar analytical database optimised for genomic query patterns, and exposes them to an AI agent through a structured MCP server. The result is a system where clinicians and researchers can ask natural language questions about variant data and receive accurate, cited answers without writing queries.

All services run on GCP, all traffic stays inside a private VPC, and the only entry point is a WireGuard VPN gateway.

---

## Architecture

```
  User (laptop)
      │
      │  WireGuard VPN
      ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  GCP VPC (internal only)                                           │
  │                                                                    │
  │  ┌────────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────┐  │
  │  │  Agent API │  │ Pipeline API │  │ Sample Svc  │  │  Web UI  │  │
  │  │ (Cloud Run)│  │ (Cloud Run)  │  │ (Cloud Run) │  │ (Next.js)│  │
  │  └──────┬─────┘  └──────┬───────┘  └──────┬──────┘  └──────────┘  │
  │         │               │                 │                        │
  │  ┌──────▼──────┐   ┌────▼─────────────────▼────────────────────┐  │
  │  │ MCP Service │   │              Firestore                    │◄┐ │
  │  │ (Cloud Run) │   │  pipeline state · samples · dash cache    │ │ │
  │  │   6 tools   │   └───────────────────────────────────────────┘ │ │
  │  └──────┬──────┘                                                  │ │
  │         │ TCP 9000          ┌──────────────────────────────────┐  │ │
  │  ┌──────▼────────────────┐  │       Stats Service              ├──┘ │
  │  │   ClickHouse (GCE)    │◄─┤       (Cloud Run)                │    │
  │  │ variants · annotations│  │  aggregates ClickHouse → cache   │    │
  │  └───────────────────────┘  └──────────────▲───────────────────┘    │
  │              ▲                             │ on completion          │
  │    ┌─────────┴─────────────────────────────┘                        │
  │    │       Cloud Workflows                                          │
  │    │  VCF Ingest  │  ClinVar Refresh                               │
  │    └────────────────────────────────────────────────────────────────┘
  └────────────────────────────────────────────────────────────────────┘
```

**Six Cloud Run services (all `ingress: internal`):**

| Service | Purpose |
|---------|---------|
| **Web UI** | Unified pipeline management, agent chat, cohort dashboard, and sample browser (Next.js) |
| **Agent API** | Natural language genomic queries — Claude reasoning loop, returns cited answers |
| **Pipeline API** | Trigger and monitor VCF ingest and ClinVar refresh pipelines |
| **MCP Service** | Six genomic query tools over Streamable HTTP — the agent's data interface |
| **Sample Service** | 1000 Genomes sample metadata API — search, filter, and browse 2,504 individuals |
| **Stats Service** | Pre-computes cohort dashboard aggregates from ClickHouse into Firestore for instant load |

---

## Repositories

| Repo | Description |
|------|-------------|
| `infra` | Pulumi (Python) — VPC, ClickHouse VM, WireGuard gateway, IAM, secrets, buckets, Artifact Registry, Firestore |
| `variant-pipeline` | Cloud Workflow: download VCF → normalize (Cloud Batch / bcftools) → load to ClickHouse |
| `clinvar-pipeline` | Cloud Workflow: ClinVar monthly refresh → version check → enrich (Cloud Batch) → upsert annotations |
| `variant-mcp-server` | Cloud Run MCP service — six structured genomic query tools over Streamable HTTP |
| `agent-service` | Cloud Run FastAPI — Claude reasoning loop over MCP tools, SSE answer stream |
| `workflow-service` | Cloud Run FastAPI — submit and monitor pipeline runs, Firestore state tracking |
| `sample-service` | Cloud Run FastAPI — 1000 Genomes sample metadata with fuzzy search |
| `vap-ui` | Cloud Run Next.js — pipeline management, agent query, cohort dashboard, sample browser |

---

## Design Rationale

**ClickHouse for analytical genomics** — the variant workload is columnar by nature: per-individual scans of millions of rows, range queries, and cohort aggregations. ClickHouse's MergeTree sort key and vectorised execution are a natural fit. Separate `variants` and `annotations` tables allow ClinVar to be refreshed monthly without touching variant call data.

**MCP as the agent's data interface** — the MCP service exposes six structured tools that the agent uses through multi-hop reasoning. Names, descriptions, and schemas are independent of ClickHouse internals, so the storage layer can evolve without touching the agent.

**Private-by-default networking** — no public Cloud Run endpoints, no external IPs. All services run with `ingress: internal` inside a private VPC. WireGuard is the single ingress point, keeping the attack surface minimal.

**Infrastructure as Python with Pulumi** — every service in this platform is written in Python, and the infrastructure is no different. Pulumi gives real loops, functions, type hints, and pytest for infra tests with no DSL context-switching.

---

## Data Sources

| Source | Use |
|--------|-----|
| [1000 Genomes Project](https://www.internationalgenome.org/) | Prototype variant data — per-individual WGS VCFs, GRCh38 |
| [ClinVar](https://www.ncbi.nlm.nih.gov/clinvar/) | Clinical significance annotations — monthly refresh |
| [GIAB HG002](https://www.nist.gov/programs-projects/genome-bottle) | Validation truth set — known variant ground truth for correctness testing |

All prototype data is public. No patient data is used at any stage.

---

## GCP Services

Cloud Run . Compute Engine (ClickHouse, n2-highmem-8) . Cloud Batch . Cloud Workflows . Cloud Scheduler . Cloud Storage . Firestore . Artifact Registry . Secret Manager . VPN Gateway (WireGuard)

---

## Tech Stack

Python . FastAPI . Next.js . ClickHouse . Anthropic Claude . Model Context Protocol . Pulumi . bcftools . WireGuard

---

## Getting Started

Step-by-step deployment instructions across all repos: [INSTALL.md](../INSTALL.md)

---

## License

[CC BY-NC 4.0](../LICENSE) — © 2025 Ryan Ratcliff. Free for non-commercial use with attribution. Commercial use requires prior written consent.
