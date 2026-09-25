# @pipeworx/crtsh

Certificate Transparency log search from crt.sh — every TLS certificate ever
issued for a domain and its subdomains, with issuer, validity window, serial
number and the full SAN list.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `crtsh_search_domain(domain, include_subdomains?, exclude_expired?, limit?)` —
  every logged certificate for a domain, de-duplicated by certificate id and
  sorted newest-first, plus `unique_names`: every distinct hostname seen across
  all of them. That list is the subdomain-enumeration answer most callers
  actually want.
- `crtsh_certificate(id?, serial?, include_pem?)` — one certificate by crt.sh id
  or by hex serial number. Returns the PEM body; a lookup by `serial` also
  carries the logged metadata, a lookup by `id` returns the PEM alone (see
  Traps).

## Auth

Keyless.

## Data sources

- <https://crt.sh/?q=%25.example.com&output=json> — certificate search by
  identity (domain, SAN, or `%`-wildcard).
- <https://crt.sh/?serial=...&output=json> — certificate search by hex serial.
- <https://crt.sh/?d=<id>> — the PEM body of one certificate.

crt.sh is operated by Sectigo and is a JSON view over a PostgreSQL database of
the public Certificate Transparency logs.

## Traps

- **crt.sh is slow.** A wildcard query on a large domain routinely takes 10-20s
  and the server enforces no query timeout of its own. Every call in this pack
  is bound at 25s rather than the pack default.
- **`output=json` is honoured only for identity-style queries** — `q=`,
  `serial=`, `Identity=`. `?id=<n>&output=json` answers the literal string
  `Unsupported output type: json` with **HTTP 200**. Parsed without checking,
  that is a silent zero; `crtJson()` rejects any body that does not start with
  `[` or `{`.
- **`?d=<id>` and `?id=<id>` are different parameters.** `d` returns the PEM;
  `id` renders an HTML page. This is why `crtsh_certificate` can return the PEM
  for an id but not its metadata — there is no JSON-by-id endpoint, so metadata
  arrives only on the `serial=` path.
- **`name_value` packs every SAN into one newline-separated field.** Splitting
  it is the difference between reporting 8 certificates and the 40 hostnames the
  caller asked for.
- **One certificate appears as several rows** — crt.sh returns one row per
  (certificate, matching identity), so a cert with six matching SANs is six
  rows with the same `id`. De-duplicate by `id` or every count is inflated.
- **A precertificate and its final certificate are two CT entries** with two
  crt.sh ids and the *same* serial number, so an id count roughly doubles the
  number of certificates actually issued. `crtsh_search_domain` reports both:
  `total_log_entries_found` (distinct crt.sh ids) and
  `unique_certificates_found` (distinct serials). Quote the second one when
  answering "how many certificates".
- **CT covers publicly trusted CAs only.** A host behind an internal or private
  CA appears nowhere here; an empty result is not proof the host does not exist.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "crtsh": {
      "url": "https://gateway.pipeworx.io/crtsh/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/crtsh/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/crtsh_search_domain \
  -H 'Content-Type: application/json' \
  -d '{"domain":"pipeworx.io","exclude_expired":true,"limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/crtsh_search_domain`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "crtsh": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-crtsh"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-crtsh
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Crtsh data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
