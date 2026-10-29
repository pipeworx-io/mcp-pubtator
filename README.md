# @pipeworx/pubtator

Biomedical literature search by entity, plus the gene–disease–chemical–variant
relations PubTator 3 has extracted from ~36M PubMed abstracts and PMC
open-access full text — every relation labelled machine-extracted and carrying
the sentences it was read from. Sourced from the PubTator 3 API run by NCBI /
NLM. Connects gene-level lookups (mygene-info, pubmed) to the literature that
mentions them.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `pubtator_find_entity(query, concept?, limit?)` — resolve a gene, disease,
  chemical, variant, species or cell-line name to its normalized PubTator id
  (`@GENE_BRCA1` / NCBI Gene 672, `@DISEASE_Breast_Neoplasms` / MeSH D001943).
  Answers "what is the id for X so I can query by it".
- `pubtator_search(query, page?)` — literature search by free text, by entity
  ids joined with `AND`/`OR`, or by a relation query
  (`relations:treat|@CHEMICAL_Doxorubicin|@DISEASE_Breast_Neoplasms`). Each hit
  carries the matched sentence with its entity mentions decoded (`text`,
  `entity`, normalized `ids`, `matched_query`), PMID, PMCID, DOI, journal and
  date. 10 per page; `total_results` and `total_pages` returned.
- `pubtator_relations(entity, type?, target?, limit?, evidence_for?)` — the
  relations PubTator 3 extracted for an entity (which drugs treat a disease,
  which genes a disease is associated with …), ranked by
  `supporting_publications`. The top `evidence_for` relations (default 3, max 5)
  carry up to 3 supporting passages each. Accepts a PubTator id or a plain name
  (resolved via autocomplete; the match is reported in `resolved_from`).
- `pubtator_relation_evidence(type, entity1, entity2, page?)` — every passage
  supporting one specific relation, 10 per page, with PMIDs. Use it to audit a
  relation `pubtator_relations` returned.
- `pubtator_annotations(pmids, full_text?)` — entity annotations for up to 20
  PMIDs: every mention in title + abstract (or PMC full text when
  `full_text: true`), normalized to NCBI Gene / MeSH / dbSNP / Taxonomy with
  character offsets, plus the relations extracted within each article.

## Auth

Keyless.

## Data sources

- <https://www.ncbi.nlm.nih.gov/research/pubtator3-api/entity/autocomplete/?query=BRCA1&concept=gene&limit=10>
  — entity name → `@TYPE_Name` id, with the backing database id.
- <https://www.ncbi.nlm.nih.gov/research/pubtator3-api/search/?text=...&page=1>
  — article search; `text_hl` is the matched sentence in PubTator's inline
  entity encoding, decoded by this pack.
- <https://www.ncbi.nlm.nih.gov/research/pubtator3-api/relations?e1=@GENE_BRCA1&type=associate&e2=...>
  — extracted relations with supporting-publication counts.
- <https://www.ncbi.nlm.nih.gov/research/pubtator3-api/publications/export/biocjson?pmids=31022191&full=true>
  — BioC JSON with every annotation and in-article relation.

Things the next person would otherwise rediscover:

- **Entity ids are name-based and case-sensitive.** The API wants
  `@GENE_BRCA1`, `@DISEASE_Breast_Neoplasms`, `@CHEMICAL_Doxorubicin` — not
  `@GENE_672` or `@DISEASE_MESH_D001943`. A search for the database-id form
  returns HTTP 200 with `count: 0` and no error, which is why every entity
  argument in this pack goes through autocomplete when it does not start with
  `@`.
- **Everything is a text-mining prediction.** PubTator 3's relations come from
  a neural relation-extraction model and its annotations from entity
  recognizers (GNormPlus, TaggerOne, tmVar…). Nothing is human-curated. Each
  relation and the annotations envelope carry an `extraction` field saying so;
  `supporting_publications` is the model's evidence count, not a curation
  status.
- **Relation queries ignore entity order.** `relations:associate|A|B` and
  `relations:associate|B|A` return the same count. Twelve relation types:
  associate, cause, compare, cotreat, drug_interact, inhibit, interact,
  negative_correlate, positive_correlate, prevent, stimulate, treat.
- **`text_hl` encoding.** `@<m>GENE_BRCA1</m> @GENE_672 @@@BRCA1@@@-mutated`
  means: id tokens (query matches wrapped in `<m>`), then the surface mention
  wrapped in `@@@`. An id token is `@` *not* followed by `@@` — the first cut of
  the decoder treated `@@@BRCA@@@` as an id and merged two mentions into one.
- **Autocomplete concepts.** `gene`, `disease`, `chemical` and `variant` are
  dense; `species` and `cellline` return `[]` for common names ("mouse",
  "HeLa"). Drop the concept filter rather than concluding the entity is absent.
- **Full text is large.** `full=true` on a PMC open-access article returns 100+
  passages (119 for PMID 31022191) and the document id becomes the PMC number;
  the PMID is recovered from the first passage's `article-id_pmid`. Passages
  are truncated to 1,500 characters with `truncated: true`.
- **Page size is fixed at 10** by the upstream; there is no `size` parameter.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "pubtator": {
      "url": "https://gateway.pipeworx.io/pubtator/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/pubtator/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/pubtator_find_entity \
  -H 'Content-Type: application/json' \
  -d '{"query":"BRCA1","concept":"gene","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/pubtator_find_entity`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-pubtator` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/pubtator/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "pubtator": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-pubtator"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Pubtator data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
