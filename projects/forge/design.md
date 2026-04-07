# Forge — Project Design Doc

*Status: Draft — Step 0 Ideation*
*Created: March 31, 2026*
*Authors: Dan Hannah & Clay*
*Note: Forge was repurposed from the micropayment concept (now "Mint") to the API gateway concept.*

---

## Overview

### What Is This?

Forge is an MCP API gateway — software that lets you register existing business APIs and wraps them in MCP (Model Context Protocol). Connect an AI system to Forge and it gains an intelligence layer on top of your company's existing software stack.

**The name:** A forge is where raw metal is heated and shaped into useful tools. Forge takes raw APIs and shapes them into MCP tools that AI agents can wield.

### The Problem

Most small-to-medium businesses run on 5-15 different software systems:

```
Billing (QuickBooks) ←→ ??? ←→ Purchasing (custom)
       ↕                              ↕
Inventory (Fishbowl) ←→ ??? ←→ Manufacturing (JobBOSS)
       ↕                              ↕
Shipping (ShipStation) ←→ ??? ←→ Engineering (SolidWorks PDM)
```

The `???` is where humans manually copy data between systems, export CSVs, write one-off scripts, or just... don't integrate at all. It's the biggest operational bottleneck in every small business.

Enterprise companies solve this with expensive middleware (MuleSoft, Boomi, Zapier Enterprise). Small businesses can't afford that — they just suffer.

### The Solution

Forge lets you:
1. **Register** existing APIs (REST, GraphQL, SOAP, even database connections)
2. **Describe** what each API does in natural language
3. **Map** API endpoints to MCP tools automatically
4. **Connect** an AI agent to the Forge MCP server
5. **Query** across all your systems in natural language

```
"What's the current inventory level for part #4521,
 and do we have any open purchase orders for it?"
```

The AI agent calls Forge's MCP tools, which call the real APIs, which return real data. No human had to open two different apps, cross-reference, and type a Slack message.

### The Architecture

```
┌──────────────────────────────────────────┐
│              AI Agent (OpenClaw, etc.)    │
│              ↕ MCP Protocol              │
├──────────────────────────────────────────┤
│              FORGE (MCP Server)           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Billing  │ │ Inventory│ │Shipping │   │
│  │ Adapter  │ │ Adapter  │ │ Adapter │   │
│  └────┬─────┘ └────┬─────┘ └────┬────┘   │
├───────┼─────────────┼────────────┼────────┤
│       ↓             ↓            ↓        │
│   QuickBooks    Fishbowl    ShipStation   │
│     API           API          API        │
└──────────────────────────────────────────┘
```

---

## Core Concepts

### API Registration

Register an API by providing:
- **Base URL** and authentication (API key, OAuth, basic auth)
- **OpenAPI/Swagger spec** (if available — auto-generates MCP tools)
- **Manual tool definitions** (for APIs without specs)
- **Natural language descriptions** (helps the AI know WHEN to use each tool)

```yaml
# forge.config.yaml
apis:
  - name: quickbooks
    type: rest
    base_url: https://quickbooks.api.intuit.com/v3
    auth:
      type: oauth2
      client_id: ${QB_CLIENT_ID}
      client_secret: ${QB_CLIENT_SECRET}
    spec: ./specs/quickbooks-openapi.yaml
    description: "Accounting and billing system. Use for invoices, payments, customers, and financial reports."

  - name: fishbowl
    type: rest
    base_url: https://localhost:28192/api
    auth:
      type: bearer
      token: ${FISHBOWL_TOKEN}
    tools:
      - name: get_inventory
        method: GET
        path: /inventory/{partNumber}
        description: "Get current inventory level for a part number"
      - name: search_parts
        method: GET
        path: /parts/search
        params: [query, category]
        description: "Search parts catalog by name or category"
```

### Auto-Generation from OpenAPI

If an API has an OpenAPI/Swagger spec, Forge can auto-generate MCP tools:
- Each endpoint becomes an MCP tool
- Parameters become tool input schemas
- Descriptions are pulled from the spec
- Human can override/curate which endpoints to expose

### Tool Discovery

The AI agent uses MCP's `list_tools` to see all available tools across all registered APIs. Forge presents them with clear names and descriptions so the agent can pick the right tool for the job.

### Cross-System Queries

The real magic: an AI agent can call tools from MULTIPLE systems in a single interaction:

```
Human: "Create an invoice for the parts we shipped to Acme Corp last week"

Agent thinks:
1. Call fishbowl.get_shipments(customer="Acme Corp", date_range="last_week")
2. Call fishbowl.get_part_prices(part_numbers=[...from step 1...])
3. Call quickbooks.create_invoice(customer="Acme Corp", line_items=[...])

Result: Invoice created, no human copied any data between systems.
```

---

## Why This Is Huge

### The Market

- ~33 million small businesses in the US alone
- Average small business uses 8-12 different SaaS tools
- Integration is consistently cited as a top 3 pain point
- Current solutions (MuleSoft, Boomi) start at $50K+/year
- Zapier helps but it's trigger-based, not query-based — you can't ASK it questions

### The Timing

- MCP is becoming the universal agent ↔ tool protocol
- AI agents are going mainstream (OpenClaw, Claude, GPT agents)
- Every SaaS tool is racing to add API access
- But nobody is building the BRIDGE between "every tool has an API" and "an AI can use them all"

### The Moat

- Network effects: more API adapters = more value for every user
- Community adapters: users contribute adapters for their specific tools
- Data flywheel: understanding which tools are used together improves recommendations
- MCP-native: built for the emerging standard, not retrofitted

---

## GMPPU as First Customer

Dan's own workplace is the perfect pilot:
- Multiple disconnected systems (Databricks, Jira, Confluence, custom tools)
- Data engineering team that understands APIs
- AI-forward culture (already using OpenClaw)
- Real pain point: cross-referencing data across systems

**Pilot scope:** Register 2-3 internal APIs, connect to Max (GMPPU AI), demo cross-system queries to leadership.

---

## Open Questions

### Technical
- How do we handle API rate limits across multiple simultaneous agent requests?
- Authentication management — securely storing and rotating credentials for N different APIs
- Error handling — when one API is down, how does the agent gracefully degrade?
- Caching layer — should Forge cache frequent API responses to reduce load?
- Schema evolution — when an API changes, how do tools update?

### Business
- Pricing model — per-registered-API? Per-query? Flat rate?
- Open source core + commercial add-ons? Or fully commercial?
- How do we compete if MuleSoft/Boomi add MCP support? (Speed + simplicity + price)
- Channel: sell to IT teams? Business owners? AI platform vendors?

### Product
- GUI for API registration or config-file only?
- Pre-built adapters for popular tools (QuickBooks, Shopify, etc.) — community or commercial?
- Monitoring dashboard — which tools are called, error rates, latency?
- Multi-tenant — can one Forge instance serve multiple departments/users?

---

## Ecosystem Fit

```
@claymore-dev/billet   → Document conversion (raw files → markdown)
@claymore-dev/anvil    → Semantic search (markdown → embeddings → search)
@claymore-dev/foundry  → Documentation platform (search + UI + MCP)
@claymore-dev/forge    → API gateway (business APIs → MCP tools)  ← THIS
Mint                   → Monetized MCP marketplace (agent wallets + paid tools)
```

Forge is infrastructure. Mint is the marketplace built on top. Forge makes tools available; Mint makes them monetizable.

---

## Next Steps (When We're Ready)

- [ ] Prototype: register one REST API, generate MCP tools, query via agent
- [ ] Test with a public API (e.g., OpenWeather) as proof of concept
- [ ] Design the adapter interface (how do you plug in a new API type?)
- [ ] Evaluate: OpenAPI auto-generation — how complete is it for real-world specs?
- [ ] Talk to Jack about GMPPU pilot feasibility
- [ ] Research competitors: Rivet, Activepieces, n8n MCP support

---

## Related

- [Mint Design Doc](../mint/design.md) — monetized MCP marketplace (builds on Forge)
- [Anvil Design Doc](../anvil/design.md) — semantic search MCP server
- [Foundry Design Doc](../foundry/design.md) — documentation platform
- [Billet Design Doc](../billet/design.md) — document ingestion pipeline
