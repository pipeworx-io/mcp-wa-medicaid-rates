# @pipeworx/wa-medicaid-rates

Washington Apple Health (Medicaid) FQHC (federally qualified health center) and rural health clinic (RHC) encounter rates, per clinic: the per-encounter rate, the date it started, and for FQHCs the category of service (medical, dental, mental health, substance use disorder, maternity support). Also the managed-care enhancement rates HCA pays per enrolled client. From the Washington State Health Care Authority (HCA).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 2034+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `wa_clinic_rates(program?, provider?, npi?, category?, period?, rate_effective_after?, limit?)` — encounter rates. FQHC rows are per clinic location and category of service; RHC rows are per clinic NPI. Filter by clinic or health-centre name fragment, 10-digit NPI, category, program, quarter, or rate start date. An unmatched filter returns the nearest names or the valid categories and periods, never a bare empty.
- `wa_enhancement_rates(provider?, npi?, type?, period?, limit?)` — the monthly managed-care enhancement amount per NPI and type (medical or mental health), with HCA's own rate-change date or note.
- `wa_rate_summary(program?, period?, group_by?)` — which quarterly publications are held and when HCA updated each, rows and NPIs, min/median/max/mean encounter rate, optionally grouped by category, rate start date or quarter.

## Auth

Keyless.

## Data sources

- Washington State Health Care Authority (HCA), Apple Health FQHC encounter rates, RHC encounter rates, and FQHC and RHC enhancement rates — the quarterly rate workbooks HCA lists on its billing guides and fee schedules page. Public, no account.

Things worth knowing:

- **Grain.** FQHC: one row per NPI + clinic location + category of service. RHC: one row per clinic NPI (an NPI can carry two rows, such as medical and dental). Rates are US dollars per encounter, not per procedure code; HCA's fee schedules by procedure code are a different publication and are not in this pack.
- **Each row has its own start date** (`rate_effective`). It can be earlier or later than the quarter of the workbook it appears in: most 2026 rows start 2026-01-01, while some RHC rows carry October 2026 start dates and some FQHC rows start in January or May. The quarter (`period`) is the start of the HCA workbook that carries the row; `data_as_of` is the start of the newest quarter.
- **Quarters.** The April, July and October 2026 workbooks are all held, so a rate can be compared across quarters (`period: "all"`). The newest quarter is the default.
- **Enhancement rates** are, per HCA's FQHC billing guide, the monthly amount HCA pays an FQHC for each client enrolled with a managed care organization, on top of the plan's payment, to make up the difference to the encounter rate. HCA's workbook covers about 40 clinics, FQHC and RHC together, and does not label which program a row belongs to. They are not encounter rates.
- RHC rows carry HCA's `comment` ("CIS Rate Update", "New Clinic", "Closed Clinic") when present, and `mco_marker`, which is true when HCA marked the "RHC encounter rate through MCOs" column.
- HCA lists one FQHC clinic-category line twice in each workbook; the duplicate is collapsed and counted in the publication entries (`duplicate_lines_removed`).
- **Organisations only.** Any row whose provider name reads as a person is left out and counted in `individual_rows_dropped` (none in the 2026 workbooks). The rule is in `src/individuals.ts`; when in doubt a row is kept.
- NPIs are strings. Names are as HCA prints them (legal entity or health centre, often with the site).
- Fee-for-service rates only: managed-care plans may pay under their own contracts.
- Every response carries `data_as_of`, `captured_at` and the source.

## Stored data and refresh

HCA publishes these as workbooks with no query interface, so every quarter HCA lists is read by `scripts/bake-index.mjs` into `src/wa-rates-data.ts` (about 2,250 rate rows and 125 enhancement rows, 670 KB), registered in `workers/gateway/src/pack-baked-indexes.json`; the deploy uploads it to KV and the gateway injects it on the first call into this pack, so nothing is evaluated at module scope. Refresh with `node mcps/wa-medicaid-rates/scripts/bake-index.mjs`: it reads HCA's page for the workbooks it links, then reads each one, and rewrites the module in place. It refuses a changed header, a workbook below its row floor, an unreadable date, a non-positive rate, or a newest quarter missing a dataset. HCA posts a new set each quarter, so the data is a quarter old at worst; `data_as_of` says which.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "wa-medicaid-rates": {
      "url": "https://gateway.pipeworx.io/wa-medicaid-rates/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/wa-medicaid-rates/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 2034+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/wa_clinic_rates \
  -H 'Content-Type: application/json' \
  -d '{"program":"FQHC","provider":"Sea Mar","category":"dental","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/wa_clinic_rates`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-wa-medicaid-rates` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/wa-medicaid-rates/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "wa-medicaid-rates": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-wa-medicaid-rates"]
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
ask_pipeworx({ question: "your question about Wa Medicaid Rates data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
