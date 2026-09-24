# @pipeworx/ferc-elibrary

FERC eLibrary MCP — search the Federal Energy Regulatory Commission's public docket and document record (elibrary.ferc.gov). Keyless JSON API, no bot wall.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `ferc_docket(docket, page?, limit?)` — all filings/orders for a docket number. Matches the docket *family*, so results can include related sub-dockets.
- `ferc_search(query?, since?, until?, full_text?, page?, limit?)` — full-text search across all dockets, optionally bounded to a filed-date window.
- `ferc_class_types(search?)` — browse the document class/type/library vocabulary used in `category` and `class_types` fields.

## Auth

Keyless.

## Data sources

- `POST https://elibrary.ferc.gov/eLibraryWebAPI/api/Search/AdvancedSearch` — undocumented (FERC publishes no API docs for this endpoint); the request/response shape was reconstructed from the Angular SPA's bundled JS and live-verified (see `docs/ferc-elibrary-api.md`).
- `GET https://elibrary.ferc.gov/eLibraryWebAPI/api/Search/GetClassTypes` — keyless, backs `ferc_class_types`.

## Quirks

- **`dateType` is snake_case** (`filed_date` / `created_date` / `posted_date`), not the UI label ("Filed"). Sending the wrong value is a **silent** failure: HTTP 200, `success:true`, `errorMessage:null`, `totalHits:0` — indistinguishable from a genuine no-hits query. This pack always sends `filed_date` in snake_case; `ferc_search` echoes the applied window back so a caller can tell the filter actually took effect.
- Upstream's own field name for accession number has a typo — **`acesssionNumber`** (three s's) — kept as-is on the wire, normalized to `accession_number` in this pack's output.
- Zero hits (or `success:false`) returns `{found:false, reason, hint}` from both `ferc_docket` and `ferc_search` rather than a bare empty array — a bare empty list would be indistinguishable from an upstream fault or a `dateType` mistake.
- FERC's own Cloudflare edge intermittently 522s (transient origin flakiness, not a rate limit or a block — confirmed not reproducible in 13 consecutive attempts after the first). This pack retries once with a short backoff before surfacing an `upstream_down:` error.
- Document *file* download (`GetDocInfoByAccessionNo`, `GetDocInfo`, `GetFileListByAccession`) is out of scope — all three 404/400 on the shapes tried during the probe, and `AdvancedSearch` already returns everything these tools need (accession numbers, dates, categories, docket families).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ferc-elibrary": {
      "url": "https://gateway.pipeworx.io/ferc-elibrary/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ferc-elibrary/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ferc_docket \
  -H 'Content-Type: application/json' \
  -d '{"docket":"ER11-4046","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ferc_docket`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ferc-elibrary": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ferc-elibrary"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ferc-elibrary
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ferc Elibrary data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
