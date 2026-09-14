# LocationLists MCP server

[![LocationLists MCP server](https://glama.ai/mcp/connectors/io.github.kylehawke-stack/locationlists/badges/score.svg)](https://glama.ai/mcp/connectors/io.github.kylehawke-stack/locationlists)

Listed on [Smithery](https://smithery.ai/servers/kylehawke/LocationLists) · [Glama](https://glama.ai/mcp/connectors/io.github.kylehawke-stack/locationlists) · [Official MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=locationlists) · [x402scan](https://www.x402scan.com/server/f7b05cea-646f-4adc-b5aa-299aba75bbf6)

Business location data for AI agents. Search, sample, count and buy ready-to-use lists of US business locations: 960 datasets compiled from each brand's official store locator or the public register that publishes the data.

Filter by city, state, zip, a radius around any place, or any column in the file. Buy a full CSV by card, or let an agent pay per row or buy the whole file in USDC on Base via x402, with no account or API key. A failed or empty query costs nothing.

This repository holds the registry listing (`server.json`) and client setup notes for the hosted [LocationLists](https://locationlists.com) MCP server. The server runs at locationlists.com; there is nothing to install or run locally.

| | |
|---|---|
| Endpoint | `https://locationlists.com/mcp` |
| Transport | Streamable HTTP, stateless, JSON responses |
| Auth | None |
| Payments | Stripe Checkout (people) or x402, USDC on Base (agents) |
| Registry name | `io.github.kylehawke-stack/locationlists` |
| Read-only profile | `https://locationlists.com/mcp/chatgpt` (search, dataset, sample, count only; no checkout) |

## What is in the catalog

Figures are read from the live [`catalog.json`](https://locationlists.com/catalog.json) (generated 2026-09-14):

- 960 datasets plus 1 bundle, 15,190,621 US business locations in total (overlapping slices counted once)
- Categories: Breakfast, Equipment, Financial, Furniture, Government, Grills, Hardware, Healthcare, Industrial, Mattresses, Nonprofits, Other Transactions, Outdoor Furniture, Retail
- Each dataset is compiled from the official locator, association directory or public register that publishes it
- One-time prices from $9 to $1,299 per dataset, by record count

## Tools

All nine tools are on `/mcp`. Tools carry `readOnlyHint` / `destructiveHint` / `openWorldHint` annotations. `initialize` returns `serverInfo.name = "LocationLists"`.

Free tools:

| Tool | Arguments | What it does |
|---|---|---|
| `search_datasets` | `query?` string, `category?` enum, `limit?` int (≤50) | Find datasets by brand, product line, location type or category. Returns slug, name, record count, coverage and page URL. |
| `get_dataset` | `slug` string | Full record: fields with descriptions, record and state counts, coverage, refresh cadence, the real last-modified date of the file, FAQs, sample URL and page URL. |
| `get_sample` | `slug` string, `rows?` int (≤10) | Up to 10 real rows from the live file, spread across the dataset, as JSON plus CSV text. |
| `count_locations` | `dataset` string, `state?`, `city?`, `county?`, `zip?`, `where?` array | How many rows match a filter on geography and any other column (e.g. `revenue_amt gt 2000000`), and what fetching them would cost. |
| `get_quote` | `slugs` string[] | Line-item prices and total for one or more datasets; points out a bundle if it covers several requested brands for less. |

Card checkout (Stripe):

| Tool | Arguments | What it does |
|---|---|---|
| `create_checkout` | `slug` string, `email?` string | Opens a Stripe Checkout session for one dataset and returns the payment URL and session id. Does not charge anything by itself. |
| `check_order` | `sessionId` string | Reports whether a Checkout session is paid and, if so, returns the permanent download link. |

Agent payments (x402, USDC on Base):

| Tool | Arguments | What it does |
|---|---|---|
| `query_locations` | `dataset` string, `state?`, `city?`, `county?`, `zip?`, `where?` array, `order_by?` object, `offset?` int, `limit?` int (≤100) | Rows from one dataset filtered on any column, sorted and paged. Priced per row and settled only after the rows are produced. Datasets under 5,000 records are not sold by the row. |
| `buy_dataset` | `dataset` string | Buy a whole dataset outright and get a permanent download link for the CSV, at the same list price a card buyer pays. |

## Agents can pay with x402

`query_locations` and `buy_dataset` use [x402](https://www.x402.org), the HTTP 402 payment protocol. An agent that holds USDC on Base can buy data with no account, API key or checkout page:

1. Call the tool without payment. The server answers with the exact price and x402 payment requirements (USDC on Base mainnet, settled through the Coinbase CDP facilitator).
2. Sign the payment with the agent's wallet and call the tool again with the payment attached.
3. The server returns the rows (`query_locations`) or the permanent download link (`buy_dataset`). Per-row queries settle only after the rows are produced, so a failed query costs nothing.

Pricing rules the agent can rely on:

- `count_locations` is free and reports how many rows a filter matches and what they would cost, before any payment.
- Per-row prices are set per dataset at roughly twice the list price spread over its record count, so a small slice of a large file costs cents. You pay for the `limit` you request.
- Past a few hundred rows, `buy_dataset` is cheaper than assembling the file row by row, and it is complete.

Clients without a wallet can use `get_quote` → `create_checkout` → `check_order` and pay by card instead.

## Client setup

### Claude Code

```bash
claude mcp add --transport http locationlists https://locationlists.com/mcp
```

### Cursor (`~/.cursor/mcp.json`)

```json
{
  "mcpServers": {
    "locationlists": {
      "url": "https://locationlists.com/mcp"
    }
  }
}
```

### VS Code (`.vscode/mcp.json`)

```json
{
  "servers": {
    "locationlists": {
      "type": "http",
      "url": "https://locationlists.com/mcp"
    }
  }
}
```

### Claude.ai and ChatGPT

- Claude.ai: Settings → Connectors → Add custom connector → URL `https://locationlists.com/mcp`, no OAuth.
- ChatGPT: Settings → Connectors (developer mode) → add `https://locationlists.com/mcp/chatgpt`, no auth. The read-only profile is used there because in-chat purchase of digital goods is not permitted in ChatGPT.

### Any MCP client, by hand

```bash
curl -s https://locationlists.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_datasets","arguments":{"query":"bobcat"}}}'
```

A quick path through the tools: `search_datasets {query:"bobcat"}` → `get_sample {slug:"bobcat-dealers"}` → `count_locations {dataset:"bobcat-dealers", state:"TX"}` → `get_quote {slugs:["bobcat-dealers"]}` → `create_checkout {slug:"bobcat-dealers"}` → `check_order {sessionId}`.

## Plain HTTP endpoints (no MCP client needed)

| URL | What |
|---|---|
| [`/llms.txt`](https://locationlists.com/llms.txt) | Curated map of the site for assistants (llmstxt.org) |
| [`/llms-full.txt`](https://locationlists.com/llms-full.txt) | Every dataset with fields, counts, refresh and price |
| [`/catalog.json`](https://locationlists.com/catalog.json) | The same catalog as JSON |
| `/data/{slug}/dataset.json` | One dataset in full, including the real last-modified date |
| `/data/{slug}/sample.csv`, `/data/{slug}/sample.json` | 10 real rows, no signup |
| `/data/{slug}/buy?src=<name>` | Redirects to Stripe Checkout for that dataset |

Example: [bobcat-dealers/dataset.json](https://locationlists.com/data/bobcat-dealers/dataset.json), [bobcat-dealers/sample.csv](https://locationlists.com/data/bobcat-dealers/sample.csv).

## Links

- Site: <https://locationlists.com>
- Privacy policy: <https://locationlists.com/privacy>
- Data license and terms: <https://locationlists.com/license>
- Glama: <https://glama.ai/mcp/connectors/io.github.kylehawke-stack/locationlists>
- Contact: kyle@locationlists.com

## License

The files in this repository (README, `server.json`) are MIT licensed; see [LICENSE](LICENSE). The datasets sold through the server are not covered by that license; they are governed by <https://locationlists.com/license>.
