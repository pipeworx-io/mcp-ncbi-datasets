# @pipeworx/ncbi-datasets

NCBI Datasets (US National Library of Medicine) — which genome assemblies exist
for an organism and how good they are, where a gene sits and what it is called
in every other database, and the NCBI taxonomy tree with per-node assembly and
gene counts.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `ncbi_genome_reports(taxon?, accession?, reference_only?, assembly_level?, limit?)`
  — assemblies with level, length, contig/scaffold N50, GC%, submitter,
  BioProject and the paired RefSeq/GenBank accession.
- `ncbi_gene_by_symbol(symbols, taxon?)` — Entrez gene id, biotype, chromosome,
  RefSeqGene coordinates, plus HGNC / Ensembl / UniProt / OMIM cross-references
  and every synonym.
- `ncbi_taxonomy(taxons)` — resolve a tax id, scientific name or common name to
  the full lineage, children, rank and live assembly/gene counts.

## Auth

Keyless. NCBI rate-limits anonymous callers by IP at roughly 5 requests/second
across `api.ncbi.nlm.nih.gov`; the pack sends an identifying User-Agent, does
not retry, and surfaces a 429 with that explanation rather than looping.

## Not the same service as `ncbi-eutils`

E-utilities is a generic search/fetch layer over ~40 Entrez databases that
hands back records you must parse. Datasets answers structured questions about
genomes, genes and taxonomy with structured rows. Use this pack for those three;
use `ncbi-eutils` for PubMed, dbSNP and the rest of Entrez.

## Data sources

- <https://api.ncbi.nlm.nih.gov/datasets/v2/genome/taxon/{taxon}/dataset_report>
- <https://api.ncbi.nlm.nih.gov/datasets/v2/genome/accession/{acc}/dataset_report>
- <https://api.ncbi.nlm.nih.gov/datasets/v2/gene/symbol/{symbols}/taxon/{taxon}>
- <https://api.ncbi.nlm.nih.gov/datasets/v2/taxonomy/taxon/{taxons}>

### Things that cost time to rediscover (measured 2026-09-17)

- **Without filters, a genome report is every assembly ever deposited.** Taxon
  562 (E. coli) has 492,559, and page one of two is a GenBank/RefSeq pair of the
  same assembly — technically correct, practically useless.
  `filters.reference_only=true` takes that to 2, so `ncbi_genome_reports`
  defaults to it for a taxon lookup and reports that it did.
- **Every assembly exists twice**, as `GCA_` (GenBank) and `GCF_` (RefSeq),
  cross-linked by `paired_accession`. Counting rows double-counts assemblies.
- **Multi-value paths do not preserve order.** `taxonomy/taxon/9606,10090`
  returns Mus musculus first. Match on the returned `query` / `symbol`, never
  on position.
- **Numeric fields are strings in gene reports** (`"gene_id":"672"`) but numbers
  in taxonomy reports (`tax_id: 9606`). Both are passed through as received.
- **A taxon path accepts a tax id, a scientific name or a common name** —
  `9606`, `Homo sapiens` and `human` all work.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ncbi-datasets": {
      "url": "https://gateway.pipeworx.io/ncbi-datasets/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ncbi-datasets/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ncbi_genome_reports \
  -H 'Content-Type: application/json' \
  -d '{"taxon":"9606","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ncbi_genome_reports`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ncbi-datasets": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ncbi-datasets"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ncbi-datasets
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ncbi Datasets data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
