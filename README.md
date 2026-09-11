# LocationLists MCP server

Ready-to-use CSV datasets of US business locations — dealer networks, licensed contractors, retail and restaurant chains — search, preview real rows, and buy in chat.

This repository holds the registry listing (`server.json`) and client setup notes for the hosted [LocationLists](https://locationlists.com) MCP server. The server itself runs at locationlists.com; there is nothing to install or run locally.

| | |
|---|---|
| Endpoint | `https://locationlists.com/mcp` |
| Transport | Streamable HTTP, stateless, JSON responses |
| Auth | None |
| Registry name | `io.github.kylehawke-stack/locationlists` |
| Read-only profile | `https://locationlists.com/mcp/chatgpt` (search, dataset, sample only — no prices or checkout) |

## What is in the catalog

Figures are read from the live [`catalog.json`](https://locationlists.com/catalog.json) (generated 2026-09-11):

- 725 datasets plus 1 bundle, 14,248,853 US business locations in total
- Categories: Healthcare, Retail, Industrial, Nonprofits, Equipment, Furniture, Breakfast, Hardware, Grills, Mattresses, Outdoor Furniture
- Each dataset is compiled from the official locator, association directory or state license register that publishes it
- Prices are one-time, $9–$199 per dataset by record count; delivery is an emailed download link after Stripe payment

## Tools

Core tools (available on `/mcp`; the first three are also on `/mcp/chatgpt`):

| Tool | Arguments | What it does |
|---|---|---|
| `search_datasets` | `query?` string, `category?` enum, `limit?` int (1–50, default 15) | Find datasets by brand, product line, location type or category. Returns slug, name, record count, coverage and page URL. Read-only. |
| `get_dataset` | `slug` string | Full record: fields with descriptions, record and state counts, coverage, refresh cadence, the real last-modified date of the file, FAQs, sample URL and page URL. |
| `get_sample` | `slug` string, `rows?` int (≤10) | Up to 10 real rows from the live file, spread across the dataset, as JSON plus CSV text. |
| `get_quote` | `slugs` string[] | Line-item prices and total for one or more datasets; points out a bundle if it covers several requested brands for less. |
| `create_checkout` | `slug` string, `email?` string | Opens a Stripe Checkout session for one dataset and returns the payment URL and session id. Does not charge anything by itself. |
| `check_order` | `sessionId` string | Reports whether a Checkout session is paid and, if so, returns the permanent download link. |

Agent-payment tools (x402, USDC on Base) for agents that carry a wallet:

| Tool | Arguments | What it does |
|---|---|---|
| `query_locations` | `dataset` string, `state?`, `city?`, `county?`, `zip?`, `limit?` int (≤100) | Rows from one dataset filtered on exact column values, priced per row and settled only after the rows are produced. Call without a payment header first to get a quote. Datasets under 5,000 records are not sold by the row. |
| `buy_dataset` | `dataset` string | Buy a whole dataset outright and get the permanent download link, at the same list price a card buyer pays. |

Tools carry `readOnlyHint` / `destructiveHint` annotations. `initialize` returns `serverInfo.name = "LocationLists"`.

## Client setup

### Claude Code

```bash
claude mcp add --transport http locationlists https://locationlists.com/mcp
```

### Cursor / VS Code (`mcp.json`)

```json
{
  "mcpServers": {
    "locationlists": {
      "type": "http",
      "url": "https://locationlists.com/mcp"
    }
  }
}
```

VS Code uses the key `servers` instead of `mcpServers` in `.vscode/mcp.json`; the entry is otherwise the same.

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

A quick path through the tools: `search_datasets {query:"bobcat"}` → `get_sample {slug:"bobcat-dealers"}` → `get_quote {slugs:["bobcat-dealers"]}` → `create_checkout {slug:"bobcat-dealers"}` → `check_order {sessionId}`.

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
- Contact: kyle@locationlists.com

## License

The files in this repository (README, `server.json`) are MIT licensed — see [LICENSE](LICENSE). The datasets sold through the server are not covered by that license; they are governed by <https://locationlists.com/license>.
