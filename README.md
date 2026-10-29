# @pipeworx/ares-cz

Czech business register (ARES) — look up any Czech company or sole trader by
IČO or by name, from the Ministry of Finance's official register of economic
subjects.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ares_lookup_ico({ ico })` — one economic subject by its 8-digit IČO:
  registered trading name, legal form, seat address, founding date, dissolution
  date, DIČ (VAT) number, CZ-NACE activity codes.
- `ares_search_name({ name, limit, offset, onlyActive })` — subjects whose
  trading name matches, with the total match count so a precise hit is
  distinguishable from a common word.
- `ares_vat_reliability({ id })` — is a Czech VAT payer flagged as unreliable
  (nespolehlivý plátce DPH)? Takes a DIČ (VAT number, CZ prefix optional) or an
  IČO. Returns an explicit `status`, the date the flag was published, the tax
  office code, the published bank accounts and the register's answer date.

## Auth

Keyless. No registration, no header, no quota documented.

## Data sources

- <https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/{ico}> — one subject by IČO.
- <https://ares.gov.cz/ekonomicke-subjekty-v-be/rest/ekonomicke-subjekty/vyhledat> — POST search, returns `pocetCelkem` (total) plus a page of subjects.

- Czech Financial Administration (Finanční správa), register of unreliable VAT
  payers and published bank accounts — queried live per request, nothing copied.
  Reuse terms: public Czech state data, called as a client; no dataset is mirrored.

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

## VAT reliability: the three answers that must not be confused

- `status: "unreliable"` — flagged; `unreliable_since` is the publication date.
- `status: "reliable"` — a registered VAT payer that is NOT flagged.
- `status: "not_a_vat_payer"` — the number is not in the VAT payer register at
  all (not registered, deregistered, or wrong). **Not a clean bill of health.**
- A malformed id is an error, never a result: the register itself answers
  "not found" to garbage, so the pack validates first.
- Individuals (personal-number DIČ, not 8 digits) get status only; name, address and
  bank accounts are withheld.
- `published_accounts` is the list of accounts the payer registered for VAT; paying
  elsewhere can make the buyer liable for the VAT.

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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ares_lookup_ico \
  -H 'Content-Type: application/json' \
  -d '{"ico":"00177041"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ares_lookup_ico`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-ares-cz` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/ares-cz/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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
ask_pipeworx({ question: "your question about Ares Cz data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
