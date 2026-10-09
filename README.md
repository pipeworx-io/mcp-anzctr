# @pipeworx/anzctr

Australian New Zealand Clinical Trials Registry (ANZCTR, anzctr.org.au): the
full registered record for one trial by ACTRN. ANZCTR is the WHO
primary registry for Australia and New Zealand, and many of its trials are not
on ClinicalTrials.gov.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1763+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `anzctr_trial({actrn})` — the full record for one trial: titles, status, phase
  and design, health conditions and condition codes, intervention and
  comparator, primary and secondary outcomes, eligibility, target sample size,
  key dates, primary sponsor, funding sources, ethics committees, secondary
  IDs. An ACTRN with no public record returns `found: false`. Every response
  carries `data_as_of` (the time the registry was read).

Use alongside `clinicaltrials` (US registry) and `isrctn` (UK/international).

## Search is not shipped yet

ANZCTR's search (`TrialSearch.aspx`) is an ASP.NET WebForms postback. A working
implementation exists in commit `f1c44beb0` (`anzctr_search`: GET the form for
its `__VIEWSTATE` and session cookie, POST it back with
`ctl00$body$searchTxtBx`, `interventionDescriptionTxtBx`, `healthConditionTxtBx`,
`registryRdBtnLst=ANZCTR` and the `btnSearch` button). On 2026-10-09 it returned
45 psilocybin records from a throwaway Worker on our Cloudflare account, but
every call from the live gateway got a Cloudflare Managed Challenge (HTTP 403,
"I'm Under Attack Mode - The University of Sydney") on `TrialSearch.aspx`, while
the record page answered normally. A laptop is challenged on the search page too. The egress
relay (`supabase/functions/egress-proxy`) does not carry cookies, which the
postback needs. Fleet #2897 carries the follow-up.

## Auth

Keyless.

## Data sources

- <https://www.anzctr.org.au/Trial/Registration/TrialReview.aspx?ACTRN=12626001215314> — the public record page for one trial.

ANZCTR has no JSON API; the record page is HTML with one `<span id="ctl00_body_CX…">` per field
and `ctl00_body_repeater_TX…_ctlNN_CX…` spans for repeated rows (conditions, outcomes, funders).

Things the next person would otherwise rediscover:

- Some ACTRNs carry a trailing letter (`ACTRN12626001117303p`); the record page
  takes the 14 digits alone, and so does the tool.
- The record page is answered from the live gateway and from a laptop; only
  `TrialSearch.aspx` is challenged.
- An unknown ACTRN 302s to `/Notification.aspx`.
- Records include contact persons with names, phone numbers and email
  addresses. The tool does not return them.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "anzctr": {
      "url": "https://gateway.pipeworx.io/anzctr/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/anzctr/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1763+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/anzctr_trial \
  -H 'Content-Type: application/json' \
  -d '{"actrn":"ACTRN12626001215314"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/anzctr_trial`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "anzctr": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-anzctr"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-anzctr
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Anzctr data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
