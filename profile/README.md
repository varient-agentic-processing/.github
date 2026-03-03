# Genomic Variant Platform (This a work in progress, the below is simply a plan.)

A cloud-native genomic variant analysis platform with an agentic natural language query interface. Built on GCP, ClickHouse, and the Model Context Protocol (MCP).

---

## What This Is

This platform ingests whole-genome sequencing VCF files, joins them with ClinVar clinical annotations, stores them in a columnar analytical database optimised for genomic query patterns, and exposes them to an AI agent through a structured MCP server. The result is a system where clinicians and researchers can ask natural language questions about variant data and receive accurate, cited answers without writing queries.

---

## Architecture

```
Public Data Sources
  1000 Genomes (GCS/S3)  .  ClinVar (NCBI FTP)  .  GIAB (NIST FTP)
         |
         v
  Cloud Storage (GCS)
  Landing zone for raw VCFs, ClinVar VCF + TSV, reference FASTA
         |
         v
  Cloud Batch  (bcftools -- normalise, split multiallelics, annotate)
         |
         v
  Cloud Run Jobs  (Python loader -- VCF to ClickHouse)
         |
         v
  ClickHouse on Compute Engine
  +-- variants      ORDER BY (individual_id, chromosome, position)
  +-- annotations   ORDER BY (chromosome, position, ref, alt)
         |
         v
  DuckDB  (federation + joined view across both tables)
         |
         v
  MCP Server  (Python . stdio transport . 6 structured tools)
         |
         v
  Agent  (Python . Anthropic API . multi-hop reasoning)
```

**ClinVar monthly refresh:** Cloud Scheduler -> Cloud Functions -> Cloud Run Job -> ClickHouse mutations (annotations table only, no variant reload required)

---

## Key Design Decisions

**ClickHouse over OpenSearch** -- the variant workload is analytical and columnar: per-individual scans of millions of rows, range queries, and cohort aggregations. ClickHouse's MergeTree sort key and vectorised execution handle this an order of magnitude more efficiently than an inverted-index document store. Separate variant and annotation tables allow ClinVar to be updated monthly without touching variant call data.

**MCP as the query interface** -- the MCP server exposes six tools that the agent uses to answer questions through multi-hop reasoning. The tool interface is datastore-agnostic: tool names, descriptions, and argument schemas are independent of ClickHouse internals, so the backend can evolve without changing the agent layer.

**DuckDB as the federation layer** -- sits between MCP and ClickHouse, providing query federation, result transformation, and a clean abstraction boundary.

---

## Components

| Directory | What It Contains |
|---|---|
|  | Cloud Batch job definitions and bcftools normalisation + annotation scripts |
|  | Cloud Run Job: parses annotated VCF, batch inserts into ClickHouse |
|  | Schema DDL, sort key definitions, projections, memory config |
|  | MCP server -- six tools, query builder, result formatter, input validators |
|  | Python agent -- reasoning loop, MCP client, system prompt |
|  | Cloud Functions download job + Cloud Run refresh pipeline |
|  | Terraform for GCP infrastructure (Compute Engine, Cloud Batch, GCS, Firestore) |
|  | Unit tests per MCP tool + validation query log against GIAB HG002 truth set |

---

## MCP Tools

| Tool | Purpose |
|---|---|
|  | Returns all queryable fields, valid enumerated values, example queries. Agent calls this first when uncertain. |
|  | Aggregate variant burden stats for one individual -- totals, pathogenic count, top genes. |
|  | Filtered variant search by individual, gene, position, clinical significance, consequence, allele frequency. |
|  | Cross-individual lookup -- all individuals carrying a variant in a genomic region. |
|  | Cohort-level counts and distributions grouped by a specified field. |
|  | Full ClinVar annotation for a specific variant by rsID or exact coordinates. |

---

## Data Sources

| Source | Use |
|---|---|
| [1000 Genomes Project](https://www.internationalgenome.org/) | Prototype variant data -- per-individual WGS VCFs, GRCh38 |
| [ClinVar](https://www.ncbi.nlm.nih.gov/clinvar/) | Clinical significance annotations -- monthly refresh |
| [GIAB HG002](https://www.nist.gov/programs-projects/genome-bottle) | Validation truth set -- known variant ground truth for correctness testing |

All prototype data is public. No patient data is used at any stage.

---

## GCP Services

Cloud Storage . Compute Engine (n2-highmem-8) . Cloud Batch . Cloud Run Jobs . Cloud Functions . Cloud Scheduler . Firestore . Artifact Registry . Secret Manager

---

## Detailed Documentation

- [](pipeline/README.md) -- VCF normalisation and annotation pipeline
- [](clickhouse/README.md) -- Schema design, sort key rationale, projection strategy
- [](server/README.md) -- MCP server implementation, tool descriptions, safety design
- [](agent/README.md) -- Agent reasoning loop, system prompt, context management
- [](clinvar_refresh/README.md) -- Monthly annotation update pipeline
- [](infra/README.md) -- GCP infrastructure and deployment
- [](tests/README.md) -- Validation approach and GIAB correctness testing

---

## Status

Prototype -- greenfield build against public 1000 Genomes data. Production migration path documented in [](docs/migration_rfc.md).
