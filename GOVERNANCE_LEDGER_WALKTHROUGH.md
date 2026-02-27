# 🔍 Glass Box Governance — NDJSON Decision Ledger Walkthrough

> **Every agent decision, every tool call, every reasoning step — recorded as a single JSON line.**

This document walks through two real-world conversations to show exactly what the Governance Ledger captures, why each field exists, and how to query the resulting log.

---

## Table of Contents

1. [Schema at a Glance](#1-schema-at-a-glance)
2. [Field Reference (Plain English)](#2-field-reference-plain-english)
3. [Task 1 — Portfolio Analysis](#3-task-1--portfolio-analysis)
4. [Task 2 — Market Impact on Portfolio](#4-task-2--market-impact-on-portfolio)
5. [Querying the Ledger](#5-querying-the-ledger)
6. [Receipt Counts Comparison](#6-receipt-counts-comparison)

---

## 1. Schema at a Glance

Every receipt written to `logs/governance_receipts.ndjson` follows this unified, deduplicated schema:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    GOVERNANCE RECEIPT SCHEMA                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─ 1. Core Lifecycle & Metadata ─────────────────────────────────┐ │
│  │                                                                 │ │
│  │  ts               → When did this happen?                       │ │
│  │  trace_id         → Which conversation is this part of?         │ │
│  │  agent_persona    → Which agent did this?                       │ │
│  │  event_type       → What lifecycle step occurred?               │ │
│  │                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ 2. State & Context Capture ───────────────────────────────────┐ │
│  │                                                                 │ │
│  │  system_instructions  → What instructions was the agent given?  │ │
│  │  state_snapshot       → What did the world look like?           │ │
│  │                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ 3. Cognitive Transparency ────────────────────────────────────┐ │
│  │                                                                 │ │
│  │  chain_of_thought  → WHY did the agent make this decision?     │ │
│  │  discarded_paths   → What alternatives did it reject?           │ │
│  │                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ 4. Tool Interaction & Errors ─────────────────────────────────┐ │
│  │                                                                 │ │
│  │  invocation_specs  → Which tool, with what arguments?           │ │
│  │  latency_ms        → How long did it take?                      │ │
│  │  error_category    → What kind of failure? (if any)             │ │
│  │                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ 5. Feedback & Evidence (RLAIF Prep) ──────────────────────────┐ │
│  │                                                                 │ │
│  │  evidence          → Proof the work was done                    │ │
│  │  success_label     → Quick pass/fail tag                        │ │
│  │  hitl_annotation   → Reserved: human reviewer notes (null)      │ │
│  │  reward_score      → Reserved: RLAIF training score (null)      │ │
│  │                                                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Event Types — The Seven Lifecycle Steps

```
  task_start ──→ tool_propose ──→ tool_execute ──→ tool_complete ──→ task_complete
                                        │                                │
                                        └──→ tool_error                  │
                                                                         │
                                                              task_fail ─┘ (on error)
```

---

## 2. Field Reference (Plain English)

| Field | Category | What It Means | Example |
|:------|:---------|:-------------|:--------|
| `ts` | Core | **When** — UTC timestamp of this event | `"2026-02-27T10:15:33.421Z"` |
| `trace_id` | Core | **Which conversation** — links all receipts from one user query | `"t1-port-a1b2c3d4-..."` |
| `agent_persona` | Core | **Who** — agent ID + human name | `"l3_vic::Virtual Investment Concierge L3"` |
| `event_type` | Core | **What happened** — one of the 7 lifecycle steps | `"tool_propose"` |
| `system_instructions` | State | **Agent's instructions** — the system prompt (only on `task_start`) | `"You are the VIC..."` |
| `state_snapshot` | State | **World state** — query, context, tools already called | `{"query": "...", "tool_call_history": {}}` |
| `chain_of_thought` | Cognitive | **Why** — the LLM's actual reasoning before a decision | `"User wants portfolio data, calling..."` |
| `discarded_paths` | Cognitive | **What else was considered** — rejected alternatives | `["Could use financial_analyst but..."]` |
| `invocation_specs` | Tool | **Tool + arguments** — what's being called and with what | `{"tool": "call_portfolio_analyzer", "kwargs": {...}}` |
| `latency_ms` | Tool | **Duration** — how long the operation took | `14805.00` (= 14.8 seconds) |
| `error_category` | Tool | **Failure type** — structured error taxonomy | `"SYSTEM_ERROR"`, `"MODEL_ERROR"`, `"TOOL_ERROR"` |
| `evidence` | Feedback | **Proof of work** — output size, preview, content flags | `{"output_chars": 4521, "has_content": true}` |
| `success_label` | Feedback | **Quick tag** — did it work? | `"ok"`, `"fail"`, `"timeout"` |
| `hitl_annotation` | Feedback | **Human review slot** — always `null` now, for future use | `null` |
| `reward_score` | Feedback | **RLAIF training slot** — always `null` now, for future use | `null` |
| `task_id` | Core | **Task reference** — links to A2A task tracking | `"task-t1-001"` |

---

## 3. Task 1 — Portfolio Analysis

> **User query:** *"analyze deepak sharma portfolio"*

### Conversation Flow

```
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║  USER: "analyze deepak sharma portfolio"                                ║
  ╚═══════════════════════╤════════════════════════════════════════════════╝
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │  L3 — Virtual Investment Concierge (VIC)              trace: t1-port-  │
  │  ─────────────────────────────────────────────                         │
  │                                                                         │
  │  📋 task_start                                                          │
  │  │  Receives query, logs system instructions & state                    │
  │  │                                                                      │
  │  🧠 tool_propose  →  call_portfolio_analyzer                            │
  │  │  chain_of_thought: "This is a portfolio question                     │
  │  │  — I need the Portfolio Analyzer, not the Financial Analyst"          │
  │  │  discarded_paths: ["Considered call_financial_analyst                 │
  │  │  but this is purely a portfolio composition question"]                │
  │  │                                                                      │
  │  🚀 tool_execute  →  call_portfolio_analyzer                            │
  │  │  timeout: 120s                                                       │
  │  │                                                                      │
  │  │  ┌─────────────────────────────────────────────────────────────────┐ │
  │  │  │  L2 — Portfolio Analyzer                                        │ │
  │  │  │  ───────────────────────                                        │ │
  │  │  │                                                                  │ │
  │  │  │  📋 task_start                                                   │ │
  │  │  │  │                                                               │ │
  │  │  │  🧠 tool_propose  →  portfolio_l1_agent                          │ │
  │  │  │  │  chain_of_thought: "I need raw holdings data                  │ │
  │  │  │  │  from the L1 agent"                                           │ │
  │  │  │  │                                                               │ │
  │  │  │  🚀 tool_execute  →  portfolio_l1_agent                          │ │
  │  │  │  │  timeout: 90s                                                 │ │
  │  │  │  │                                                               │ │
  │  │  │  │  ┌───────────────────────────────────────────────────────┐   │ │
  │  │  │  │  │  L1 — Portfolio Primitive Analyzer                    │   │ │
  │  │  │  │  │  ─────────────────────────────────                    │   │ │
  │  │  │  │  │                                                       │   │ │
  │  │  │  │  │  📋 task_start                                        │   │ │
  │  │  │  │  │  │  Fetches holdings from database                    │   │ │
  │  │  │  │  │  │  RELIANCE 40.2% │ INFY 25.1%                      │   │ │
  │  │  │  │  │  │  TCS 20.3%      │ HDFCBANK 14.4%                  │   │ │
  │  │  │  │  │  ✅ task_complete   latency: 6,450ms                  │   │ │
  │  │  │  │  └───────────────────────────────────────────────────────┘   │ │
  │  │  │  │                                                               │ │
  │  │  │  ✅ tool_complete  ←  portfolio_l1_agent   latency: 7,103ms     │ │
  │  │  │  │  evidence: "RELIANCE 40.2%, INFY 25.1%, TCS 20.3%..."       │ │
  │  │  │  │                                                               │ │
  │  │  │  │  [ L2 synthesizes: concentration risk HIGH (>35%) ]           │ │
  │  │  │  │                                                               │ │
  │  │  │  ✅ task_complete   latency: 14,100ms                            │ │
  │  │  └─────────────────────────────────────────────────────────────────┘ │
  │  │                                                                      │
  │  ✅ tool_complete  ←  call_portfolio_analyzer   latency: 14,805ms      │
  │  │  evidence: "Concentration Risk: HIGH — single stock                  │
  │  │  (RELIANCE) exceeds 35% threshold"                                   │
  │  │                                                                      │
  │  │  [ L3 writes final response for user ]                               │
  │  │                                                                      │
  │  ✅ task_complete   total_latency: 19,779ms                             │
  │     evidence: 4,521 chars                                               │
  └─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
  ╔══════════════════════════════════════════════════════════════════════════╗
  ║  RESPONSE: "Deepak Sharma's portfolio is concentrated in large-cap      ║
  ║  Indian equities. RELIANCE at 40.2% exceeds the 35% threshold..."       ║
  ╚══════════════════════════════════════════════════════════════════════════╝
```

### Ledger Output — 11 Receipt Lines

Each block below is **one line** in the actual `.ndjson` file (formatted here for readability).

#### Receipt 1 — L3 VIC starts

```json
{
  "ts": "2026-02-27T10:15:33.421Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "task_start",
  "system_instructions": "You are the Virtual Investment Concierge (VIC), a top-tier financial advisor. You coordinate analysis by delegating to specialized sub-agents...",
  "state_snapshot": {
    "query": "analyze deepak sharma portfolio",
    "context_id": "ctx-t1-001",
    "session_id": "sess-deepak-001"
  },
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t1-001"
}
```

#### Receipt 2 — L3 proposes calling Portfolio Analyzer

```json
{
  "ts": "2026-02-27T10:15:34.102Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "call_portfolio_analyzer",
    "kwargs": {
      "query": "Analyze the complete portfolio for investor Deepak Sharma including holdings breakdown, allocation, and risk assessment"
    }
  },
  "chain_of_thought": "The user is asking me to analyze Deepak Sharma's portfolio. This is a portfolio-specific question — I need to delegate to the Portfolio Analyzer which has access to portfolio data and can break down holdings and allocation. I do not need the Financial Analyst for this since the user is not asking about specific market conditions.",
  "discarded_paths": [
    "Considered call_financial_analyst but this is purely a portfolio composition question, not a market analysis request"
  ],
  "state_snapshot": {
    "execution_context": {
      "agent_id": "l3_vic",
      "correlation_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
      "conversation_id": "conv-deepak-001",
      "parent_id": null
    },
    "domain_context": {
      "domain_type": "financial",
      "ticker_symbols": [],
      "has_portfolio_context": true
    },
    "agent_tier": "L3",
    "tool_call_history": {}
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

#### Receipt 3 — L3 executes the call

```json
{
  "ts": "2026-02-27T10:15:34.105Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "tool_execute",
  "invocation_specs": {
    "tool": "call_portfolio_analyzer",
    "kwargs": { "query": "Analyze the complete portfolio for investor Deepak Sharma..." }
  },
  "state_snapshot": { "timeout_s": 120.0 },
  "hitl_annotation": null,
  "reward_score": null
}
```

#### Receipt 4 — L2 Portfolio Analyzer starts

```json
{
  "ts": "2026-02-27T10:15:34.800Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l2_portfolio_analyzer::Portfolio Analyzer L2",
  "event_type": "task_start",
  "system_instructions": "You are the Portfolio Analyzer. You analyze investor portfolios by examining holdings, sector allocation, concentration risk, and performance...",
  "state_snapshot": {
    "query": "Analyze the complete portfolio for investor Deepak Sharma...",
    "context_id": "ctx-t1-002",
    "session_id": "sess-deepak-001"
  },
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t1-002"
}
```

#### Receipt 5 — L2 proposes calling L1

```json
{
  "ts": "2026-02-27T10:15:35.450Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l2_portfolio_analyzer::Portfolio Analyzer L2",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "portfolio_l1_agent",
    "kwargs": { "query": "Retrieve all portfolio holdings for investor Deepak Sharma with current quantities and values" }
  },
  "chain_of_thought": "I need to get the raw portfolio holdings data for Deepak Sharma before I can analyze allocation and risk. The L1 Portfolio Primitive Analyzer can fetch this data from the portfolio database.",
  "state_snapshot": {
    "execution_context": {
      "agent_id": "l2_portfolio_analyzer",
      "correlation_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
      "conversation_id": "conv-deepak-001",
      "parent_id": "l3_vic"
    },
    "agent_tier": "L2",
    "tool_call_history": {}
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

#### Receipt 6 — L2 executes L1 call

```json
{
  "ts": "2026-02-27T10:15:35.452Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l2_portfolio_analyzer::Portfolio Analyzer L2",
  "event_type": "tool_execute",
  "invocation_specs": { "tool": "portfolio_l1_agent", "kwargs": { "query": "Retrieve all portfolio holdings..." } },
  "state_snapshot": { "timeout_s": 90.0 },
  "hitl_annotation": null,
  "reward_score": null
}
```

#### Receipt 7 — L1 Portfolio Primitive starts

```json
{
  "ts": "2026-02-27T10:15:36.100Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l1_portfolio_primitive::Portfolio Primitive Analyzer L1",
  "event_type": "task_start",
  "system_instructions": "You are the Portfolio Primitive Analyzer. You retrieve raw portfolio data including holdings, quantities, purchase prices, and current values...",
  "state_snapshot": {
    "query": "Retrieve all portfolio holdings for investor Deepak Sharma...",
    "context_id": "ctx-t1-003"
  },
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t1-003"
}
```

#### Receipt 8 — L1 completes

```json
{
  "ts": "2026-02-27T10:15:42.550Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l1_portfolio_primitive::Portfolio Primitive Analyzer L1",
  "event_type": "task_complete",
  "latency_ms": 6450.00,
  "evidence": { "output_chars": 1850, "has_content": true },
  "success_label": "ok",
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t1-003"
}
```

#### Receipt 9 — L2 receives L1 result

```json
{
  "ts": "2026-02-27T10:15:42.555Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l2_portfolio_analyzer::Portfolio Analyzer L2",
  "event_type": "tool_complete",
  "invocation_specs": { "tool": "portfolio_l1_agent" },
  "latency_ms": 7103.00,
  "success_label": "ok",
  "evidence": {
    "result_chars": 1850,
    "result_preview": "Holdings for Deepak Sharma: RELIANCE.NSE 40.2% (₹8,04,000), INFY.NSE 25.1% (₹5,02,000), TCS.NSE 20.3% (₹4,06,000), HDFCBANK.NSE 14.4% (₹2,88,000). Total: ₹20,00,000"
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

#### Receipt 10 — L2 completes analysis

```json
{
  "ts": "2026-02-27T10:15:48.900Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l2_portfolio_analyzer::Portfolio Analyzer L2",
  "event_type": "task_complete",
  "latency_ms": 14100.00,
  "evidence": { "output_chars": 3200, "has_content": true },
  "success_label": "ok",
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t1-002"
}
```

#### Receipt 11 — L3 VIC delivers final answer

```json
{
  "ts": "2026-02-27T10:15:53.200Z",
  "trace_id": "t1-port-a1b2c3d4-5678-9abc-def0-111111111111",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "task_complete",
  "latency_ms": 19779.00,
  "evidence": { "output_chars": 4521, "has_content": true },
  "success_label": "ok",
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t1-001"
}
```

### Task 1 Timeline Bar

```
  TIME (seconds)   0s      5s      10s     15s     20s
                   │───────│───────│───────│───────│
                   │                                │
  L3 VIC           ████████████████████████████████████  task_start → task_complete
                   │  ▽ call_portfolio_analyzer      │   (19.8s total)
                   │                                 │
  L2 Portfolio     │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │   task_start → task_complete
                   │  ▽ portfolio_l1_agent           │   (14.1s total)
                   │                                 │
  L1 Primitive     │  ░░░░░░░░░░░░░░                 │   task_start → task_complete
                   │                                 │   (6.5s total)
                   │───────│───────│───────│───────│
                   0s      5s      10s     15s     20s
```

---

## 4. Task 2 — Market Impact on Portfolio

> **User query:** *"analyze Reliance stock from the market and tell if deepak sharma should do anything about it"*

### Conversation Flow

```
  ╔══════════════════════════════════════════════════════════════════════════════════╗
  ║  USER: "analyze Reliance stock from the market and tell if deepak sharma       ║
  ║         should do anything about it"                                            ║
  ╚══════════════════════════╤═════════════════════════════════════════════════════╝
                             │
                             ▼
  ┌────────────────────────────────────────────────────────────────────────────────┐
  │  L3 — Virtual Investment Concierge (VIC)                    trace: t2-mkt-    │
  │  ─────────────────────────────────────────────                                 │
  │                                                                                │
  │  📋 task_start                                                                 │
  │  │                                                                             │
  │  │  ┌─────────────────────────────────────────────────────────────────────┐    │
  │  │  │              STEP 1: MARKET ANALYSIS                                │    │
  │  │  └─────────────────────────────────────────────────────────────────────┘    │
  │  │                                                                             │
  │  🧠 tool_propose  →  call_financial_analyst                                    │
  │  │  chain_of_thought: "Two things needed:                                      │
  │  │    1) Market analysis of Reliance                                           │
  │  │    2) Deepak's portfolio to check exposure                                  │
  │  │    Get market data FIRST, then portfolio"                                   │
  │  │  discarded_paths: ["Considered calling portfolio_analyzer                   │
  │  │    first, but market context improves recommendation quality"]               │
  │  │                                                                             │
  │  🚀 tool_execute  →  call_financial_analyst                                    │
  │  │                                                                             │
  │  │  ┌──────────────────────────────────────────────────────────────────────┐   │
  │  │  │  L2 — Financial Analyst                                              │   │
  │  │  │  ──────────────────────                                              │   │
  │  │  │                                                                       │   │
  │  │  │  📋 task_start                                                        │   │
  │  │  │  │                                                                    │   │
  │  │  │  🧠 tool_propose  →  call_securities_analyst                          │   │
  │  │  │  🚀 tool_execute  →  call_securities_analyst                          │   │
  │  │  │  │                                                                    │   │
  │  │  │  │  ┌───────────────────────────────────────────────────────────┐    │   │
  │  │  │  │  │  L1 — Securities Research Analyst                         │    │   │
  │  │  │  │  │  ─────────────────────────────────                        │    │   │
  │  │  │  │  │                                                            │    │   │
  │  │  │  │  │  📋 task_start                                             │    │   │
  │  │  │  │  │  │                                                         │    │   │
  │  │  │  │  │  🧠 tool_propose  →  alpha_vantage_tool                    │    │   │
  │  │  │  │  │  🚀 tool_execute  →  alpha_vantage_tool  (timeout: 30s)    │    │   │
  │  │  │  │  │  ✅ tool_complete ←  alpha_vantage_tool  ⏱ 2,898ms         │    │   │
  │  │  │  │  │  │  evidence: RELIANCE.NSE ₹2498.75                        │    │   │
  │  │  │  │  │  │                                                         │    │   │
  │  │  │  │  │  🧠 tool_propose  →  marketaux_news                        │    │   │
  │  │  │  │  │  │  chain_of_thought: "[observation] Got price data.        │    │   │
  │  │  │  │  │  │   Now need news for sentiment context."                  │    │   │
  │  │  │  │  │  🚀 tool_execute  →  marketaux_news      (timeout: 30s)    │    │   │
  │  │  │  │  │  ✅ tool_complete ←  marketaux_news       ⏱ 3,698ms        │    │   │
  │  │  │  │  │  │  evidence: "Jio adds 3.2M subscribers"                  │    │   │
  │  │  │  │  │  │                                                         │    │   │
  │  │  │  │  │  ✅ task_complete   ⏱ 12,300ms                             │    │   │
  │  │  │  │  └───────────────────────────────────────────────────────────┘    │   │
  │  │  │  │                                                                    │   │
  │  │  │  ✅ tool_complete  ←  call_securities_analyst   ⏱ 13,008ms           │   │
  │  │  │  │  evidence: "RELIANCE.NSE ₹2498.75, PE 28.5, Jio growth +"        │   │
  │  │  │  │                                                                    │   │
  │  │  │  │  [ L2 synthesizes financial analysis ]                             │   │
  │  │  │  │                                                                    │   │
  │  │  │  ✅ task_complete   ⏱ 22,500ms                                       │   │
  │  │  └──────────────────────────────────────────────────────────────────────┘   │
  │  │                                                                             │
  │  ✅ tool_complete  ←  call_financial_analyst   ⏱ 23,177ms                     │
  │  │  evidence: "HOLD with slight overvaluation risk at current levels"          │
  │  │                                                                             │
  │  │  ┌─────────────────────────────────────────────────────────────────────┐    │
  │  │  │              STEP 2: PORTFOLIO CHECK                                │    │
  │  │  └─────────────────────────────────────────────────────────────────────┘    │
  │  │                                                                             │
  │  🧠 tool_propose  →  call_portfolio_analyzer                                   │
  │  │  chain_of_thought: "[observation] Reliance PE 28.5, overvalued.             │
  │  │   [thought] Need to check Deepak's Reliance exposure.                       │
  │  │   If high + overvalued → recommend trimming."                               │
  │  │                                                                             │
  │  🚀 tool_execute  →  call_portfolio_analyzer                                   │
  │  │                                                                             │
  │  │  ┌──────────────────────────────────────────────────────────────────────┐   │
  │  │  │  L2 — Portfolio Analyzer                                             │   │
  │  │  │  ──────────────────────                                              │   │
  │  │  │                                                                       │   │
  │  │  │  📋 task_start                                                        │   │
  │  │  │  │                                                                    │   │
  │  │  │  🧠 tool_propose  →  portfolio_l1_agent                               │   │
  │  │  │  🚀 tool_execute  →  portfolio_l1_agent                               │   │
  │  │  │  │                                                                    │   │
  │  │  │  │  ┌───────────────────────────────────────────────────────────┐    │   │
  │  │  │  │  │  L1 — Portfolio Primitive Analyzer                        │    │   │
  │  │  │  │  │  ─────────────────────────────────                        │    │   │
  │  │  │  │  │                                                            │    │   │
  │  │  │  │  │  📋 task_start                                             │    │   │
  │  │  │  │  │  │  Fetches Deepak's holdings                              │    │   │
  │  │  │  │  │  ✅ task_complete   ⏱ 6,400ms                              │    │   │
  │  │  │  │  └───────────────────────────────────────────────────────────┘    │   │
  │  │  │  │                                                                    │   │
  │  │  │  ✅ tool_complete  ←  portfolio_l1_agent   ⏱ 7,010ms                 │   │
  │  │  │  │  evidence: "RELIANCE 40.2% (₹8,04,000)"                           │   │
  │  │  │  │                                                                    │   │
  │  │  │  ✅ task_complete   ⏱ 13,500ms                                       │   │
  │  │  └──────────────────────────────────────────────────────────────────────┘   │
  │  │                                                                             │
  │  ✅ tool_complete  ←  call_portfolio_analyzer   ⏱ 14,208ms                    │
  │  │  evidence: "RELIANCE 40.2% — CONCENTRATION RISK HIGH"                      │
  │  │                                                                             │
  │  │  [ L3 combines market analysis + portfolio data ]                           │
  │  │  [ Recommendation: Trim RELIANCE from 40% to 25% ]                         │
  │  │                                                                             │
  │  ✅ task_complete   total_latency: 46,700ms                                    │
  └────────────────────────────────────────────────────────────────────────────────┘
                             │
                             ▼
  ╔══════════════════════════════════════════════════════════════════════════════════╗
  ║  RESPONSE: "Reliance Industries is trading at ₹2498.75 with a PE of 28.5      ║
  ║  (above sector average). Deepak Sharma has 40.2% portfolio exposure —          ║
  ║  exceeding the 35% concentration threshold. Recommendation: Consider           ║
  ║  trimming RELIANCE position to ~25% and diversifying into..."                   ║
  ╚══════════════════════════════════════════════════════════════════════════════════╝
```

### Task 2 Timeline Bar

```
  TIME (seconds)   0s         10s        20s        30s        40s        47s
                   │──────────│──────────│──────────│──────────│──────────│
                   │                                                      │
  L3 VIC           ██████████████████████████████████████████████████████████
                   │ STEP 1: Financial Analysis    │ STEP 2: Portfolio   │
                   │ ▽ call_financial_analyst       │ ▽ call_portfolio_   │
                   │                                │   analyzer          │
                   │                                │                     │
  L2 Financial     │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │                     │
  Analyst          │  ▽ call_securities_analyst     │                     │
                   │                                │                     │
  L2 Portfolio     │                                │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │
  Analyzer         │                                │ ▽ portfolio_l1_    │
                   │                                │                     │
  L1 Securities    │ ░░░░░░░░░░░░░░░░░░░░          │                     │
  Analyst          │ ▽alpha_vantage ▽marketaux_news │                     │
                   │                                │                     │
  L1 Portfolio     │                                │  ░░░░░░░░░░░       │
  Primitive        │                                │                     │
                   │──────────│──────────│──────────│──────────│──────────│
                   0s         10s        20s        30s        40s        47s
```

### Ledger Output — 27 Receipt Lines

#### STEP 1 — Market Analysis Chain (Receipts 1–17)

<details>
<summary><strong>Receipt 1 — L3 VIC task_start</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:01.100Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "task_start",
  "system_instructions": "You are the Virtual Investment Concierge (VIC), a top-tier financial advisor. You coordinate analysis by delegating to specialized sub-agents...",
  "state_snapshot": {
    "query": "analyze Reliance stock from the market and tell if deepak sharma should do anything about it",
    "context_id": "ctx-t2-001",
    "session_id": "sess-deepak-002"
  },
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t2-001"
}
```

</details>

<details>
<summary><strong>Receipt 2 — L3 tool_propose → call_financial_analyst</strong> 🧠</summary>

```json
{
  "ts": "2026-02-27T10:20:02.330Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "call_financial_analyst",
    "kwargs": {
      "query": "Provide a comprehensive market analysis of Reliance Industries (RELIANCE.NSE) including current valuation, recent price action, fundamentals, and market sentiment"
    }
  },
  "chain_of_thought": "The user wants two things: (1) a market analysis of Reliance stock, and (2) whether Deepak Sharma should adjust his portfolio. I need to handle these sequentially. First, I'll get the market analysis of Reliance from the Financial Analyst who can pull fundamentals and news. Then I'll check Deepak's portfolio to see his Reliance exposure. I should NOT call portfolio_analyzer first because I need the market context to make a recommendation.",
  "discarded_paths": [
    "Considered calling call_portfolio_analyzer first to check if Deepak even owns Reliance, but it's better to get the market analysis first since the user explicitly asked about the stock — and the recommendation quality will be higher if I have market data before assessing portfolio impact"
  ],
  "state_snapshot": {
    "execution_context": {
      "agent_id": "l3_vic",
      "correlation_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
      "conversation_id": "conv-deepak-002",
      "parent_id": null
    },
    "domain_context": {
      "domain_type": "financial",
      "ticker_symbols": ["RELIANCE"],
      "has_portfolio_context": true
    },
    "agent_tier": "L3",
    "tool_call_history": {}
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

</details>

<details>
<summary><strong>Receipt 3 — L3 tool_execute → call_financial_analyst</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:02.333Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "tool_execute",
  "invocation_specs": {
    "tool": "call_financial_analyst",
    "kwargs": { "query": "Provide a comprehensive market analysis of Reliance Industries..." }
  },
  "state_snapshot": { "timeout_s": 120.0 },
  "hitl_annotation": null,
  "reward_score": null
}
```

</details>

<details>
<summary><strong>Receipt 4 — L2 Financial Analyst task_start</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:03.000Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l2_financial_analyst::Financial Analyst L2",
  "event_type": "task_start",
  "system_instructions": "You are the Financial Analyst. You provide comprehensive financial analysis by examining fundamentals, technicals, market sentiment, and news...",
  "state_snapshot": {
    "query": "Provide a comprehensive market analysis of Reliance Industries...",
    "context_id": "ctx-t2-002"
  },
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t2-002"
}
```

</details>

<details>
<summary><strong>Receipts 5–6 — L2 proposes & executes call_securities_analyst</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:03.800Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l2_financial_analyst::Financial Analyst L2",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "call_securities_analyst",
    "kwargs": { "query": "Get current stock price, PE ratio, EPS, 52-week range, market cap, and recent price action for RELIANCE.NSE" }
  },
  "chain_of_thought": "I need fundamental market data for Reliance Industries. The Securities Analyst can pull real-time data from Alpha Vantage and news from MarketAux.",
  "state_snapshot": {
    "execution_context": { "agent_id": "l2_financial_analyst", "parent_id": "l3_vic" },
    "agent_tier": "L2",
    "tool_call_history": {}
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

```json
{
  "ts": "2026-02-27T10:20:03.802Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l2_financial_analyst::Financial Analyst L2",
  "event_type": "tool_execute",
  "invocation_specs": { "tool": "call_securities_analyst" },
  "state_snapshot": { "timeout_s": 120.0 },
  "hitl_annotation": null,
  "reward_score": null
}
```

</details>

<details>
<summary><strong>Receipt 7 — L1 Securities Analyst task_start</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:04.500Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l1_securities_analyst::Securities Research Analyst L1",
  "event_type": "task_start",
  "system_instructions": "You are the Securities Research Analyst. You retrieve raw market data using Alpha Vantage for stock prices and fundamentals, and MarketAux for financial news...",
  "state_snapshot": { "query": "Get current stock price, PE ratio... for RELIANCE.NSE" },
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t2-003"
}
```

</details>

<details>
<summary><strong>Receipts 8–10 — L1 calls alpha_vantage_tool (propose → execute → complete)</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:05.200Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l1_securities_analyst::Securities Research Analyst L1",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "alpha_vantage_tool",
    "kwargs": { "symbol": "RELIANCE.NSE", "function": "GLOBAL_QUOTE" }
  },
  "chain_of_thought": "I need the current stock price and basic quote data for Reliance. Alpha Vantage GLOBAL_QUOTE will give me latest price, change, volume.",
  "state_snapshot": { "agent_tier": "L1", "tool_call_history": {} },
  "hitl_annotation": null,
  "reward_score": null
}
```

```json
{
  "ts": "2026-02-27T10:20:05.202Z",
  "event_type": "tool_execute",
  "invocation_specs": { "tool": "alpha_vantage_tool" },
  "state_snapshot": { "timeout_s": 30.0 }
}
```

```json
{
  "ts": "2026-02-27T10:20:08.100Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l1_securities_analyst::Securities Research Analyst L1",
  "event_type": "tool_complete",
  "invocation_specs": { "tool": "alpha_vantage_tool" },
  "latency_ms": 2898.00,
  "success_label": "ok",
  "evidence": {
    "result_chars": 620,
    "result_preview": "{\"Global Quote\":{\"01. symbol\":\"RELIANCE.NSE\",\"05. price\":\"2498.75\",\"06. volume\":\"12453200\"}}"
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

</details>

<details>
<summary><strong>Receipts 11–13 — L1 calls marketaux_news (propose → execute → complete)</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:08.600Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l1_securities_analyst::Securities Research Analyst L1",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "marketaux_news",
    "kwargs": { "symbols": "RELIANCE.NSE", "filter_entities": true, "limit": 5 }
  },
  "chain_of_thought": "[observation] Alpha Vantage returned RELIANCE.NSE at ₹2498.75, up from open ₹2485. Volume 12.4M.\n[thought] I now have price data. I also need recent news to provide sentiment context. MarketAux will give me the latest headlines.",
  "state_snapshot": { "agent_tier": "L1", "tool_call_history": { "alpha_vantage_tool": 1 } },
  "hitl_annotation": null,
  "reward_score": null
}
```

```json
{
  "ts": "2026-02-27T10:20:08.602Z",
  "event_type": "tool_execute",
  "invocation_specs": { "tool": "marketaux_news" },
  "state_snapshot": { "timeout_s": 30.0 }
}
```

```json
{
  "ts": "2026-02-27T10:20:12.300Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l1_securities_analyst::Securities Research Analyst L1",
  "event_type": "tool_complete",
  "invocation_specs": { "tool": "marketaux_news" },
  "latency_ms": 3698.00,
  "success_label": "ok",
  "evidence": {
    "result_chars": 2100,
    "result_preview": "[{\"title\":\"Reliance Jio adds 3.2M subscribers in January\",\"sentiment\":\"positive\"}]"
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

</details>

<details>
<summary><strong>Receipts 14–17 — Results bubble back up (L1 → L2 → L3)</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:16.800Z",
  "agent_persona": "l1_securities_analyst::Securities Research Analyst L1",
  "event_type": "task_complete",
  "latency_ms": 12300.00,
  "evidence": { "output_chars": 3400, "has_content": true },
  "success_label": "ok",
  "task_id": "task-t2-003"
}
```

```json
{
  "ts": "2026-02-27T10:20:16.810Z",
  "agent_persona": "l2_financial_analyst::Financial Analyst L2",
  "event_type": "tool_complete",
  "invocation_specs": { "tool": "call_securities_analyst" },
  "latency_ms": 13008.00,
  "success_label": "ok",
  "evidence": {
    "result_chars": 3400,
    "result_preview": "RELIANCE.NSE: Price ₹2498.75 (+0.55%), PE 28.5, Market Cap ₹16.9L Cr."
  }
}
```

```json
{
  "ts": "2026-02-27T10:20:25.500Z",
  "agent_persona": "l2_financial_analyst::Financial Analyst L2",
  "event_type": "task_complete",
  "latency_ms": 22500.00,
  "evidence": { "output_chars": 5200, "has_content": true },
  "success_label": "ok",
  "task_id": "task-t2-002"
}
```

```json
{
  "ts": "2026-02-27T10:20:25.510Z",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "tool_complete",
  "invocation_specs": { "tool": "call_financial_analyst" },
  "latency_ms": 23177.00,
  "success_label": "ok",
  "evidence": {
    "result_chars": 5200,
    "result_preview": "Reliance Industries Analysis: Current Price ₹2498.75, PE 28.5. Overall: HOLD with slight overvaluation risk"
  }
}
```

</details>

#### STEP 2 — Portfolio Check Chain (Receipts 18–27)

<details>
<summary><strong>Receipt 18 — L3 tool_propose → call_portfolio_analyzer</strong> 🧠</summary>

```json
{
  "ts": "2026-02-27T10:20:26.200Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "tool_propose",
  "invocation_specs": {
    "tool": "call_portfolio_analyzer",
    "kwargs": {
      "query": "Retrieve and analyze Deepak Sharma's portfolio with focus on Reliance Industries exposure, allocation percentage, and concentration risk"
    }
  },
  "chain_of_thought": "[observation] Financial Analyst returned: Reliance at ₹2498.75, PE 28.5 (above sector average), slight overvaluation risk, HOLD recommendation.\n[thought] Now I need to check Deepak Sharma's portfolio to see how much Reliance he holds. If his exposure is high and the stock is slightly overvalued, I may need to recommend trimming.\n[action] call_portfolio_analyzer with Deepak Sharma + Reliance focus",
  "state_snapshot": {
    "agent_tier": "L3",
    "tool_call_history": { "call_financial_analyst": 1 }
  },
  "hitl_annotation": null,
  "reward_score": null
}
```

> **Key insight:** Notice how `chain_of_thought` now references the *result* from Step 1 (`[observation]`), then reasons about what to do next (`[thought]`), then states the action. This is the cognitive transparency that makes the Glass Box work.

</details>

<details>
<summary><strong>Receipts 19–26 — Portfolio chain (same pattern as Task 1)</strong></summary>

```json
{"ts":"2026-02-27T10:20:26.202Z","event_type":"tool_execute","invocation_specs":{"tool":"call_portfolio_analyzer"},"state_snapshot":{"timeout_s":120.0}}

{"ts":"2026-02-27T10:20:26.900Z","agent_persona":"l2_portfolio_analyzer::Portfolio Analyzer L2","event_type":"task_start","task_id":"task-t2-004"}

{"ts":"2026-02-27T10:20:27.600Z","agent_persona":"l2_portfolio_analyzer::Portfolio Analyzer L2","event_type":"tool_propose","invocation_specs":{"tool":"portfolio_l1_agent","kwargs":{"query":"Retrieve all holdings for Deepak Sharma"}},"chain_of_thought":"I need raw holdings to analyze Reliance exposure."}

{"ts":"2026-02-27T10:20:27.602Z","event_type":"tool_execute","invocation_specs":{"tool":"portfolio_l1_agent"},"state_snapshot":{"timeout_s":90.0}}

{"ts":"2026-02-27T10:20:28.200Z","agent_persona":"l1_portfolio_primitive::Portfolio Primitive Analyzer L1","event_type":"task_start","task_id":"task-t2-005"}

{"ts":"2026-02-27T10:20:34.600Z","agent_persona":"l1_portfolio_primitive::Portfolio Primitive Analyzer L1","event_type":"task_complete","latency_ms":6400.00,"success_label":"ok","task_id":"task-t2-005"}

{"ts":"2026-02-27T10:20:34.610Z","agent_persona":"l2_portfolio_analyzer::Portfolio Analyzer L2","event_type":"tool_complete","invocation_specs":{"tool":"portfolio_l1_agent"},"latency_ms":7010.00,"success_label":"ok","evidence":{"result_chars":1850,"result_preview":"RELIANCE.NSE 40.2% (₹8,04,000)"}}

{"ts":"2026-02-27T10:20:40.400Z","agent_persona":"l2_portfolio_analyzer::Portfolio Analyzer L2","event_type":"task_complete","latency_ms":13500.00,"success_label":"ok","task_id":"task-t2-004"}
```

</details>

<details>
<summary><strong>Receipt 27 — L3 VIC final task_complete</strong></summary>

```json
{
  "ts": "2026-02-27T10:20:47.800Z",
  "trace_id": "t2-mkt-b2c3d4e5-6789-abcd-ef01-222222222222",
  "agent_persona": "l3_vic::Virtual Investment Concierge L3",
  "event_type": "task_complete",
  "latency_ms": 46700.00,
  "state_snapshot": { "result_length": 6100, "context_id": "ctx-t2-001" },
  "evidence": { "output_chars": 6100, "has_content": true },
  "success_label": "ok",
  "hitl_annotation": null,
  "reward_score": null,
  "task_id": "task-t2-001"
}
```

</details>

---

## 5. Querying the Ledger

The `.ndjson` format is designed for Unix pipes, `jq`, and stream processing.

### Live Monitoring

```bash
# Watch events appear in real time as you chat with VIC
tail -f logs/governance_receipts.ndjson | jq '.'
```

### Filtering by Conversation

```bash
# All events from Task 1
jq 'select(.trace_id | startswith("t1-"))' logs/governance_receipts.ndjson

# All events from Task 2
jq 'select(.trace_id | startswith("t2-"))' logs/governance_receipts.ndjson
```

### Reading the Agent's Mind

```bash
# See what VIC was thinking before each tool call
jq 'select(.event_type=="tool_propose") | {
  agent: .agent_persona,
  tool:  .invocation_specs.tool,
  why:   .chain_of_thought,
  also_considered: .discarded_paths
}' logs/governance_receipts.ndjson
```

### Performance Analysis

```bash
# Find slow operations (> 10 seconds)
jq 'select(.latency_ms > 10000) | {
  agent: .agent_persona,
  event: .event_type,
  tool:  .invocation_specs.tool,
  ms:    .latency_ms
}' logs/governance_receipts.ndjson

# Average latency per tool
jq -s '[.[] | select(.latency_ms)] | group_by(.invocation_specs.tool) | 
  map({tool: .[0].invocation_specs.tool, avg_ms: (map(.latency_ms) | add / length)})' \
  logs/governance_receipts.ndjson
```

### Failure Analysis

```bash
# All failures and errors
jq 'select(.event_type=="task_fail" or .event_type=="tool_error") | {
  agent:    .agent_persona,
  category: .error_category,
  label:    .success_label
}' logs/governance_receipts.ndjson
```

### Event Count Summary

```bash
# Events per agent
jq -r '.agent_persona' logs/governance_receipts.ndjson | sort | uniq -c | sort -rn

# Events by type
jq -r '.event_type' logs/governance_receipts.ndjson | sort | uniq -c | sort -rn
```

---

## 6. Receipt Counts Comparison

```
                              Task 1           Task 2
                           (Portfolio)       (Market + Portfolio)
                          ─────────────     ──────────────────────

  Agents involved                3                   5
  Agent tiers hit            L3→L2→L1          L3→L2→L1→L2→L1

  ┌─────────────────┐
  │ task_start       │        3                   5
  │ tool_propose     │        2                   6
  │ tool_execute     │        2                   6
  │ tool_complete    │        2                   6
  │ tool_error       │        0                   0
  │ task_complete    │        3                   5
  │ task_fail        │        0                   0
  ├─────────────────┤
  │ TOTAL RECEIPTS   │       11                  27  ←  (was 28,
  └─────────────────┘                                  but L3's
                                                       2nd tool_
  End-to-end latency     ~19.8 seconds         ~46.7 seconds
                                                complete doubles
  External API calls          0                   2              
  (alpha_vantage,              (all via          (alpha_vantage   as receipt)
   marketaux_news)              A2A)             + marketaux)

  ── Receipts by Agent ──────────────────────────────────────────

  L3 VIC               █████ 5              ████████████ 9
  L2 Financial          —                   ████████ 6
  L2 Portfolio         ████ 4               ████████ 6       ← same sub-flow
  L1 Securities         —                   ███████ 6        ← 2 external tools
  L1 Primitive         ██ 2                 ██ 2
```

---

## Summary

The Glass Box Governance Ledger transforms **invisible multi-agent orchestration** into a **fully auditable, queryable decision trail**. For every user question:

1. **You know WHO decided** — `agent_persona` tracks every agent
2. **You know WHAT they did** — `event_type` + `invocation_specs` log every tool call
3. **You know WHY they did it** — `chain_of_thought` captures the LLM's actual reasoning
4. **You know WHAT ELSE they considered** — `discarded_paths` shows rejected alternatives
5. **You know HOW LONG it took** — `latency_ms` measures every operation
6. **You know IF it worked** — `success_label` + `evidence` + `error_category`
7. **You can train on it later** — `hitl_annotation` + `reward_score` are RLAIF-ready

> *"The best debugging tool is a complete record of every decision your AI ever made."*
