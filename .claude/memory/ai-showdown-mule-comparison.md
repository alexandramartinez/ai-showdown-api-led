---
name: ai-showdown-mule-comparison
description: What the ai-showdown-2 repo is — 3 AIs' MuleSoft API-led solutions being compared — and the headline findings per AI
metadata:
  type: project
---

The `ai-showdown-2` repo compares MuleSoft API-led solutions built by 3 AIs from the same prompt (`instructions-given.md`: Customers+Orders, 3 layers, 11 CRUD ops, mock/seed in system layer, ≥3 APIs, latest Mule/Maven/DW). Folders: `claude/`, `curie/` (CurieTech AI), `vibes/` (MuleSoft Vibes).

Headline findings (verified 2026-07-07):
- **claude/** — 4 apps (1 exp/1 prc/2 sys), parent reactor POM, RAML+APIkit, 18 MUnit tests, all 11 ops functional & persist (File connector, mutations survive restart), best docs. Weakness: oldest runtime **Mule 4.9.0**.
- **curie/** — intended 5 apps but **sys-orders-api was never built** (errored out) → 7/11 ops fail at runtime (process layer dangles against :8082). No parent POM, version drift across apps, uses **ObjectStore connector** (customers only). Most tests (33) + best error handling (502 gateway). Runtime 4.11.3.
- **vibes/** — 4 apps but **no API spec, no APIkit, no MUnit tests, no global error handler, no README**; system layer reads static JSON via `readUrl` so **writes don't persist**. Runtime 4.11.2, but used the *latest* HTTP connector 1.11.3.
- All 3: **zero security** added; all Java 17; all let DataWeave ship with runtime (none pinned DW).

**Why:** user is running an evaluation ("showdown") and will likely ask follow-ups per AI.
**How to apply:** treat each folder as an independent solution; when asked about versions, VERIFY against docs.mulesoft.com release-notes pages — see [[mulesoft-latest-versions-mid-2026]].
