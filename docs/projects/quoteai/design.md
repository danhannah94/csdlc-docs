# QuoteAI — Project Design Doc

*Status: Draft — Step 0 Refinement*
*Created: April 7, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This?

QuoteAI is an AI-powered quote generation tool built for Brehob, an industrial air equipment company. A salesperson describes what a customer needs in plain English, and QuoteAI generates a professional draft quote — formatted in Brehob's standard template — by searching past quotes, matching products from the catalog, and pulling accurate pricing from structured data.

Input: "Customer needs a 50HP rotary screw compressor, 125PSI, for automotive paint shop, air-cooled"
Output: A complete Brehob-formatted quote with product specs, pricing, terms, and delivery info.

### Why This Exists

**The problem:** Brehob's quoting process relies heavily on tribal knowledge. John Hannah has 22 years of experience and quote history — he knows which products fit which applications, what the margins should be, and how to structure complex multi-item quotes. He's retiring. When he leaves, that knowledge walks out the door.

**Current pain points:**
- **48-hour quote turnaround** — customers wait too long, deals slip
- **Inconsistent formatting** — quotes vary by salesperson, no enforced template
- **Tribal knowledge dependency** — product selection requires deep domain expertise that only a few people have
- **Manual pricing lookups** — salespeople juggle Excel pricing sheets, multiplier tables, and list prices across multiple tabs
- **No institutional memory** — past quotes aren't searchable or reusable

**The opportunity:** Brehob's director is actively evaluating AI solutions. John has warm relationships and a retirement timeline that creates natural urgency. This is a real business with a real budget and a real pain point — not a solution looking for a problem.

### Who Is It For?

**MVP — John Hannah and Brehob's sales team.** The immediate users are 2-5 salespeople at Brehob who generate quotes for industrial compressed air systems, vacuum systems, dryers, filters, and related equipment.

**Post-MVP — Industrial distributors broadly.** Every industrial distributor has the same problem: experienced reps retiring, pricing spread across spreadsheets, inconsistent quoting processes. The architecture is designed to generalize, but v1 is laser-focused on Brehob.

---

## Business Context

### Brehob Company Profile

- **Industry:** Industrial air equipment — compressors, vacuum systems, dryers, filters, controls
- **Brands carried:** Quincy, Zeks, Hankison, Powerex, Dekker, CRP
- **Offices:** Indianapolis, Cincinnati, Elkhart, Fort Wayne, Detroit (Troy, MI)
- **Customer types:** Manufacturing plants, hospitals/medical facilities, government/municipal (DWSD, VA), universities (U of M), steel mills (US Steel), automotive (Ford)
- **Quote complexity range:**
  - Simple: Single product + price (e.g., one compressor model with specs)
  - Complex: Multi-item systems with controls, commissioning, custom scoping (e.g., US Steel — 7 line items, $100K+, CAM Technologies integration)

### Revenue Model

**Phase A — Brehob engagement:**
- Target: $2-5K/month subscription
- Commission structure: 50% of revenue for first 10 customers → 25% (11-25) → 15% ongoing + 10% renewal
- Sales team: Matt Hannah (procurement/supply chain background), John Hannah (warm intros + retirement income stream)

**Phase B — Generalized SaaS (post-validation):**
- Target: Industrial distributors with similar quoting workflows
- MCP integration layer for connecting inventory, pricing, shipping, and quoting subsystems (see Future Vision section)

### Key People

| Person | Role | Relevance |
|--------|------|-----------|
| John Hannah | Brehob sales rep (retiring) | Domain expert, 22 years of quote history, primary user for validation |
| John's Director | Brehob management | Budget holder, actively evaluating AI solutions |
| Dan Hannah | Builder | Architecture, development, business relationship |
| Matt Hannah | Sales | Customer acquisition, has procurement/supply chain background |
| Clay | AI Lead | Design, development, sub-agent orchestration |

---

## Data Analysis

### Source Data (Google Drive)

All source data lives in the shared Google Drive folder: **"QuoteAI - Brehob Quote History"**

#### Document Types Discovered

**1. Product Specification Sheets** (~26 product category folders)

| Folder | Example Products | Format |
|--------|-----------------|--------|
| Quincy Rotary Screw Compressors | QMB10, QMB15, QMB20, QMB25, QMB30, QSI500, QSD-100 Oil Free, QSLP-10/15 | .doc |
| Quincy Reciprocating Compressors | Various models | .doc |
| Quincy Oil-Less Reciprocating | Various models | .doc |
| Zeks Refrigerated Air Dryers | 250HSG and others | .doc |
| Zeks Filters | Various models | .doc |
| Zeks Expandair Flow Controllers | Various models | .doc |
| Zeks Oil Mist Eliminator | Various models | .doc |
| Zeks Regenerative Dryers | Various models | .doc |
| Hankison Filters | Various models | .doc |
| Hankison Refrigerated Air Dryers | Various models | .doc |
| Powerex Oilless Air Compressors | Various models | .doc |
| Powerex Vacuum | Various models | .doc |
| Dekker Vacuum | Various models | .doc |
| Quincy Vacuum Pumps | Rotary models | .doc |
| Medical Compressors | Various models | .doc |
| Quincy NFPA Compliant Medical | Various models | .doc |
| QMOD Desiccant Dryers | Various models | .doc |
| Quincy Engine Driven | Various models | .doc |
| Quincy Steady Pressure Control | Various models | .doc |
| Quincy Recip Climate Control | Various models | .doc |
| Brehob Posi-Drain | Various models | .doc |
| CRP Oil Water Separator | Various models | .doc |
| Quincy Air Filters | Various models | .doc |

Structure (consistent across all):
```
ROTARY SCREW AIR COMPRESSOR

CHARACTERISTICS
Manufacturer:           Quincy
Series / Model:         QMB30 (Base Mount)
Cooling:                Air
Pressure:               [filled per config]
Capacity CFM (ACFM):    [filled per config]
Electric Motor Data:    30HP, 3600 RPM, ODP, 1.15 SF
Voltage:                [filled per config]
Drive System:           Belt
Control System:         Constant Speed / Load - No Load
Standard Enclosure:     Included
Noise Level:            81 dBA @ 1 meter enclosed
Dimensions:             52"L x 41"W x 37"H
Weight:                 1200 lbs.

Model QMB30 as described above    Net $ Each
Delivery:
```

Key observation: Price and some specs are left blank — these are templates filled per-quote, not fixed catalog entries. The AI needs to understand which model fits which application and fill in the right values.

**2. Customer Quotes** (20+ customer folders in "Customer Quotes")

Known customers with quote history:
- US Steel (complex — controls, commissioning, $100K+)
- Ford Brookpark
- BASF
- University of Michigan (multiple: Ann Arbor, North Campus audit, final proposal)
- VA Ann Arbor (vacuum system)
- DWSD Springwells (Detroit water)
- Mahle
- Liebherr
- Cyberlink
- Detroit Receiving
- Slate Trucks
- VCNA Detroit
- Groeb Farms

Plus year-based folders: 2009, 2012, 2014, 2015 Quotes

Structure varies by complexity. Simple quotes have a product spec + price. Complex quotes (US Steel example) include:
- Multiple line items with part numbers and unit prices
- Scope of work and deliverables
- Customer responsibilities
- Terms (delivery, freight, FOB, commissioning schedule)
- Brehob letterhead with all office locations
- Pricing summary table

**3. Pricing Quotation Worksheets** (Excel — the calculator)

Files: `Pricing Quotation Work Sheet-2020.xlsx`, `Pricing quotation Sheet 2019.xlsx`, `Pricing Quotation Sheet 3.9.2017.xlsx`, plus older `.xls` and `.doc` versions back to 2004.

Structure (10 tabs per workbook — "Option-1" through "Option-10"):
```
Header:
  QUOTE#, SALESMAN, CUSTOMER, CONTACT, CODE
  DATE REQUESTED, DATE REQUIRED, VALID FOR, DELIVERY
  BILL TO ADDRESS, SHIP TO ADDRESS

Line Items (per row):
  REF | QTY | DESCRIPTION/PART# | CFM | PSI | HP | V/PH/HZ | COOLED (A/W) |
  LIST EACH | MULTIPLIER | COST EACH | GROSS PROFIT | QUOTE EACH | EXTENDED TOTAL

Footer:
  Special Pricing/Concession Info, FOB, Delivery Time
```

The pricing formula: `LIST PRICE × MULTIPLIER = COST`, then markup for `QUOTE PRICE`. This is the core pricing logic.

Note: Dan mentioned newer pricing sheets exist. The 2020 version is sufficient for MVP; updated sheets can be ingested later.

---

## Architecture

### MVP Architecture

```
┌─────────────────────────────────┐
│  Next.js (localhost:3000)       │
│                                 │
│  ┌───────────────────────────┐  │
│  │  Chat Interface           │  │
│  │  "Need a quote for a      │  │
│  │   50HP rotary screw..."   │  │
│  └─────────────┬─────────────┘  │
│                │                │
│  ┌─────────────▼─────────────┐  │
│  │  Generated Quote          │  │
│  │  (Brehob template format, │  │
│  │   printable/exportable)   │  │
│  └───────────────────────────┘  │
└────────────────┬────────────────┘
                 │
    ┌────────────▼────────────┐
    │  API Route / Server     │
    │  (Next.js API routes)   │
    └──┬──────────────────┬───┘
       │                  │
┌──────▼───────┐  ┌───────▼────────┐
│  Claude API  │  │  Supabase      │
│              │  │                │
│  - Receives  │  │  - Products    │
│    context   │  │    (catalog)   │
│  - Generates │  │  - Pricing     │
│    draft     │  │    (tables)    │
│    quote     │  │  - Past Quotes │
│              │  │    (pgvector)  │
└──────────────┘  └────────────────┘
```

### Why This Stack

| Choice | Reasoning |
|--------|-----------|
| **Next.js** | React frontend + API routes in one project. Dan knows it. Fast to iterate. |
| **Localhost** | Zero deployment overhead during iteration. Supabase is still cloud, so backend is portable when ready to deploy. |
| **Supabase** | Already have a Pro account. Postgres + pgvector + auth + API in one service. No separate vector DB needed. |
| **Claude API** | Best at structured output and following complex formatting instructions. Dan has API access. |
| **Chat interface** | Lets the AI ask clarifying questions. Avoids premature form design before we know what fields matter. |

### Data Model (Supabase)

```sql
-- Product catalog (from spec sheet .doc files)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manufacturer TEXT NOT NULL,        -- 'Quincy', 'Zeks', 'Hankison', etc.
    category TEXT NOT NULL,            -- 'Rotary Screw Compressor', 'Refrigerated Air Dryer', etc.
    series TEXT,                        -- 'QMB', 'QSI', 'QSLP', etc.
    model TEXT NOT NULL,               -- 'QMB30', '250HSG', etc.
    mount_type TEXT,                   -- 'Base Mount', 'Tank Mount', etc.
    cooling TEXT,                      -- 'Air', 'Water'
    hp_range NUMRANGE,                 -- horsepower range this model covers
    cfm_range NUMRANGE,               -- capacity range
    psi_range NUMRANGE,               -- pressure range
    voltage_options TEXT[],            -- ['230/1/60', '460/3/60', etc.]
    drive_system TEXT,                 -- 'Belt', 'Direct', etc.
    control_system TEXT,               -- 'Constant Speed / Load - No Load', etc.
    dimensions TEXT,                   -- '52"L x 41"W x 37"H'
    weight_lbs INTEGER,
    noise_dba NUMERIC,
    refrigerant TEXT,                  -- for dryers: 'R404', 'R134a', etc.
    connections TEXT,                   -- '1-1/2" MPT', etc.
    raw_text TEXT NOT NULL,            -- full original document text
    embedding VECTOR(1536),            -- for semantic search
    source_file TEXT,                  -- Drive file ID for provenance
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Pricing data (from Excel pricing sheets)
CREATE TABLE pricing (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID REFERENCES products(id),
    model TEXT NOT NULL,
    list_price NUMERIC,                -- manufacturer list price
    multiplier NUMERIC,                -- Brehob's cost multiplier
    cost NUMERIC GENERATED ALWAYS AS (list_price * multiplier) STORED,
    default_margin NUMERIC,            -- typical markup percentage
    pricing_year INTEGER,              -- which pricing sheet year
    notes TEXT,
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Past quotes (from customer quote .doc files)
CREATE TABLE past_quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_name TEXT NOT NULL,        -- 'US Steel', 'Ford Brookpark', etc.
    customer_contact TEXT,
    quote_date DATE,
    total_value NUMERIC,
    num_items INTEGER,
    complexity TEXT,                    -- 'simple', 'standard', 'complex'
    summary TEXT,                       -- AI-generated summary of quote scope
    raw_text TEXT NOT NULL,            -- full original document text
    embedding VECTOR(1536),            -- for semantic search
    source_file TEXT,                  -- Drive file ID
    source_folder TEXT,                -- customer folder name
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Quote line items (parsed from past quotes)
CREATE TABLE quote_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id UUID REFERENCES past_quotes(id),
    item_number INTEGER,
    description TEXT NOT NULL,
    quantity INTEGER DEFAULT 1,
    unit_price NUMERIC,
    total_price NUMERIC,
    product_id UUID REFERENCES products(id),  -- linked if identifiable
    notes TEXT
);

-- Index for vector similarity search
CREATE INDEX ON products USING ivfflat (embedding vector_cosine_ops) WITH (lists = 20);
CREATE INDEX ON past_quotes USING ivfflat (embedding vector_cosine_ops) WITH (lists = 20);
```

### Pricing as a Tool (MCP-Ready Design)

**Critical design decision:** The LLM should **never calculate prices.** Pricing follows a deterministic formula (`list × multiplier + margin`) that an LLM will hallucinate. Instead:

1. The LLM identifies which products the customer needs
2. A **pricing function** (API route or MCP tool) receives the product ID + quantity and returns the exact price
3. The LLM assembles the final quote using the returned prices

This keeps the AI doing what it's good at (product selection, language, formatting) and keeps it away from what it's bad at (math). It also means pricing updates are instant — change the number in the database, every future quote is correct.

For MVP, this is a simple API route. Post-MVP, it becomes an MCP tool that any agent can call — part of the larger "AI intranet" vision (see Future Vision).

```typescript
// MVP: API route
// POST /api/pricing
// { productId: "xxx", quantity: 2 }
// Returns: { listPrice: 15000, multiplier: 0.65, cost: 9750, suggestedQuote: 12675, margin: 0.30 }

// Post-MVP: MCP tool
// Tool: get_pricing
// Params: { model: "QMB30", quantity: 2, voltage: "460/3/60" }
// Returns: same structured pricing data
```

---

## MVP Scope

### What's In

1. **Chat interface** — Single text input where salesperson describes customer needs
2. **Product matching** — Semantic search over product catalog to find the right equipment
3. **Past quote retrieval** — Find similar past quotes for context (pricing patterns, common configurations)
4. **Pricing lookup** — Deterministic pricing from structured data (no LLM math)
5. **Quote generation** — Claude generates a formatted draft quote in Brehob's template style
6. **Quote display** — Rendered in the UI, printable/copyable
7. **Brehob branding** — Output looks like a real Brehob quote (letterhead, office locations, terms format)

### What's Out (Post-MVP)

- User authentication / multi-tenancy
- Quote history tracking / saved quotes
- Approval workflows
- Inventory integration
- CRM integration
- Email/PDF export (copy-paste is fine for MVP)
- Multiple pricing tiers / customer-specific pricing
- Real-time pricing sheet sync
- Analytics dashboard
- Mobile UI

### MVP User Flow

```
1. Salesperson opens localhost:3000
2. Types: "Groeb Farms needs a new compressor system.
           200 CFM, 125 PSI, food-grade environment,
           need oilless for contamination requirements.
           460/3/60 power available."
3. QuoteAI:
   a. Searches product catalog → matches Powerex oilless compressors
   b. Searches past quotes → finds similar food/pharma quotes
   c. Calls pricing function → gets current list/multiplier/suggested quote price
   d. Claude generates draft quote in Brehob format
4. Output displays:
   - Brehob-formatted quote with:
     - Customer info
     - Product specs (model, CFM, PSI, HP, voltage, dimensions)
     - Pricing (unit price, extended total)
     - Standard terms (delivery, FOB, freight)
   - Confidence indicators (e.g., "Matched 3 similar past quotes")
   - Suggested options/accessories (dryer, filters if not specified)
5. Salesperson reviews, copies, edits as needed in Word
```

### MVP Success Criteria

- John Hannah can describe a customer need and get a recognizable draft quote in < 30 seconds
- Product selection matches what John would have picked (>80% of the time for common equipment)
- Pricing is accurate to the database (100% — no LLM math)
- Output format is close enough to Brehob's template that minimal editing is needed
- John's reaction: "This is pretty close to what I would have written"

---

## Ingestion Plan

### Phase 1: Product Catalog

1. Download all product spec `.doc` files from Drive (26 category folders)
2. Convert `.doc` → text using `textutil` (macOS native, works well based on testing)
3. Parse structured fields: manufacturer, model, specs, dimensions, weight
4. Generate embeddings via OpenAI `text-embedding-3-small` (1536-dim)
5. Load into `products` table in Supabase

Estimated volume: ~100-200 product spec documents

### Phase 2: Pricing Data

1. Download pricing Excel files (2017, 2019, 2020 versions + older)
2. Parse with `openpyxl` / `SheetJS` — extract list prices and multipliers per model
3. Load into `pricing` table, keyed by model
4. Validate: spot-check prices against known quotes

Note: The Excel formulas reference other cells — we need `data_only=True` mode and may need to manually verify some values. The 2020 sheet had `#VALUE!` errors on calculated fields (empty inputs), but the structure is sound.

### Phase 3: Past Quotes

1. Download customer quote `.doc` files from all customer folders
2. Convert to text, generate embeddings for semantic search
3. Where possible, parse structured data: customer name, line items, prices, dates
4. Load into `past_quotes` and `quote_line_items` tables

Estimated volume: ~50-200 customer quote documents

### Ingestion Script Design

```bash
# One-time ingestion script
quoteai-ingest/
├── download.ts        # Pull files from Google Drive
├── convert.ts         # .doc/.xlsx → text/JSON (textutil + openpyxl)
├── parse-products.ts  # Extract structured fields from product specs
├── parse-pricing.ts   # Extract pricing tables from Excel
├── parse-quotes.ts    # Extract quote data from customer docs
├── embed.ts           # Generate embeddings (OpenAI API)
├── load.ts            # Insert into Supabase
└── validate.ts        # Spot-check data integrity
```

---

## Prompt Engineering

### System Prompt (Draft)

```
You are QuoteAI, a quote generation assistant for Brehob, an industrial air
equipment distributor. You help salespeople create professional quotes.

WORKFLOW:
1. Understand the customer's requirements (application, capacity, pressure,
   power, environment, special requirements)
2. If requirements are ambiguous, ask ONE clarifying question
3. Search the product catalog for matching equipment
4. Search past quotes for similar configurations
5. Use the pricing tool for exact pricing (NEVER calculate prices yourself)
6. Generate a draft quote in Brehob's standard format

PRODUCT KNOWLEDGE:
- Brands: Quincy (compressors), Zeks (dryers, filters, flow controllers),
  Hankison (filters, dryers), Powerex (oilless/medical), Dekker (vacuum)
- Key specs: CFM, PSI, HP, voltage/phase/Hz, cooling (air/water)
- Common pairings: compressor + dryer + filter is a typical system

QUOTE FORMAT:
[Brehob letterhead]
Quote #: [auto-generated]
Date: [today]
Customer: [from input]
Salesman: [from input or default]

[Line items table with: QTY, Description, CFM, PSI, HP, V/PH/HZ, Unit Price, Extended]

[Options section if applicable]

Terms:
- Delivery: [standard or specified]
- FOB: Factory
- Freight: [included or extra]
- Validity: 30 days

RULES:
- ALWAYS use the pricing tool for prices. Never estimate or calculate.
- When unsure between two products, present both as options.
- Include relevant accessories (dryers, filters) if the customer hasn't
  specified them — compressed air systems usually need treatment.
- Flag anything unusual: "This is a medical/clean room application —
  verify NFPA compliance requirements."
```

---

## Post-MVP Roadmap

### Phase 2: Production Ready
- Deploy to staging URL (Cloudflare Pages or Vercel)
- User auth (Supabase Auth)
- Quote history — save, edit, version generated quotes
- PDF export with actual Brehob letterhead
- Updated pricing sheets from current Brehob data

### Phase 3: Intelligence Layer
- Quote win/loss tracking (did the customer accept?)
- Margin optimization suggestions ("Similar quotes averaged 32% margin")
- Seasonal patterns ("Compressor quotes spike in Q2 for manufacturing plants")
- Auto-detect when pricing data is stale

### Phase 4: Integration Layer
- **Inventory MCP tool** — real-time stock check ("QMB30 in stock at Detroit warehouse, 2-week lead time for Indianapolis")
- **Shipping MCP tool** — freight cost estimation based on weight, destination, carrier
- **CRM integration** — pull customer history, contact info, past purchases
- **Email integration** — send quote directly from the app

### Future Vision: The MCP Industrial Intranet

> *"Imagine having a centralized MCP registry for a company that ties together a bunch of APIs for an inventory system, a pricing system, and a quoting system. Maybe you even add a shipping system. All different subsystems tied together by an MCP layer of intelligence."* — Dan Hannah

This is the Path B play. Every mid-size industrial distributor has the same problem:
- Inventory in one system
- Pricing in another
- Quoting in a third
- Shipping in a fourth
- None of them talk to each other
- The "integration layer" is a person who knows all the systems

When that person retires, the company loses its connective tissue. An MCP registry that lets AI agents orchestrate across all subsystems replaces that connective tissue with something that doesn't retire, doesn't forget, and gets smarter over time.

QuoteAI is the first tool. Pricing lookup is the first MCP tool. Inventory is the second. Each one proves the pattern. The registry itself becomes the product.

```
┌──────────────────────────────────────────┐
│          MCP Tool Registry               │
│                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Pricing  │ │Inventory │ │ Shipping │ │
│  │  Tool    │ │  Tool    │ │   Tool   │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       │             │             │       │
└───────┼─────────────┼─────────────┼───────┘
        │             │             │
   ┌────▼───┐   ┌────▼────┐  ┌────▼────┐
   │ Pricing│   │Inventory│  │Shipping │
   │ System │   │ System  │  │ System  │
   │(Excel/ │   │  (ERP)  │  │(Carrier │
   │ DB)    │   │         │  │  APIs)  │
   └────────┘   └─────────┘  └─────────┘
```

---

## Technical Decisions Log

| Decision | Choice | Reasoning | Date |
|----------|--------|-----------|------|
| Retrieval engine | Supabase pgvector (not Anvil) | QuoteAI needs write operations, structured data, auth — Anvil is read-only markdown search. pgvector gives us vector search + relational data in one service. | Apr 7, 2026 |
| MVP deployment | Localhost | Zero overhead during rapid iteration. Supabase backend is cloud, so portable when ready to deploy. Demo on laptop in John's office. | Apr 7, 2026 |
| UI pattern | Chat interface (not structured form) | Don't know what fields matter yet. Chat lets AI ask clarifying questions. Avoids premature form design. Form comes post-MVP when we understand the workflow. | Apr 7, 2026 |
| Pricing approach | Deterministic tool (not LLM) | LLMs hallucinate math. Pricing must be 100% accurate. API route for MVP, MCP tool post-MVP. | Apr 7, 2026 |
| Doc conversion | textutil (macOS native) | All source docs are .doc (Office 97-2003). textutil handles them perfectly, no dependencies. | Apr 7, 2026 |
| Embedding model | OpenAI text-embedding-3-small | 1536-dim, cheap, good quality. Could switch to local (like Anvil's ONNX approach) post-MVP if cost matters. | Apr 7, 2026 |
| Full-send vs MVP | MVP first | Need John's feedback before investing in architecture. Demo sells the vision. Iteration will reshape the product. | Apr 7, 2026 |

---

## Open Questions

1. **Quote numbering system** — Does Brehob have an existing format? (e.g., "256-09-013-nc" from US Steel quote)
2. **Current pricing sheets** — Dan mentioned newer pricing data exists. Need to ingest latest version.
3. **Terms and conditions** — Are these standard across all quotes or customer-specific?
4. **Multi-option quotes** — The pricing Excel has 10 tabs (Option 1-10). How often do quotes include multiple options?
5. **Accessories/ancillaries** — How does John decide what accessories to include? Is there a standard "system" configuration?
6. **Competitor pricing** — Does Brehob track competitor quotes or win/loss reasons?
7. **Volume** — How many quotes does a salesperson generate per week/month?
8. **Google Docs API** — Enabled as of Apr 7, 2026. Need to verify Drive API access for bulk download of .doc files.
9. **PDF quotes** — Some files in Drive are PDFs (submittal packages). Do these need to be ingested too, or are they reference material?

---

## File Inventory Summary

| Category | Folders | Est. Files | Format | Priority |
|----------|---------|------------|--------|----------|
| Product Specs | 26 | 100-200 | .doc | P0 — core catalog |
| Customer Quotes | 20+ customers + year folders | 50-200 | .doc, .pdf | P0 — training data |
| Pricing Sheets | 1 folder | 8 files | .xlsx, .xls, .doc | P0 — pricing engine |
| Lead Reports | 1 folder | TBD | TBD | P2 — post-MVP |

Total estimated documents to ingest: **150-400 files**

---

*Next step: Step 0 refinement with Dan. Walk through open questions, validate data model assumptions against actual quote process, then scope MVP epics.*
