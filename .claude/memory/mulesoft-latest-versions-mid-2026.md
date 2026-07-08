---
name: mulesoft-latest-versions-mid-2026
description: Verified latest GA MuleSoft component versions as of mid-2026, and a caution about how to check them
metadata:
  type: reference
---

Latest GA MuleSoft versions verified against docs.mulesoft.com release notes on 2026-07-07:
- **Mule Runtime**: 4.12.0 (Jun 2, 2026 GA, DataWeave 2.12.0). Latest 4.11.x patch = 4.11.5 (Jun 2026). Lineage: 4.9.0 (Feb 6 2025) → 4.11.0 (Feb 4 2026) → 4.11.1/2/3/4/5 → 4.12.0.
- **DataWeave**: 2.12.0 (ships with the runtime; do not pin separately).
- **mule-maven-plugin**: 4.10.0 (lineage …4.7.0, 4.8.0, 4.9.0, 4.9.1, 4.10.0).
- **APIkit module**: 1.12.1 (1.12.0 also current; full 1.11.0–1.11.18 line exists).
- **HTTP connector**: 1.11.3 (May 14, 2026).
- **MUnit**: 3.7.1 (current list shows 3.6.0–3.7.1).
- **Java**: 17 (required by Mule 4.9+).

**Why / caution:** A subagent wrongly reported that 4.11.1/4.11.2/4.11.3, APIkit 1.10.4/1.11.12, and MUnit 3.4.0 "don't exist" — it inferred non-existence from HTTP 404s on per-patch doc URLs. Those versions ARE real.
**How to apply:** Never conclude a version doesn't exist from a 404 on a per-patch release-notes URL. Check the component's release-notes INDEX page (e.g. `/release-notes/apikit/apikit-release-notes`, `/release-notes/munit/munit-release-notes`) which lists the full lineage. MuleSoft artifacts resolve from MuleSoft repos, NOT Maven Central — Maven Central searches return junk for these. Related: [[ai-showdown-mule-comparison]].
