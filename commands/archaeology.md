Perform a full System Archaeology on `$ARGUMENTS` using these 7 patterns:

1. **Boundary Mapping** — List all entry points (HTTP routes, queue consumers, cron jobs, CLI commands) with file path and line number

2. **Dependency Excavation** — Map ALL dependencies both directions:
   - Outbound: HTTP clients, DB collections, Redis keys, queues this service produces to
   - Inbound: what other services/clients call this? (search for its URL/port in other repos)

3. **Data Flow Tracing** — Pick the top 2 core business entities and trace full journey:
   - Where does it enter? → how is it transformed at each step? → where persisted? → where does it exit or trigger downstream?
   - Show the exact struct/model at each stage

4. **Trust Boundary Analysis** — Identify all auth boundaries:
   - Which endpoints are public vs protected?
   - What auth mechanism? (JWT, API key, session)
   - Where is authorization enforced? (middleware, handler, service layer)
   - Flag any endpoints that look unprotected but shouldn't be

5. **Dead Code Detection** — Flag potentially unused code:
   - Endpoints registered but never called
   - Functions defined but never referenced
   - Env variables declared but unused
   - DB collections written but never read (or vice versa)

6. **Failure Mode Mapping** — Find all fragile spots:
   - External calls with no timeout or retry
   - Errors silently swallowed (missing error handling)
   - Single points of failure with no fallback
   - Queue consumers with no dead-letter handling
   - Rate each finding: high / medium / low severity

7. **Change Archaeology** — Read git history and code comments:
   - Files that change most frequently (hotspots)
   - TODO / FIXME / HACK comments with context
   - Any patterns that look like workarounds or tech debt

Then produce TWO output files:

---

### Output 1 — `archaeology_$ARGUMENTS.md` (produce this FIRST, before the HTML)

A quick-read Markdown summary with:

```
# System Archaeology — <project name>

## 🔴 Critical Findings
| Project | Pattern | Finding |
|---------|---------|---------|
| ...     | Auth    | ...     |

## 🟡 High Findings
...

## Boundary Map
| Method | Path | Handler | File:Line |
...

## Trust Boundaries
| Endpoint | Status | Gap |
...

## Top Failure Modes
...

## Tech Debt Highlights
...
```

Write this file immediately after all pattern agents complete — before synthesizing HTML. This gives a fast readable summary while the HTML renders.

---

### Output 2 — `archaeology_$ARGUMENTS.html` — single self-contained file with:

- **Tab 1 - Risk Scorecard**: Summary dashboard with green/yellow/red rating per pattern category
- **Tab 2 - Boundary Map**: All entry points in a table + visual diagram
- **Tab 3 - Dependency Graph**: SVG showing inbound/outbound connections with color-coded edge types
- **Tab 4 - Data Flows**: Step-by-step trace for each core entity with struct snapshots
- **Tab 5 - Trust Boundaries**: Table of endpoints with auth status, gaps highlighted in red
- **Tab 6 - Dead Code**: Findings table with file, line, confidence level
- **Tab 7 - Failure Modes**: Findings table with file, line, severity badge (high=red/medium=yellow/low=gray)
- **Tab 8 - Tech Debt**: Hotspots, TODOs, and workarounds with context

Tailwind CDN + vanilla JS only, no build step.
Dark mode toggle, mobile responsive.
Use a workflow with parallel agents — one agent per pattern for maximum depth.
