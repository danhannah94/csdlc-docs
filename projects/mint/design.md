# Mint — Project Design Doc

*Status: Draft — Step 0 Ideation (Rough Capture)*
*Created: March 29, 2026 (originally "Forge", renamed March 31)*
*Authors: Dan Hannah & Clay*
*Note: This is intentionally rough. Capturing the vision before it evaporates.*

---

## Overview

### What Is This?

Mint is a micropayment layer for the MCP (Model Context Protocol) ecosystem. It lets AI agents pay for tool calls using stablecoin-backed wallets — removing all human friction from the transaction.

**The pitch:** Your agent finds the best sports betting analysis tool on an MCP registry. It costs $0.0001 per call. Your agent has a wallet with $5 in it. It just... uses the tool and pays. You never see a checkout page, never enter a credit card, never sign up for an API key. The agent is authorized to spend. The human tops up occasionally and forgets about it.

### Why This Matters

The current model for paid APIs is built for humans:
1. Find the API → 2. Sign up → 3. Enter credit card → 4. Get API key → 5. Configure it → 6. Use it

That's 6 steps of friction. For an agent that wants to use a tool RIGHT NOW, it's a dead end.

Mint collapses this to:
1. Agent discovers tool → 2. Agent pays and uses it

That's it. The wallet was pre-funded. The authorization was pre-granted. The payment rail is crypto (stablecoins) so it's instant, global, and programmable.

### The Bigger Vision

MCP is becoming the universal protocol for agent ↔ tool communication. But there's no **economic layer.** Tools are either free (open source) or behind traditional paywalls (API keys, subscriptions). There's no middle ground for:

- **Microtransactions** — tools that cost fractions of a cent per call
- **Pay-per-use** — no subscriptions, just usage
- **Agent-native payments** — the agent pays, not the human (for each transaction)
- **Tool marketplaces** — discover, evaluate, and pay for tools in one flow

Mint IS that middle ground.

---

## Core Concepts

### Agent Wallets

Every OpenClaw agent (or any MCP-compatible agent) gets a wallet:
- Funded with stablecoins (USDC or similar)
- Human sets a balance and spending limits ("max $5/month", "max $0.01 per call")
- Agent is authorized to spend within those limits autonomously
- Human tops up when balance gets low (or auto-top-up from linked account)

### Tool Pricing

Tool providers set pricing:
- Per-call pricing (e.g., $0.0001 per search query)
- Tiered pricing (first 100 calls free, then $0.00005 each)
- Subscription pricing (optional, for power users)
- Free tier always available (keeps the ecosystem accessible)

### The Registry

An MCP-native registry where tools are:
- **Discoverable** — agents can search by capability, not just name
- **Priced** — transparent per-call costs
- **Rated** — quality scores based on usage, accuracy, reliability
- **Verified** — tool providers prove their identity and tool behavior

Think npm meets the App Store but for agent tools, with built-in payments.

### Settlement

- Stablecoin-based (USDC on a low-fee L2 — Base, Arbitrum, etc.)
- Micro-batched settlements (not one tx per tool call — that'd be insane gas costs)
- Off-chain tracking, periodic on-chain settlement
- Providers can cash out to fiat whenever

---

## Use Cases (Examples)

| Tool | Cost/Call | What It Does |
|------|-----------|-------------|
| Sports betting analysis | $0.001 | Odds comparison, edge detection, historical analysis |
| Graphic design templates | $0.01 | Access premium Figma/Canva templates programmatically |
| Legal clause search | $0.005 | Search across case law databases |
| Premium weather data | $0.0001 | High-resolution forecasts, historical data |
| Stock screener | $0.002 | Real-time screening with custom criteria |
| Translation (high quality) | $0.001 | Better than free alternatives, domain-specific |
| Code review | $0.01 | Specialized static analysis tools |

The common thread: **these tools already exist as paid APIs, but the friction prevents agents from using them autonomously.**

---

## Open Questions (Lots of Them)

### Technical
- Which L2 chain? Base is Coinbase-backed (credibility), Arbitrum has ecosystem. Need low fees + stablecoin liquidity.
- How do we handle disputes? Agent pays for a tool call that returns garbage — is there a refund mechanism?
- Off-chain ledger design — how do we batch microtransactions efficiently?
- MCP protocol extensions — does MCP need a `payment` capability in the spec, or do we wrap it?

### Business
- Who runs the registry? Us? Decentralized? Hybrid?
- Revenue model — transaction fees? Registry listing fees? Both?
- How do we bootstrap supply (tool providers) AND demand (agents) simultaneously?
- Regulatory implications of agent-controlled crypto wallets?

### Social / Philosophical
- Do humans trust agents to spend money autonomously? What's the trust-building path?
- How do we prevent a race to the bottom on pricing?
- How does this interact with existing MCP registries (if any emerge)?
- Could this enable agent-to-agent economic activity? (Agent A pays Agent B's tool, which pays Agent C's data source...)

---

## Why Us?

- We're already deep in the OpenClaw / MCP ecosystem
- We understand the agent workflow (we live it daily)
- Dan's data engineering + crypto interest + startup energy
- Clay's ability to prototype and iterate fast
- Jack at GM has similar thinking on this space (potential collaborator)
- Small team = fast iteration, no committee decisions

---

## What This Is NOT (Yet)

- Not a blockchain project — it's a payments layer that USES blockchain
- Not a token launch — no "Mint coin," just stablecoins
- Not a DAO — centralized to start, decentralize if it makes sense
- Not competing with Stripe — this is agent-native, not human-native

---

## Next Steps (When We're Ready)

- [ ] Research existing MCP registry efforts (is anyone building this?)
- [ ] Talk to Jack about the vision — get his technical perspective
- [ ] Prototype: agent wallet + single paid tool call (proof of concept)
- [ ] Explore Base/Arbitrum for settlement layer
- [ ] Write a proper whitepaper if we decide to pursue seriously

---

## Related

- [Anvil Design Doc](../anvil/design.md) — our MCP server (potential first tool on Mint)
- [Foundry Design Doc](../foundry/design.md) — doc platform (separate project)
- [CSDLC Process](../../methodology/process.md)
