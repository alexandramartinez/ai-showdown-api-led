# AI Showdown: MuleSoft API-Led Connectivity

Three different AIs were given the **same prompt** and asked to build a MuleSoft **API-led connectivity** network (Experience → Process → System) for a **Customers + Orders** domain. This repo holds all three solutions side by side so you can compare how each one interpreted and executed the same brief.

## 📄 Start here

- **[The prompt they were all given →](instructions-given.md)** — the exact instructions (3 layers, 11 operations, mock/seed in the system layer, ≥3 APIs, latest Mule/Maven/DataWeave).
- **[The full comparison →](COMPARISON.md)** — architecture, API specs, operation coverage, error handling, mocking/seeding, versions, and a scorecard.

## 🤖 The three contestants

| | Model | Folder | What it built |
|---|---|---|---|
| 🟦 **Claude** | Opus 4.8, **MAX effort** | [`claude/`](claude/) | 4 apps (1 exp · 1 prc · 2 sys) with a **parent reactor POM** |
| 🟨 **CurieTech** | (not disclosed) | [`curie/`](curie/) | **5 apps** (1 exp · 2 prc · 2 sys) — per-entity split at both process + system layers |
| 🟩 **MuleSoft Vibes** | Sonnet (effort unknown) | [`vibes/`](vibes/) | 4 apps (1 exp · 1 prc · 2 sys) |

> **Model note:** Claude Code ran on **Opus 4.8 at MAX effort**. MuleSoft Vibes' UI showed it running on **Sonnet** (reasoning effort not shown). CurieTech AI did not surface which model/effort it used. Worth keeping in mind when reading the results — these weren't necessarily comparable tiers of compute.

## The task, in one line

Build the layered API network for these **11 operations** — list customers, one customer's details, a customer's orders, one order's details, create/edit customer, delete customer *only if no orders*, create/edit order, cancel order *(status change, not delete)*, delete order — with **mock + seeded data in the system layer** (no real DB yet).

## 🏁 Scorecard at a glance

| Dimension | 🟦 Claude | 🟨 CurieTech | 🟩 MuleSoft Vibes |
|---|---|---|---|
| # Apps | 4 | 5 | 4 |
| All 3 layers | ✅ | ✅ | ✅ |
| Parent/aggregator POM | ✅ | ❌ | ❌ |
| Contract-first (RAML + APIkit) | ✅ | ✅ | ❌ no specs |
| 11 operations implemented | ✅ 11/11 | ✅ 11/11 | ⚠️ coded, writes don't persist |
| Business rules (delete-guard, cancel≠delete) | ✅ | ✅ | ✅ (on read-only data) |
| Security | ❌ none | ❌ none | ❌ none |
| Error handling (code) | ✅ global, all apps | ✅ global + 502 gateway | ❌ none |
| Mock/seed persistence | ✅ File (survives restart) | ✅ ObjectStore (in-memory) | ❌ read-only |
| MUnit tests | ✅ 18 | ✅ 41 | ❌ 0 |
| Docs/rationale | ✅ plan + READMEs | ✅ per-app READMEs | ❌ none |
| Runtime version | 4.9.0 (oldest) | 4.11.3 / 4.9.3 (drift) | 4.11.2 (consistent) |

## TL;DR

### 🟦 Claude — *most engineering-disciplined*

**Pros**
- Contract-first (RAML + APIkit) with a documented design rationale
- Parent reactor POM → consistent versions across all apps
- All 11 operations work, and data **persists across restarts** (only one that does)
- Global error handling in every app; 18 MUnit tests

**Cons**
- Oldest runtime (Mule 4.9.0 — still valid, but behind)
- No security layer

### 🟨 CurieTech — *most complete + granular*

**Pros**
- Full 5-app per-entity split (process **and** system layers)
- All 11 operations working end-to-end
- Most tests (41 MUnit cases) and richest error handling (502 gateway semantics)
- Dedicated status-change endpoint (`PATCH .../status`)

**Cons**
- Version drift across its five apps — no central POM to govern them
- No security layer; data lost on restart (in-memory ObjectStore)

### 🟩 MuleSoft Vibes — *a runnable demo*

**Pros**
- Runs, and uses the freshest HTTP connector (1.11.3)
- Consistent runtime version across all apps
- All 11 operations coded with correct business rules

**Cons**
- Skips the API-led rigor: **no specs, no APIkit, no tests, no error handler, no docs**
- Writes don't persist (system layer is read-only)
- No security layer

> All three used real, released MuleSoft versions and Java 17; none used the true latest Mule 4.12.0. Full detail in **[COMPARISON.md](COMPARISON.md)**.

## Repo layout

```
ai-showdown-2/
├── instructions-given.md   ← the shared prompt
├── COMPARISON.md           ← detailed head-to-head analysis
├── README.md               ← you are here
├── claude/                 ← 🟦 Claude's solution (4 Mule apps + parent pom)
├── curie/                  ← 🟨 CurieTech's solution (5 Mule apps)
└── vibes/                  ← 🟩 MuleSoft Vibes' solution (4 Mule apps)
```
