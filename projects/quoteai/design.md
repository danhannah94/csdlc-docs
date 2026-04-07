# QuoteAI — Project Design Doc

*Status: Draft — Step 0 Refinement (Rev 6)*
*Created: April 7, 2026*
*Last Updated: March 29, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This?

QuoteAI is an external SaaS tool that accelerates the quoting pipeline for industrial equipment distributors. A salesperson fills out a structured form (mirroring the spreadsheet template they already use), and QuoteAI assembles a professional draft quote by cross-referencing equipment descriptions from a library of past quotes, matching products from the catalog, and formatting the output in the company's standard template.

**Critical distinction:** Salespeople own pricing completely. They negotiate deals, manage margins, and present quotes to customers in person (required for $10K+ deals). AI's job is NOT to calculate or suggest prices — it's to dramatically speed up the assembly of accurate, consistent, well-described quotes.

Input: Structured fields (customer info, equipment specs, TM number) + line items with descriptions/part numbers + pricing the salesperson has already determined
Output: A professionally formatted draft quote with equipment descriptions pulled from past quotes and product specs, ready for senior review and approval

### The Real Value Proposition

**QuoteAI is an efficiency pipeline and institutional memory system.** The real bottleneck at companies like Brehob isn't math — it's assembling the right technical language for equipment descriptions, cross-referencing vendor quotes, and formatting everything consistently.

A senior sales engineer like John Hannah has 22 years of quoting history in his head. He knows exactly how to describe a Quincy rotary screw compressor system for a food-grade environment because he's written that description 50 times. When he retires, that language library walks out the door.

QuoteAI captures that library, makes it searchable, and puts it at every salesperson's fingertips — without them ever talking directly to an AI.

### Why This Exists

**The problem:** Brehob's quoting process relies heavily on tribal knowledge. John Hannah has 22 years of experience and quote history — he knows which products fit which applications, how to describe equipment in vendor-appropriate language, and how to structure complex multi-item quotes. He's retiring. When he leaves, that knowledge walks out the door.

**Current pain points:**
- **48-hour quote turnaround** — customers wait too long, deals slip (John averaged 3-4 quotes/week, ~175/year — 75% for complete systems)
- **Inconsistent formatting** — quotes vary by salesperson, no enforced template. ~50% of quotes include multiple options, but every territory manager formats them differently (some use separate tabs, some cram options onto one tab)
- **Tribal knowledge dependency** — equipment description quality requires deep domain expertise. Knowing how to describe a system configuration correctly is the hard part, not the math.
- **Inconsistent spreadsheet submissions** — Every salesperson fills out the quoting spreadsheet differently, leading to formatting inconsistencies and missing information
- **Manual description assembly** — salespeople dig through old quotes and vendor docs to find the right technical language for equipment descriptions
- **No institutional memory** — past quotes aren't searchable or reusable. 22 years of perfectly-crafted descriptions sit in individual .doc files
- **Poor win/loss tracking** — tracking exists but is inconsistent across RSMs. Reasons for wins or losses are not always pursued or recorded
- **Quote log in SharePoint Excel** — no real dashboard, no KPIs, no visibility into pipeline health

**The opportunity:** Brehob's director is actively evaluating AI solutions. John has warm relationships and a retirement timeline that creates natural urgency. This is a real business with a real budget and a real pain point — not a solution looking for a problem.

### Who Is It For?

**MVP — John Hannah and Brehob's sales team.** The immediate users are 2-5 salespeople at Brehob who generate quotes for industrial compressed air systems, vacuum systems, dryers, filters, and related equipment.

**Post-MVP — Industrial distributors broadly.** Every industrial distributor has the same problem: experienced reps retiring, quote descriptions spread across individual files, inconsistent quoting processes. The architecture is designed to generalize, but v1 is laser-focused on Brehob.

---

## The Actual Quoting Process (from John, March 29 call)

Understanding the real workflow was a breakthrough. Our original design assumed AI would handle pricing — wrong. Here's how quotes actually work at Brehob:

### Full Pipeline

```
1. Salesperson meets with customer, understands their needs
2. Salesperson gets VENDOR QUOTES for the specific equipment
   (e.g., "US Wire Rope - IPPI installation quote.doc")
3. Salesperson builds THEIR quote using vendor quote numbers
   into a pricing quotation spreadsheet
4. Spreadsheet is submitted for review:
   - < $10K → Indy person reviews and approves
   - ≥ $10K → Senior sales (John + others) review and approve
5. Approved quote gets FORMATTED into the official Brehob quote
   (The Indy person currently does this formatting manually)
6. Official quote is sent to the customer
7. For $10K+ quotes: salesperson presents IN PERSON with:
   - The quote itself
   - Product brochures
   - Technical documents (dimensional drawings, etc.)
   - "What they're really buying is the technical docs"
```

### Key Insight: Where AI Actually Adds Value

The original design had AI doing too much (pricing, product selection, engineering). The corrected understanding reveals 4 specific AI roles:

| Role | What It Does | Why It Matters |
|------|-------------|----------------|
| **1. Equipment Library Search** | "I need a 100HP rotary screw system" → here are 5 past quotes with similar configs and the technical language used | Replaces John's 22-year mental index |
| **2. Description Assembly** | Pull proven equipment descriptions from past quotes and vendor specs instead of writing from scratch | Consistency + speed. No more hunting through old .doc files |
| **3. Template Formatting** | Standardize the output so the Indy person has less cleanup (or none) | The Indy person's current role is basically manual form validation + formatting — we automate both |
| **4. Quote Log + KPIs** | Replace the SharePoint Excel quote log with a real database and dashboard | Pipeline visibility, win/loss tracking, per-salesperson metrics |

### What AI Does NOT Do

- **Price calculation or suggestion** — Salespeople own this entirely. They need pricing expertise for negotiation.
- **Engineering assessment** — No suggesting air audits, equipment sizing, or scope changes
- **Direct interaction with salespeople** — Per John: "There needs to be a wall between the sales reps and the AI or else they will take advantage of it." The structured form IS the wall.

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

### Business Model: External SaaS (Not Consulting)

**Decision (March 29):** QuoteAI is an external product, NOT an internal integration project.

**Why external, not internal:**
- Integrating with Brehob's M365/SharePoint/Azure requires consultant access to their business systems — onboarding overhead, VPN, security clearances, and a billing model that doesn't work for a side project
- Building inside their ecosystem means everything is coupled to THEIR systems — useless for customer #2
- Dan doesn't want to be a consultant ("I don't think Brehob will shell out the money to pay me what I'd be interested in to be a consultant in my spare time")
- Our dev setup (OpenClaw + sub-agents) is "not kosher with existing IT systems"
- An external system we own is a PRODUCT — build once, sell many times

**The model:**
- Monthly subscription ($2-5K/mo target)
- Brehob uploads pricing sheets and quote history to OUR platform (one-time setup with John's help)
- We own the IP, the infrastructure, the customer relationship
- Every new customer is a new tenant, not a new consulting engagement
- Commission structure: 50% for first 10 customers → 25% (11-25) → 15% ongoing + 10% renewal
- Sales team: Matt Hannah (procurement/supply chain background), John Hannah (warm intros + retirement income stream)

**Copilot integration as Phase 2 upsell:** Once Brehob sees value, we can offer a Teams integration so senior sales can query the quote database in natural language. That's a consulting engagement AFTER subscription revenue is flowing.

### Copilot Studio Assessment (March 29 Research)

Investigated whether to build inside Microsoft's ecosystem. Key findings:

**What Copilot CAN do:**
- MCP support is GA — Copilot Studio natively connects to MCP servers via Tools → Add Tool → MCP
- Power Automate integration for approval workflows
- Azure AI Search as a knowledge source for RAG
- Deploys to Teams, SharePoint pages

**What Copilot CANNOT do:**
- Custom structured forms with field-level validation (it's conversational, not form-based)
- Custom branding / formatted quote documents
- John's "wall" requirement — a chatbot IS the AI, a form is the wall
- Fine-grained control over behavior and edge cases

**Azure all-in cost estimate:** ~$100-130/mo (AI Search Basic ~$75 + Postgres Burstable ~$20 + App Service ~$13 + OpenAI usage ~$10)

**Verdict:** Copilot is interesting for Phase 2 (natural language queries against quote database in Teams) but cannot be the primary interface. The structured form + approval workflow needs a custom app.

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

**Key examples from John's call (March 29):**
- **Good quote template:** `4M Industries, Inc. - 100hp System - Final` — this is what a properly formatted official quote looks like. Use as our output template.
- **Vendor quote example:** `US Wire Rope - IPPI installation quote.doc` — vendor quotes feed INTO the salesperson's quote. The official quote pulls language from these vendor docs.
- **Pricing spreadsheet example:** `Slate Trucks - 6 Turbo System - Pricing Quote.xlsx` — the description/part number column references existing quotes for equipment. This is the cross-referencing AI should automate.

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

The pricing formula: `LIST PRICE × MULTIPLIER = COST`, then markup for `QUOTE PRICE`. Salespeople own this math — they need to know these numbers cold for in-person negotiations.

Note: John confirmed updated pricing sheets are available in the shared Google Drive. Need to check Drive for his full library (may include sheets newer than 2020). The pricing formula structure is consistent across years.

---

## Architecture

### MVP Architecture (Revised)

```
┌──────────────────────────────────────────────────────┐
│  Next.js Web App (external SaaS)                     │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Quote Request Form (structured, validated)     │  │
│  │  [Customer info] [Equipment specs] [Line items] │  │
│  │  [Pricing — entered by salesperson]             │  │
│  │  [Context: free text for special requirements]  │  │
│  └───────────────────┬────────────────────────────┘  │
│                      │                               │
│  ┌───────────────────▼────────────────────────────┐  │
│  │  AI-Assembled Draft Quote                       │  │
│  │  • Equipment descriptions from library          │  │
│  │  • Vendor language cross-referenced             │  │
│  │  • Formatted in Brehob official template        │  │
│  │  • Salesperson's pricing preserved exactly       │  │
│  └───────────────────┬────────────────────────────┘  │
│                      │                               │
│  ┌───────────────────▼────────────────────────────┐  │
│  │  Approval Workflow                              │  │
│  │  < $10K → Indy reviewer                        │  │
│  │  ≥ $10K → Senior sales (John + others)         │  │
│  │  Feedback loops for declined quotes             │  │
│  └───────────────────┬────────────────────────────┘  │
│                      │                               │
│  ┌───────────────────▼────────────────────────────┐  │
│  │  Quote Log + KPI Dashboard                      │  │
│  │  • All quotes tracked in real database          │  │
│  │  • Per-salesperson metrics                      │  │
│  │  • Pipeline visibility, win/loss tracking       │  │
│  └────────────────────────────────────────────────┘  │
└───────────────────┬──────────────────────────────────┘
                    │
   ┌────────────────▼─────────────────┐
   │  Backend Services                 │
   │  (Next.js API routes)            │
   └──┬──────────┬──────────┬─────────┘
      │          │          │
┌─────▼──┐ ┌────▼─────┐ ┌──▼──────────┐
│ LLM    │ │ MCP      │ │ MCP         │
│  API   │ │Equipment │ │Past Quote   │
│(Sonnet)│ │Search    │ │Search       │
│        │ │Tool      │ │Tool         │
│Assemble│ │          │ │             │
│ draft  │ │Product   │ │Cross-ref    │
│ quote  │ │catalog   │ │descriptions │
└────────┘ └──┬───────┘ └──┬──────────┘
              │             │
     ┌────────▼─────────────▼──────────┐
     │  Docker: Postgres + pgvector    │
     │  (localhost:5432 → managed PG)  │
     │                                 │
     │  - products (catalog + vector)  │
     │  - past_quotes (text + vector)  │
     │  - quote_log (all quotes + KPI) │
     │  - vendor_quotes (reference)    │
     │  - feedback (quality signal)    │
     └────────────────────────────────┘
```

### What Changed from Rev 5

| Rev 5 (Old) | Rev 6 (New) | Why |
|---|---|---|
| AI calculates pricing via MCP pricing tool | Salesperson enters pricing; AI never touches it | Salespeople own pricing for negotiation leverage |
| MCP pricing server (get_pricing, list_pricing) | REMOVED from MVP | Pricing is the salesperson's job, not the system's |
| AI generates complete quote from specs | AI assembles descriptions from past quotes | The bottleneck is descriptions, not math |
| Indy person reviews and formats | App replaces Indy formatting role; approval workflow built in | Indy person's job is basically form validation + formatting — we automate both |
| No approval workflow in MVP | Tiered approval ($10K threshold) in MVP | Core to the real process |
| No quote log | Quote log + KPI dashboard included | Replaces SharePoint Excel — massive upgrade |
| Copilot/MS integration considered | External SaaS, MS integration is Phase 2 upsell | Consulting overhead kills the business model |

### MCP-First Architecture

MCP is still first-class, but the tool set has shifted to match the real problem:

```
quoteai/
├── mcp-servers/
│   ├── equipment/        # MCP server: search_equipment, get_product, get_specs
│   │   ├── src/
│   │   └── package.json
│   └── quotes/           # MCP server: search_past_quotes, get_quote, get_descriptions
│       ├── src/
│       └── package.json
├── app/                  # Next.js web UI
├── ingestion/            # Data pipeline scripts
└── mcp-config.json       # Registry for all tools
```

#### Equipment Search Tool

```typescript
// MCP Server: @quoteai/equipment
// Tools:

// search_equipment — semantic search for matching equipment
// Params: { query: "oilless compressor for food grade", cfm_min: 150, psi: 125 }
// Returns: ranked list of matching products with specs and similarity scores

// get_product — full details for a specific model
// Params: { model: "QMB30" }
// Returns: complete product spec sheet data (description, dimensions, weight, etc.)

// get_specs — structured spec comparison
// Params: { models: ["QMB25", "QMB30"] }
// Returns: side-by-side comparison table
```

#### Past Quote Search Tool

```typescript
// MCP Server: @quoteai/quotes
// Tools:

// search_past_quotes — find similar past quotes by equipment or customer type
// Params: { query: "100HP rotary screw system for manufacturing plant" }
// Returns: ranked list of similar past quotes with descriptions and line items

// get_quote — full details of a specific past quote
// Params: { quote_id: "uuid" }
// Returns: complete quote with all line items, descriptions, terms

// get_descriptions — pull equipment description language from past quotes
// Params: { equipment_type: "rotary screw compressor", context: "food grade" }
// Returns: proven description language from similar past quotes
```

### Why This Stack

| Choice | Reasoning |
|--------|-----------|
| **Next.js** | React frontend + API routes in one project. Dan knows it. Fast to iterate. |
| **External SaaS** | Own the product, sell to many customers. No consulting overhead. |
| **Docker Postgres + pgvector** | Fully local for MVP — zero external dependency. Same schema migrates to managed Postgres. |
| **MCP servers** | Equipment search + past quote search as independent tools. Composable, testable by agents. |
| **Anthropic API (Sonnet)** | ~$0.02 per quote assembly. Handles description matching and template formatting well. |
| **OpenAI Embeddings** | `text-embedding-3-small` at $0.02/million tokens. Entire corpus costs ~$0.01 to embed. |
| **Structured form (the "wall")** | Per John: salespeople must not interact directly with AI. The form enforces consistency AND prevents misuse. |

### LLM & Embedding APIs

**Quote assembly:** Anthropic API with Claude Sonnet. ~$0.02 per quote (2K tokens in, 1K out). Development budget: $5-20/month with $25 hard cap. Sonnet over Opus — structured output quality is comparable, 10x cheaper.

**Embeddings:** OpenAI `text-embedding-3-small` (1536-dim, $0.02/M tokens). Claymore already has an OpenAI API key. Entire corpus embeds for ~$0.01.

### Data Model (Postgres + pgvector via Docker)

```sql
-- Product catalog (from spec sheet .doc files)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manufacturer TEXT NOT NULL,
    category TEXT NOT NULL,
    series TEXT,
    model TEXT NOT NULL,
    mount_type TEXT,
    cooling TEXT,
    hp_range NUMRANGE,
    cfm_range NUMRANGE,
    psi_range NUMRANGE,
    voltage_options TEXT[],
    drive_system TEXT,
    control_system TEXT,
    dimensions TEXT,
    weight_lbs INTEGER,
    noise_dba NUMERIC,
    refrigerant TEXT,
    connections TEXT,
    raw_text TEXT NOT NULL,
    embedding VECTOR(1536),
    source_file TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Past quotes (from customer quote .doc files)
CREATE TABLE past_quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_name TEXT NOT NULL,
    customer_contact TEXT,
    quote_number TEXT,              -- TM#-YY-###-initials format
    quote_date DATE,
    total_value NUMERIC,
    num_items INTEGER,
    complexity TEXT,                -- 'simple', 'standard', 'complex'
    summary TEXT,                   -- AI-generated summary
    raw_text TEXT NOT NULL,
    embedding VECTOR(1536),
    source_file TEXT,
    source_folder TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Quote line items (parsed from past quotes — the DESCRIPTION LIBRARY)
CREATE TABLE quote_line_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id UUID REFERENCES past_quotes(id),
    item_number INTEGER,
    description TEXT NOT NULL,      -- THE KEY FIELD: proven equipment description language
    quantity INTEGER DEFAULT 1,
    unit_price NUMERIC,
    total_price NUMERIC,
    product_id UUID REFERENCES products(id),
    embedding VECTOR(1536),         -- embed descriptions for semantic search
    notes TEXT
);

-- Vendor quotes (referenced in building final quotes)
CREATE TABLE vendor_quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vendor_name TEXT NOT NULL,
    description TEXT,
    raw_text TEXT NOT NULL,
    embedding VECTOR(1536),
    source_file TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Quote log (replaces SharePoint Excel — ALL quotes tracked here)
CREATE TABLE quote_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_number TEXT NOT NULL,     -- TM#-YY-###-initials
    salesperson TEXT NOT NULL,
    tm_number TEXT NOT NULL,
    customer_name TEXT NOT NULL,
    customer_contact TEXT,
    customer_email TEXT,
    customer_phone TEXT,
    bill_to_address TEXT,
    ship_to_address TEXT,
    total_amount NUMERIC,
    status TEXT DEFAULT 'draft',    -- draft, pending_review, approved, sent, won, lost
    approval_tier TEXT,             -- 'indy' (<$10K) or 'senior' (≥$10K)
    approved_by TEXT,
    approved_at TIMESTAMPTZ,
    win_loss_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Quote log line items (the actual quote content)
CREATE TABLE quote_log_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_log_id UUID REFERENCES quote_log(id),
    item_number INTEGER,
    description TEXT NOT NULL,
    quantity INTEGER DEFAULT 1,
    cfm NUMERIC,
    psi NUMERIC,
    hp NUMERIC,
    voltage TEXT,
    cooling TEXT,
    unit_price NUMERIC,            -- entered by salesperson
    extended_total NUMERIC,
    source_quote_id UUID REFERENCES past_quotes(id),  -- which past quote the description came from
    notes TEXT
);

-- Feedback on AI-assembled descriptions
CREATE TABLE feedback (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_log_id UUID REFERENCES quote_log(id),
    rating TEXT,                    -- 'up' or 'down'
    comment TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Indexes
CREATE INDEX ON products USING ivfflat (embedding vector_cosine_ops) WITH (lists = 20);
CREATE INDEX ON past_quotes USING ivfflat (embedding vector_cosine_ops) WITH (lists = 20);
CREATE INDEX ON quote_line_items USING ivfflat (embedding vector_cosine_ops) WITH (lists = 20);
CREATE INDEX ON vendor_quotes USING ivfflat (embedding vector_cosine_ops) WITH (lists = 20);
```

### Input Validation (The "Wall")

The structured form serves two purposes: data quality AND preventing direct AI interaction.

**Required fields (per John):** Company name, contact person, address, email, phone, ship-to, CFM, PSI, voltage, water/air cooled, HP, date quote required, TM number, model & description of what is being sold.

**Validation rules:**
- CFM: 1–5,000 range (Brehob's equipment range)
- PSI: 10–500 range
- HP: 1–500 range
- Voltage: Must match known patterns (e.g., "230/1/60", "460/3/60")
- TM Number: Required (salesperson's territory number)
- Company name, contact, address, email, phone, ship-to: Required text fields
- Date required: Valid future date
- At least one line item with description

**This replaces the Indy person's validation role.** She currently catches missing fields and formatting issues manually — the form does this automatically.

### Approval Workflow

```
Quote submitted
  ├── Total < $10K → Indy reviewer queue
  │     ├── Approved → Official quote generated → Salesperson notified
  │     └── Declined → Feedback to salesperson → Revise and resubmit
  │
  └── Total ≥ $10K → Senior sales queue (John + others)
        ├── Approved → Official quote generated → Salesperson notified
        └── Declined → Feedback to salesperson → Revise and resubmit
```

Notifications via email. The approved quote is formatted in the official Brehob template (see `4M Industries` as reference).

### Quote Log + KPI Dashboard

Replaces the SharePoint Excel spreadsheet with a real database and visualization layer.

**Dashboard views:**
- All quotes by status (draft → pending → approved → sent → won/lost)
- Per-salesperson metrics (volume, win rate, average deal size)
- Pipeline value (total $ in each status)
- Quote turnaround time (submitted → approved)
- Win/loss reasons (structured capture — fixing the inconsistency John flagged)
- Trend lines (monthly volume, seasonal patterns)

This alone could sell the product. Every sales manager wants visibility into their pipeline.

---

## MVP Scope (Revised)

### What's In

1. **Structured quote request form** — Mirrors existing spreadsheet with enforced validation. The "wall" between salespeople and AI.
2. **Equipment description search** — AI finds matching descriptions from past quotes and product specs
3. **Past quote cross-referencing** — "Show me quotes similar to this configuration"
4. **Draft quote assembly** — AI formats a Brehob-template quote using matched descriptions + salesperson's pricing
5. **Tiered approval workflow** — <$10K to Indy, ≥$10K to senior sales. Decline → feedback → resubmit.
6. **Quote log** — Every quote tracked in a real database (replaces SharePoint Excel)
7. **Basic KPI dashboard** — Quote volume, status breakdown, per-salesperson metrics
8. **Feedback buttons** — 👍/👎 on AI-assembled descriptions

### What's Out (Post-MVP)

- User authentication / multi-tenancy (just Brehob for MVP)
- PDF export with actual Brehob letterhead (copy-paste is fine for MVP)
- Copilot / Teams integration
- Win/loss analytics and recommendations
- Inventory integration
- CRM integration
- Email integration (auto-send to acquotes@brehob.com)
- Multiple pricing tiers / customer-specific pricing
- Vendor quote ingestion pipeline (manual upload for MVP)
- Mobile UI

### MVP User Flow (Revised)

```
1. Salesperson opens app
2. Fills out quote request form:
   - Customer: Groeb Farms
   - Contact: Jane Smith
   - Address / Ship To: 110 W Michigan Ave, Fremont, MI
   - Email: jsmith@groebfarms.com
   - Phone: 231-924-3800
   - TM Number: 256
   - Date Required: 04/30/2026

3. Adds line items:
   - Item 1: Powerex Oilless Scroll Compressor
     CFM: 200 | PSI: 125 | HP: 50 | Voltage: 460/3/60 | Cooling: Air
     → AI searches equipment library → suggests description language
        from similar past quotes (e.g., "Powerex SEQ1007 Oil-Free
        Scroll Compressor, 200 CFM @ 125 PSI, 50HP...")
     → Salesperson reviews/edits description
     → Salesperson enters THEIR price: $45,000

   - Item 2: Zeks Refrigerated Air Dryer
     → AI matches → suggests description
     → Salesperson enters price: $8,500

4. Salesperson adds free-text context:
   "Food-grade environment, need oilless for contamination
    requirements. Similar to the pharma work we did last year."

5. Clicks "Submit for Review"
   → Total = $53,500 (≥ $10K) → routes to senior sales queue

6. Senior sales (John) reviews:
   - Sees the draft quote in official Brehob format
   - Equipment descriptions are professional and consistent
   - Pricing is what the salesperson entered (John validates margins)
   - Approves ✅

7. Official quote is generated in Brehob template format
   - Salesperson downloads/copies
   - Presents to customer with brochures and technical docs

8. Quote logged in dashboard (status: sent)
   - Later marked won/lost with reason
```

### MVP Success Criteria

- Salespeople can submit a quote request form in < 5 minutes (vs. hunting through old docs)
- AI-suggested equipment descriptions match the quality of John's past quotes (>80% usable without major edits)
- Approval workflow correctly routes by $10K threshold
- Quote log has 100% of quotes (vs. inconsistent SharePoint tracking)
- John's reaction: "This is how every quote should look"
- Indy person's formatting workload drops to near-zero

---

## Prompt Engineering

### System Prompt (Draft — Revised)

```
You are QuoteAI, a quote assembly assistant for Brehob, an industrial air
equipment distributor. You help assemble professional, consistent quote
documents by matching equipment descriptions from Brehob's library of past
quotes and product specifications.

IMPORTANT: You do NOT set or suggest pricing. The salesperson owns all
pricing decisions. Your job is to find the right equipment descriptions
and format them into a professional quote document.

IMPORTANT: You do NOT interact with salespeople directly. You operate
behind a structured form. Your outputs are description suggestions and
formatted quote documents.

WORKFLOW:
1. Receive structured input from the quote request form
2. For each line item, search the equipment library for matching products
3. Search past quotes for similar configurations
4. Pull the best equipment description language from matches
5. Assemble descriptions into draft quote in Brehob's official template
6. Preserve the salesperson's pricing exactly as entered

DESCRIPTION MATCHING:
- Cross-reference the description/part number against existing past quotes
- Pull proven technical language — don't invent new descriptions
- When multiple past quotes match, prefer the most recent and most similar
- Include relevant technical details: model, CFM, PSI, HP, voltage,
  dimensions, weight, control system, cooling type
- Match the tone and style of Brehob's official quotes
  (see "4M Industries, Inc. - 100hp System - Final" as template)

QUOTE FORMAT:
[Brehob letterhead]
Quote #: [TM#-YY-###-initials format, e.g. 256-26-001-jfh]
Date: [today]
Customer: [from form]
Salesman: [from form / TM number]

[Line items with AI-assembled descriptions + salesperson pricing]

Terms:
- Delivery: [standard or specified]
- FOB: Factory
- Freight: [included or extra]
- Validity: 30 days

RULES:
- NEVER modify, calculate, or suggest pricing. Use exactly what the
  salesperson entered.
- When suggesting descriptions, cite the source quote (e.g., "Based on
  US Steel 2008 quote, line item 3")
- If no matching past quotes exist, use product spec sheet language
- If a line item is ambiguous, flag it for the salesperson to clarify
- Quote number format: TM number - year (2-digit) - sequential number
  for the year - optional initials. Initials may include middle name
  (e.g., jfh = John Francis Hannah)
```

---

## Ingestion Plan

### Phase 1: Product Catalog

1. Download all product spec `.doc` files from Drive (26 category folders)
2. Convert `.doc` → text using `textutil` (macOS native)
3. Parse structured fields: manufacturer, model, specs, dimensions, weight
4. Generate embeddings via OpenAI `text-embedding-3-small` (1536-dim)
5. Load into `products` table

Estimated volume: ~100-200 product spec documents

### Phase 2: Past Quotes (THE CORE ASSET)

This is the most important ingestion phase — John's 22 years of quoting expertise.

1. Download customer quote `.doc` files from all customer folders
2. Convert to text, generate embeddings for semantic search
3. Parse structured data: customer name, line items, descriptions, prices, dates
4. **Crucially: embed individual LINE ITEM DESCRIPTIONS** — these are the reusable atoms
5. Load into `past_quotes`, `quote_line_items`, and `vendor_quotes` tables

Estimated volume: ~50-200 customer quote documents + vendor quotes

### Phase 3: Pricing Data (Reference Only)

Pricing data is still ingested for reference (product lookup, not AI pricing), but the salesperson enters all prices manually.

1. Download pricing Excel files from Drive (John uploaded full library)
2. Parse with `SheetJS` — extract models, list prices, multipliers
3. Load into a `pricing_reference` table for product lookup
4. NOT used by AI for price suggestions — only for salesperson reference

### Ingestion Script Design

```
quoteai/ingestion/
├── download.ts          # Pull files from Google Drive
├── convert.ts           # .doc/.xlsx → text/JSON (textutil + SheetJS)
├── parse-products.ts    # Extract structured fields from product specs
├── parse-quotes.ts      # Extract quote data + line item descriptions
├── parse-vendors.ts     # Extract vendor quote content
├── embed.ts             # Generate embeddings (products + quotes + line items)
├── load.ts              # Insert into Postgres
└── validate.ts          # Spot-check data integrity
```

---

## Agent Testing Strategy

### How Clay and Sub-Agents Test QuoteAI

1. **MCP tool unit tests** — Sub-agent calls `search_equipment({ query: "oilless compressor" })` directly, verifies response structure
2. **Description matching tests** — Given a line item, does the AI pull relevant past quote descriptions?
3. **Template formatting tests** — Does output match the 4M Industries reference template?
4. **Approval routing tests** — Does the $10K threshold route correctly?
5. **End-to-end tests** — Submit a quote request form → verify descriptions → verify formatting → verify quote log entry

---

## Post-MVP Roadmap

### Phase 2: Production Ready
- Deploy to hosted URL (Vercel or Cloudflare Pages)
- User authentication / multi-tenancy
- PDF export with actual Brehob letterhead
- Vendor quote upload + auto-parsing
- Updated pricing sheets from current Brehob data

### Phase 3: Intelligence Layer
- Quote win/loss analytics (patterns, seasonal trends)
- Accessory recommendations based on past quote patterns (John's "best practices" tribal knowledge)
- Margin analysis tools (for senior sales only — behind the wall)
- Auto-detect stale data (old descriptions, outdated products)

### Phase 4: Copilot / Teams Integration (Upsell)
- Natural language queries against quote database via Teams
- "What did we quote Ford last year for their rotary screw system?"
- "How many quotes are pending approval?"
- This is a consulting engagement AFTER subscription revenue is flowing

### Phase 5: Multi-Tenant / Generalized
- Onboard second industrial distributor
- Abstract Brehob-specific templates into configurable templates
- White-label option

### Future Vision: The MCP Industrial Intranet

> *"Imagine having a centralized MCP registry for a company that ties together a bunch of APIs for an inventory system, a pricing system, and a quoting system."* — Dan Hannah

QuoteAI's MCP servers (equipment search, past quote search) are the first tools in this registry. As more tools are added (inventory, shipping, CRM), the registry itself becomes the product — an AI-powered connective tissue for industrial operations.

---

## Technical Decisions Log

| Decision | Choice | Reasoning | Date |
|----------|--------|-----------|------|
| Retrieval engine | Postgres + pgvector (not Anvil) | QuoteAI needs write operations, structured data, auth. pgvector gives us vector search + relational data in one service. | Apr 7 |
| MVP database | Docker Postgres + pgvector | Fully local for MVP. Same schema migrates to managed Postgres for production. | Apr 7 |
| UI pattern | Structured form (the "wall") | Salespeople must not interact with AI directly. Form enforces consistency AND prevents misuse. Per John. | Apr 7, Mar 29 |
| MCP architecture | First-class from day one | CSDLC mandates MCP as first-class. Equipment search + past quote search as independent tools. | Apr 7 |
| Doc conversion | textutil (macOS native) | All source docs are .doc (Office 97-2003). textutil handles them perfectly. | Apr 7 |
| Embedding model | OpenAI text-embedding-3-small | 1536-dim, $0.02/M tokens. Claymore already has OpenAI API key. | Apr 7 |
| LLM for assembly | Anthropic Sonnet (via API) | ~$0.02/quote. Handles description matching and template formatting well. | Apr 7 |
| Trust mechanism | Source citations + description provenance | Every AI-suggested description cites which past quote it came from. | Apr 7 |
| AI scope (REVISED) | Description assembly + formatting ONLY | AI does NOT handle pricing, engineering, or direct salesperson interaction. Per John (Mar 29 call). | Mar 29 |
| Pricing ownership | Salesperson only | Salespeople own pricing for negotiation leverage. AI never suggests or calculates prices. Per John. | Mar 29 |
| AI wall | Structured form, no chat | "There needs to be a wall between the sales reps and the AI or else they will take advantage of it." Per John. | Mar 29 |
| Approval workflow | Tiered by $10K threshold | <$10K → Indy person. ≥$10K → Senior sales (John). Per John. | Mar 29 |
| Indy person's role | Automated out | Her current job is manual form validation + formatting. The app does both. | Mar 29 |
| Quote log | Real database + dashboard | Replaces SharePoint Excel. Proper KPIs, win/loss tracking, pipeline visibility. | Mar 29 |
| Quote numbering | TM#-YY-###-initials format | Existing Brehob standard. Initials may include middle name (e.g., jfh = John Francis Hannah). | Mar 29 |
| Terms & conditions | Standard (not per-customer) | Same terms on every quote. Simplifies template. Per John. | Mar 29 |
| Business model | External SaaS (NOT consulting) | Internal MS integration requires consultant access, couples to one customer, kills scalability. External = product company. | Mar 29 |
| Copilot role | Phase 2 upsell only | Copilot can't do structured forms, custom templates, or the "wall." Good for natural language quote queries in Teams post-MVP. | Mar 29 |
| MCP pricing tool | REMOVED from MVP | Salespeople own pricing. No AI price calculation or suggestion. | Mar 29 |
| Required form fields | 15+ fields per John | Company, contact, address, email, phone, ship-to, CFM, PSI, voltage, cooling, HP, date, TM number, model & description. | Mar 29 |
| Data sensitivity | Cloud OK | Brehob already cloud-based. No on-prem requirement. | Mar 29 |

---

## Open Questions

### Resolved
- ~~LLM API access~~ → Anthropic Sonnet via API, OpenAI for embeddings
- ~~Supabase vs local~~ → Docker Postgres + pgvector for MVP
- ~~Chat vs form~~ → Structured form (the "wall")
- ~~AI scope~~ → Description assembly + formatting ONLY. No pricing, no engineering.
- ~~Copilot vs custom~~ → External SaaS. Copilot is Phase 2 upsell.
- ~~Business model~~ → External SaaS, not consulting. Monthly subscription.
- ~~Pricing ownership~~ → Salesperson only. AI never touches pricing.
- ~~Approval workflow~~ → Tiered: <$10K Indy, ≥$10K senior sales.
- ~~Quote numbering~~ → TM#-YY-###-initials (initials may include middle name).
- ~~Terms~~ → Standard across all quotes.
- ~~Data sensitivity~~ → Cloud OK.
- ~~Required fields~~ → 15 fields per John.

### Still Open
- **Template reference:** Need to study `4M Industries, Inc. - 100hp System - Final` closely to extract the exact output template format.
- **Updated pricing sheets:** John said he uploaded his full library to Drive. Need to check what's there.
- **Vendor quote handling:** How do we handle vendor quotes in the system? Manual upload per quote? Ingested in bulk?
- **Quote numbering auto-increment:** Does the system auto-generate quote numbers or does the salesperson enter them?
- **Multi-option quote formatting:** ~50% of quotes have multiple options. How do we handle this in the form and output?

---

## File Inventory Summary

| Category | Folders | Est. Files | Format | Priority |
|----------|---------|------------|--------|----------|
| Product Specs | 26 | 100-200 | .doc | P0 — core catalog |
| Customer Quotes | 20+ customers + year folders | 50-200 | .doc, .pdf | P0 — description library (THE core asset) |
| Pricing Sheets | 1 folder | 8+ files | .xlsx, .xls, .doc | P1 — reference only |
| Vendor Quotes | mixed | TBD | .doc | P1 — feeds into quote assembly |
| Lead Reports | 1 folder | TBD | TBD | P2 — post-MVP |

Total estimated documents to ingest: **150-400 files**

---

*Next step: Dan deciding on whether to proceed with external SaaS approach. If yes: study the 4M Industries reference template, check Google Drive for updated pricing sheets, then scope MVP epics (E0: scaffold, E1: ingestion, E2: MCP servers, E3: web UI + form + approval + dashboard).*
