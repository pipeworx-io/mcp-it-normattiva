# @pipeworx/it-normattiva

Italian national legislation — search, resolve an exact URN:NIR citation, and
read one article's original-language text as of any date (multivigenza),
with publication (Gazzetta Ufficiale) dates kept separate from emanation and
in-force dates.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1815+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `it_normattiva_search(query?, title?, act_type?, in_force_on?, emanation_date_from?, emanation_date_to?, publication_date_from?, publication_date_to?, limit?, offset?)`
  — search Italian national acts by full-text words, title words, official
  act type (e.g. `"LEGGE"`, `"DECRETO LEGISLATIVO"`, `"DECRETO-LEGGE"`,
  `"COSTITUZIONE"`, `"REGOLAMENTO"`) and/or date range. Every result carries a
  constructed `urn` field (act type, date and number -- e.g. `urn:nir:stato:decreto.legislativo:2003-06-30;196`)
  that chains straight into the two tools below. Free-text `query` matching
  is OR-across-words over the whole corpus, not an exact phrase — narrow with
  `title`/`act_type`/dates for tight results.
- `it_normattiva_get_act(urn, as_of?, original?)` — resolve an exact citation
  (URN:NIR, e.g. `"urn:nir:stato:decreto.legislativo:2003-06-30;196"`, the
  Privacy Code) to its title, act type, emanation date, Gazzetta Ufficiale
  publication date and the opening text (preamble through the start of
  Article 1). `as_of` (YYYY-MM-DD) resolves the version in force on that date
  (multivigenza); `original: true` returns the text as originally enacted.
- `it_normattiva_get_article(urn, article, annex?, as_of?, original?)` — the
  original-language text of one article (e.g. `article: "1"`, `"12-bis"`) of
  an act, as of a given date. `urn` is the bare act citation (no article
  suffix); `annex` selects an article inside an allegato instead of the main
  body. Publication, emanation and in-force-from/until dates all come back as
  separate fields, never folded together.

Both `get_act` and `get_article` return `found: false` with a `message` (and,
if the citation is genuinely ambiguous — most commonly the same act
republished across more than one Gazzetta Ufficiale supplement — a
`candidates` list) rather than an error, when nothing or more than one thing
resolves.

## Auth

Keyless.

## Data sources

- Istituto Poligrafico e Zecca dello Stato (IPZS), Normattiva Open Data —
  Italy's official consolidated-law database (laws, decree-laws, legislative
  decrees, the Constitution, regulations and other national acts), published
  for the Presidenza del Consiglio dei Ministri. Every act is addressable
  article by article via its URN:NIR citation and carries its full
  multivigenza history — the text as amended, plus every historical version
  with its own in-force window.

## Notes

- **Multivigenza, concretely:** the upstream resolves a version by appending
  a selector to the citation (`!vig=<date>` for "in force on this date",
  `@originale` for "as enacted") — this pack exposes those as `as_of` /
  `original` rather than asking a caller to hand-build the suffix.
- **Article 1's text usually includes the full preamble.** Italian decrees
  and laws place the enacting formula and every "VISTO .../VISTA ..." recital
  before "Art. 1" in the same structural block the upstream returns, so
  `it_normattiva_get_article` with `article: "1"` (and `it_normattiva_get_act`
  with no article) can return a long block of preamble text ahead of the
  article itself — that is the source's own structure, not truncation or
  duplication on this pack's part.
- **`act_type` and the `urn` act-type segment are related but not spelled the
  same.** The URN segment joins the official label's words with dots and
  turns a hyphen into a dot too (`"DECRETO LEGISLATIVO"` ->
  `"decreto.legislativo"`, `"DECRETO-LEGGE"` -> `"decreto.legge"`). Build URNs
  from `it_normattiva_search` results rather than by hand where possible.
- Keyless, live proxy. One upstream request per tool call. Nothing from
  Normattiva is baked into the pack or mirrored (the gateway bundle is near
  its ceiling — see `docs/international-legal-scope.md`).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "it-normattiva": {
      "url": "https://gateway.pipeworx.io/it-normattiva/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/it-normattiva/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1815+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/it_normattiva_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"protezione dei dati personali","act_type":"DECRETO LEGISLATIVO","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/it_normattiva_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "it-normattiva": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-it-normattiva"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-it-normattiva
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about It Normattiva data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
