# @pipeworx/ares-cz

Czech business register (ARES) — look up any Czech company or sole trader by
IČO or by name, from the Ministry of Finance's official register of economic
subjects.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `ares_lookup_ico({ ico })` — one economic subject by its 8-digit IČO:
  registered trading name, legal form, seat address, founding date, dissolution
  date, DIČ (VAT) number, CZ-NACE activity codes.
- `ares_search_name({ name, limit, offset, onlyActive })` — subjects whose
  trading name matches, with the total match count so a precise hit is
  distinguishable from a common word.

## Auth

Keyless. No registration, no header, no quota documented.

## Data sources

- <https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ico}> — one subject by IČO.
- <https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/vyhledat> — POST search, returns `pocetCelkem` (total) plus a page of subjects.

ARES is the Ministry of Finance's consolidated view over the commercial
register (OR), the trade register (RŽP), the statistical register (RES) and the
VAT payer register — so one IČO answers questions that would otherwise need
three lookups.

## Traps

- **IČO must be zero-padded to 8 digits on the REST path.** "177041" 404s,
  "00177041" resolves; the pack pads for you.
- **A dissolved company still resolves.** `active` is derived from whether
  `datumZaniku` is present, *not* from the lookup succeeding — treating a 200 as
  "still trading" is wrong.
- **`pocet` (page size) caps at 200** and a larger value is a 400, so the pack
  clamps instead of passing through.
- Field names in `raw` stay Czech on purpose; the mapped fields are renamed to
  English so a caller does not need to know `obchodniJmeno` is the trading name.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ares-cz": {
      "url": "https://gateway.pipeworx.io/ares-cz/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ares-cz/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/ares_lookup_ico \
  -H 'Content-Type: application/json' \
  -d '{"ico":"00177041"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ares_lookup_ico`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ares-cz": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ares-cz"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ares-cz
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ares Cz data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
