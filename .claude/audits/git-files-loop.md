# Audit-and-refine loop — Nubi (master charter)

**Loop window:** 2026-06-10 15:56 → 19:58 SAST (4h). One cron job fires every 15 min and
re-enters this charter. This file is the ANCHOR — every wakeup reads it FIRST and must not
deviate from the scope/constraints here.

**On each iteration:**
1. Read this file. Read the iteration log so you don't redo finished work.
2. If a prior audit Workflow has completed, fold its findings into the Status table and the
   per-area gap lists (below), then dispatch FIX agents for the highest-severity gaps.
3. Otherwise audit the next not-started areas.
4. Run the relevant tests. Update the Status table + append an iteration-log entry.
5. **If now > 19:58 SAST:** CronList → CronDelete the loop job, write a final summary at the
   bottom, report. Otherwise end the turn (cron carries the next fire).

## PROCESS CONSTRAINTS (hard — the user set these)
- **Few agents, not many.** Cap concurrent agents at **≤6** per Workflow. Prefer focused
  fan-out over breadth. One audit Workflow OR one fix Workflow in flight at a time — never
  spawn a second while one is running (token + worktree contention).
- **Git worktrees for independence.** Any agent that MUTATES files runs with
  `isolation: 'worktree'` so parallel fixes don't collide. Read-only audit agents do not need
  worktrees. After a fix Workflow returns, the orchestrator reviews diffs and merges the
  worktree branches back to `main` (the feature branch was merged to main ~16:10; `main` is now the base);
  note any conflicts in the log rather than force-merging.
- **Stay on charter.** New ideas get written into "Backlog / ideas" below, not acted on,
  unless they are clearly in-scope of an existing area.
- **Tests gate.** Backend: `cd backend && python -m pytest -q`. Frontend build:
  `npm run build`. Never mark an area done with failing tests it touched.

## MACRO VISION (the "why" — keep every fix aligned to this)
Nubi should serve **three personas over the same underlying flow/query model**, so a user
picks the surface that fits their brain, not a different product:
1. **File/folder people** — everything is a folder tree of plain files (`.sql`, `.py`, `.md`,
   `.toml`, `.json`); edit in their own editor, git-sync round-trips. (Fabric/dbt-style repo.)
2. **Notebook people** — the same flow as ordered cells in a notebook UI (run, inspect,
   variables between cells). (Hex/Deepnote-style.)
3. **React-flow / canvas people** — the same DAG as a visual node graph. (Prefect/n8n-style.)
The three MUST be projections of one canonical model (cells + edges + settings), so a flow
authored in any surface is faithfully editable in the other two and on disk.
**Competitive parity target:** Microsoft Fabric + SQLMesh + Prefect — i.e. orchestration with
logging, scheduling, triggers, retries, environments/versioning, and incremental models.

## WORKSTREAMS

### A. Git + files + sync-everything  (original scope)
1. Git integration quality; is the panel in the **RHS sidebar**; reachable from **dashboards
   AND queries AND flows** (not just some). Consistent component/affordances.
2. Sync **everything** to files, bidirectional, with the canonical layout below.
3. **Flows-as-files**: each cell is a file — python→`.py`, sql→`.sql`, note→`.md`; round-trips
   preserving cell id/order/settings via `flows/<flow>/flow.toml`.
4. **Output-shape contract**: `queries/<name>.json` (same basename) declares output cols+types;
   validated after execution.
5. **Long-term variables**: org+project-scoped typed store; Python `nubi.vars.set/get`,
   SQL `{{ vars.key }}`; run-scoped overlay, `persist=True` survives a run; synced to
   `variables.json`. SQL templating parity with query named-params (jinja-like).
6. **nubi.toml** project config synced; **secrets NEVER in files** — referenced by name only,
   resolved server-side. (HIGH severity if any value leaks — audit env_sync.py.)
7. **File view in portal** — browse the synced tree read-only first.
8. **Pre-run compute estimate** per connector (BigQuery dry_run bytes; DuckDB/Postgres/
   Snowflake EXPLAIN where possible; "unsupported" otherwise) shown before Run.
9. **Logging** — one efficient logs table (org_id, run_id, cell_id, ts, level, stream, msg),
   batched, indexed (run_id, ts); per-flow + per-cell UI tail.
10. **Run info** — runs list, per-run timeline, per-cell status/duration/log link, vars snapshot.

Canonical git layout:
`nubi.toml` · `queries/<name>.sql` + `<name>.json` · `flows/<flow>/flow.toml` +
`cells/NN_<name>.{py,sql,md}` · `dashboards/<name>.html` · `variables.json` · `environments/…`

### B. Security audit + light pen-testing
- AuthN/AuthZ on every route (org scoping, IDOR/cross-org access), JWT/RLS correctness,
  predicate injection (AST not string-concat), connector capability gate, secret handling,
  SSRF via connectors/httpfs, file-path traversal in sync/file-view, SQL injection surfaces,
  rate-limiting, CORS, dependency CVEs. Produce concrete exploit sketches + fixes; FIX the
  high/criticals. Use the `security-review` skill where useful. NON-DESTRUCTIVE only.

### C. Frontend audit
- Component quality, dead code, a11y (labels/roles/contrast/focus), error/loading/empty
  states, dark-mode contrast regressions, bundle size (build warns >500kB — code-split),
  consistency of the git/file/flow surfaces.

### D. Responsive audit
- Landing + app shell + flows/notebook/canvas + queries + dashboards at mobile (375),
  tablet (768), desktop (1280/1440). Screenshot via headless Chrome, find overflow/cramping/
  broken layouts, fix.

### E. Docs audit
- `docs/` accuracy vs code (drift), coverage of the new features (flows-as-files, variables,
  estimates, logging, file view), the three-persona story, getting-started correctness.

## Status

| # | Area | Status | Notes |
|---|------|--------|-------|
| A1 | Git audit (RHS sidebar / dashboards+queries+flows / full sync) | audited | NOT in RHS sidebar (only ChatPanel is); git UI only in ProjectSettings + a buried env-dropdown icon. Fix pending. |
| A2 | Sync everything, bidirectional | DONE-partial | Registry now drives pull: added `flow` KindHandler + `folder` field; pull_project iterates KIND_REGISTRY so FLOWS NOW IMPORT (was push-only) with hard-validation skip. Remaining: converge env_sync JSON-by-uuid vs YAML-by-slug, deletion propagation, envs/datasets/automations handlers. |
| A3 | Flow cells as .py/.sql/.md files | DONE-core | Shipped backend/app/git/flow_files.py: serialize_flow_files/load_flow_files — flows/<slug>__<id8>/flow.toml + cells/NN_<key>.{sql,py,md}, ui→[layout], lossless round-trip (4 tests pass). Remaining: wire into env_sync/project-remote write+read paths + frontend file editing. |
| A4 | Output-shape JSON contract | audited | No output schema anywhere. Plan: add output_schema to RegisteredQuery + validate post-exec + queries/<n>.json. |
| A5 | Variables store + query+flow templating DONE | store/CRUD + {{vars.*}} in /query (slice 2) + flow SQL cells (slice 3-read) | Store/CRUD merged. Slice 2: {{vars.*}} bound in /query (cache+injection-safe). Slice 3-READ DONE (me): TaskContext.vars field + "vars" in _BOUND_NAMESPACES/_resolve_native/_resolve_value; shared load_vars_namespace() in vars/store.py wired at the 2 durable runtime sites + the async preview route (org_id added there too); the SYNC runtime preview helper left vars={} (can't await — async route covers interactive). +2 flow-bind tests; full suite 3224 pass. Slice 3-READ-py DONE (me): python cells now read a `vars` dict (injected in the registry.py wrapper alongside inputs/params/secrets) +1 sandbox test; suite 3225 pass. NEXT (slice 3-WRITE): Python set_var + run-scoped overlay (persist flag); then sync vars→variables.json (git). |
| A6 | nubi.toml (secrets excluded) | audited | No nubi.toml; secrets hygiene GOOD (no value leak found) but no inline-credential scan before commit. |
| A7 | File view in portal | DONE-partial | Backend endpoints shipped (worktree agent, merged): GET /projects/{id}/git/files + /git/files/content on environments.py with path-traversal guards (.. / abs / unknown-folder → 400; missing → 404), 11 tests. Remaining: frontend GitFilesPanel.jsx + gitenv.js helpers. |
| A8 | Compute estimate per connector | DONE-partial | Scaffolding shipped: QueryEstimate model + Connector.estimate()->None default + BigQuery dry_run (exact bytes) + DuckDB EXPLAIN (~rows) + 3 tests. Remaining: Postgres/Snowflake/etc EXPLAIN impls + /query/estimate endpoint (needs query() refactor) + UI chip. |
| A9 | Logging store + UI | DONE-partial | Preview now returns target-cell logs (fixed). Remaining: 0007_logs table, batched sink, tail API, live flush. |
| A10 | Run info | DONE-partial | Fixed: env+duration_s in serializer, Automations newest-first. Remaining: spec snapshot, pagination, cancel, org-wide runs. |
| B | Security audit | B1,B2-int,B3,B4(http+httpfs),B6-read,B7 FIXED | HIGH: B1 SQL-injection ✅, B2-interactive RLS ✅. MED/LOW: B3 secret quote-escaping ✅, B4 SSRF — http_json ✅ AND httpfs/s3-endpoint ✅ (guard_s3_endpoint blocks metadata/link-local only; private/MinIO stays legal; +5 tests), B6 GitSync.read ✅, B7 hmac tick-secret ✅. **OPEN:** B2-SCHEDULED (design Q for user); B5 git-token-in-argv (needs real-remote test); rate-limiting; /git/restore route guard (marginal — GitSync.read already hardened). |
| C | Frontend audit | HIGH FIXED | Code-splitting DONE: App.jsx lazy-loads all authed/heavy pages (+ public Dashboard/Editor) behind one Suspense boundary; vite manualChunks splits echarts/reactflow/arrow/duckdb/monaco into vendor chunks. Entry chunk 4382kB→1893kB (~57%↓); echarts(1MB)/reactflow/duckdb/arrow OFF the landing path. Build green; smoke-verified landing + a lazy route render. MED remaining: git UI not RHS (=A1); modal focus-trap. LOW: dead files (Chart/Playground/Dashboard/QueryCell.jsx). Dark contrast PASSES AA (closed). |
| D | Responsive audit | DONE-partial | FIXED the one real bug: pricing grid now lg:grid-cols-3 + 2xl:grid-cols-5 with 2xl:max-w-[110rem] container (cards no longer shrink as screen grows). Build green. LOW remaining: compare-table scroll affordance. |
| E | Docs audit | DONE | FIXED+MERGED (worktree agent): git-sync.md (file-view endpoints + 3-layout reconciliation + flows-on-disk), flows.md (one-flow-three-views), connectors.md + connector-security.md (estimate), dashboards.md (tabs + scan/slice). 5 docs updated, accurate to code. |

### Highest-severity gaps (raw audit: /private/tmp/claude-501/.../tasks/wuw7v8ekk.output, 671 lines)
**HIGH:**
- Three divergent git serializers (portability.KIND_REGISTRY vs sync.KIND_FOLDER vs env_sync) with two on-disk
  formats (JSON-by-uuid vs YAML-by-slug) → consolidate on ONE KindHandler registry. (A2)
- Flows push but never pull in project sync; KIND_REGISTRY has no 'flow' handler. (A2/A3)
- Cells are one JSON blob, not per-cell files → unusable git diffs. Layout: flows/<slug>__<id8>/flow.toml +
  cells/NN_<key>.{sql,py,md}; flow.toml is order/id source of truth; ui coords → separate layout.toml. (A3)
- Git panel absent from dashboards/queries/flows; NO right-hand git sidebar; no status/diff before push. (A1)
- Flow SQL cells string-interpolate {{params}} into SQL — contradicts the never-string-concat guarantee;
  route the `sql` key through the bound-param Jinja engine instead. (A5, also a security item → B)
- No {{vars.*}} namespace / no persistent variable store (migration 0007 core, not ee). (A5)
- No output_schema declaration/validation; a column rename ships silently to every embed. (A4)
- Connector ABC has no estimate(); add default-None method (do NOT touch the strict 7-key capabilities dict);
  BigQuery dry_run is the biggest cost-safety win. (A8)
- Logs persist only at task completion → running task shows zero logs; python stderr lost on success,
  stdout lost on failure. Add 0007_logs table + batched LogSink + tail API. (A9)
- No spec snapshot on flow_run → historical runs render the CURRENT draft DAG (wrong). (A10)
- GET /flows/{id}/runs returns ALL runs, no pagination. (A10)
**MEDIUM/notable:** project pull is upsert-only (no deletion propagation, manifest never read); env pull drops
file deletions; force-with-lease likely always fails (no tracking ref); git token visible in subprocess argv;
git workspace defaults to world-readable tempdir; no nubi.toml manifest in env layout; no inline-secret scrub
before commit; dataset schema re-inferred with no drift detection; run-detail polling re-ships all logs+results
every 1.5s; no run cancellation (state exists, nothing sets it); no org-wide runs page.

### B/C/D/E audit — highest-severity (raw: tasks/wf457gthc.output, 278 lines)
**B SECURITY (2 HIGH — cross-tenant exfiltration path in flows):**
- B1 [HIGH] SQL injection: flow SQL cells string-interpolate attacker-controlled `{{params}}` raw into `sql` before
  sqlglot.parse (executor.py:228-248,555); RLS added via tree.where() only attaches to outer SELECT → `params={"region":
  "x' UNION SELECT secret FROM other_tenant --"}` exfiltrates. Fix: bind params positionally (route through
  resolve_named_params/render_sql_template like /query), don't interpolate `sql`. → worktree agent (executor.py/planner.py).
- B2 [HIGH] RLS never applied to flow query cells: every path hardcodes claims["policies"]={} (flows.py:630,1096,1265,1644;
  runtime.py; flows_tick claims=None) → flow query cells read ALL rows. Fix: thread caller identity.policies for INTERACTIVE
  runs (safe); SCHEDULED runs (claims=None) need a stored owner/service policy context = DESIGN DECISION (don't rush). → me.
- B3-B7 [MED/LOW]: DuckDB CREATE SECRET f-string quoting (escape quotes); no SSRF allowlist (http_json/httpfs → metadata IP);
  git token in subprocess argv + world-readable /tmp workspace; /git/restore unvalidated path (apply _validate_git_file_path);
  no rate limiting + non-constant-time tick-secret (use hmac.compare_digest).
**C FRONTEND:** [HIGH] no code-splitting → React.lazy(App.jsx) + vite manualChunks (echarts/reactflow/arrow/duckdb-wasm/monaco).
  [MED] git UI not RHS sidebar (=A1); modal focus-trap missing. [LOW] dead files (Chart/Playground/Dashboard/QueryCell.jsx);
  dark-contrast PASSES AA (non-finding, closed). 
**D RESPONSIVE:** [MED] pricing xl:grid-cols-5 narrows cards at 1280. [LOW] compare-table scroll affordance. Tooling note:
  use Playwright explicit viewport, not --headless --window-size (false clipping).
**E DOCS:** [HIGH] flows-as-files undocumented + 3 competing layouts (.yaml push / .json version-pin / .toml new-unwired) —
  reconcile + document. [MED] file-view endpoints, Connector.estimate, dashboard tabs/drawer/scan-slice, three-persona story
  all undocumented. → worktree agent.

## Macro / three-persona alignment check (revisit each iteration)
- [ ] Canonical flow model identified (cells + edges + settings) — single source of truth?
- [ ] File projection round-trips (A3)
- [ ] Notebook projection faithful
- [ ] Canvas/react-flow projection faithful
- [ ] Scheduling + triggers + retries present (Prefect parity)
- [ ] Incremental/environments/versioning present (SQLMesh/Fabric parity)

## Macro/persona read from the audit
- **Canonical model EXISTS and is sound**: flow = TaskSpec cells + edges + settings in one spec; a notebook IS a
  flow (notebook_to_flowspec/flowspec_to_notebook round-trips losslessly). So the **three personas are achievable
  as projections** — the gap is the FILE projection (A3 per-cell files) and a faithful canvas/notebook parity check,
  not a model rewrite. This is the single most leveraged area: nail flows-as-files and the file/folder persona lands.
- **Prefect parity**: runtime has retries/attempts/leases/map-fanout/branch/timeouts/watermarks + scheduler tick.
  Missing for parity: run cancellation, live logs, org-wide runs view, triggers surfacing. (workstream A9/A10 + later)
- **SQLMesh/Fabric parity**: environments + versioning + checkpoint/promote exist (env-branch layer). Missing:
  spec snapshot per run, incremental-model output contracts (ties to A4 output_schema + cell_materialize typing).

## Fix sequencing (do in this order across iterations — dependency-aware)
1. ~~Safe bug fixes (run env/duration, automations order, preview logs)~~ DONE iter 2.
2. **Connector estimate scaffolding** (A8) — additive, isolated, default-None safe. Good first worktree agent.
3. ~~Registry-driven pull + flow handler~~ DONE iter 4 (flows now round-trip; folder map centralised). Remaining
   under A2: full env_sync/project-remote format convergence + deletion propagation (larger, later).
4. **Flows-as-files per-cell layout** (A3) on top of the registry. 
5. **Variables store + {{vars.*}} + bound-param SQL fix** (A5) — migration 0007 + vars/store + template wiring.
6. **Output_schema contract** (A4), **nubi.toml + secret-scrub** (A6), **logs table + tail** (A9),
   **run spec-snapshot + pagination + cancel + org runs** (A10), **file-view endpoints+panel** (A7),
   **RHS git sidebar across pages** (A1).
7. Then workstreams B (security/pentest), C (frontend), D (responsive), E (docs).

## NEW WORKSTREAM F — Dashboard reactive filter graph + pre-agg (user, ~16:20)
Goal: a **simple, efficient dependency-graph linking system** for dashboard filters/variables, replacing
the legacy rendering engine (legacy `src/contexts`, `slices`, `store` folders — DO NOT COPY, study only;
they are ugly/over-complex). Requirements:
- Dashboard has **variables**; widgets/queries **link** to them and re-render efficiently when they change.
- **Filter→filter dependencies**: a filter's options can depend on ANOTHER filter's output (e.g. country→city);
  dependents must **re-fire** to fetch new data when their upstream changes. Common case — first-class, not bolted on.
- Handle **circular-dependency detection/filtering** (cycle in the filter graph must be detected and broken,
  not infinite-loop). "circular dependant filtering and filtering in general."
- **Pre-aggregation** to make this cheap, but the pre-agg system must be **generic enough** to cover the
  dependent-filter case (e.g. distinct-values / cascading-filter option queries get pre-agged too), tying into
  the existing rollup/pre-agg story (docs/pre-aggregations.md) and edge cache.
Design seed (to validate in audit): model dashboard as a **reactive DAG**: nodes = {variables, filter-option
queries, widget queries}; edges = `{{vars.*}}`/filter references (reuse the A5 vars namespace + template parser
to EXTRACT the dependency edges statically). On a variable change, topologically walk only the dirty subgraph,
dedupe identical option-queries, batch, and serve option-lists from a pre-agg/edge-cache keyed on
(query + upstream-filter values + RLS claims). Detect cycles at graph-build (Kahn/DFS) and surface a clear error.
This UNIFIES with: A5 (variables/templating provide the edge syntax), A8 (estimate gates expensive option
refetches), edge-cache + pre-agg (the collapse mechanism). Audit task added; do AFTER the A-stream serializer/
vars foundations so the edge syntax is settled.

## NEW WORKSTREAM G — Client-Compute Plan (user, ~16:45; spec: CLIENT_COMPUTE_PLAN.md, 237 lines)
Make the "no viewer tax" wedge real: move slice-type interactions to the browser.
- **G-P1 backend seed DONE** (worktree agent, merged): Variable.`mode: 'scan'|'slice'|None` field on
  backend/app/dashboards/spec.py (None≡scan, pure metadata, legacy-safe) + tests. Remaining: useWidgetData hook,
  client slice recompute, run-location badge, budget guardrails (all frontend).
- **G-P1 (Phase 1)**: param classes `scan` vs `slice` (slice never hits server — region/status/top-N/sort
  recompute on the already-fetched base result in DuckDB-WASM); shared `useWidgetData(query_id, params)` hook
  with in-flight coalescing (two widgets sharing query_id+scan params → ONE POST); run-location badge
  (local/edge/server/sample); client budget guardrails (register base only if ≤64MB; halve on deviceMemory≤4;
  256MB/dashboard LRU). Acceptance: slice change → 0 requests; killing backend keeps slice working; legacy specs
  (no `mode`) byte-identical. Spec field: add `mode` to dashboard param/Variable (backend/app/dashboards/spec.py).
- G-P2 browser result cache (returning viewers); G-P3 pinned extracts (client_mode rollups); G-P4 local re-agg.
  Out of scope: per-query toggle, WASM httpfs, multithreaded WASM, mixing local+server tables.

## NEW WORKSTREAM H — Dashboard Tabs + Filter/Variable overhaul (user, ~16:45; spec: DASHBOARD_TABS_AND_FILTERS_IMPLEMENTATION.md, 241 lines)
NO DB migration — all additive spec JSON. Ties into workstream F (reactive filter graph).
- **Track T (Tabs)**: T1 spec model DONE (worktree agent, merged): `Tab` model + `spec.tabs` + `widget.tab_id` on
  backend/app/dashboards/spec.py with dup-id & undeclared-tab_id hard errors, drawers exempt, tab_id None → first tab.
  Remaining T2-T6: TabBar.jsx, SpecRenderer partition, _tab URL param, EditorPage tab strip, e2e.
- (legacy) T1 spec model `spec.tabs` + `widget.tab_id` (backend/app/dashboards/spec.py; dup-id & undeclared-tab_id hard errors; drawers stay global); T2 TabBar.jsx (role=tablist, arrow-key nav, variants, tabStyleToCss via sanitized path); T3 SpecRenderer partitions by active tab; T4 `_tab` URL param; T5 EditorPage tab strip (add/rename/reorder/delete, move-widget-to-tab); T6 tests.
- **Track F (Filter overhaul)**: F1 input primitives (src/dashboards/inputs/ NEW) + fix dropdown clipping
  (SpecRenderer overflow-hidden ~L459); query-backed autocomplete; select-all/exclude ("all but selected");
  date presets. This IS the concrete impl of workstream F's cascading-filter UX.

## Backlog / ideas (captured, not yet acted on)
- Deprecate legacy org-level /git/sync,/history,/restore (nested-repo footgun) once project layer covers it.

## Iteration log
- **15:56 SAST · iter 1** — Wrote charter. Launched read-only audit Workflow `wuw7v8ekk`
  (10 agents) over workstream A (git, files, flows, vars, output, config/secrets, file-view,
  estimates, logging, run-info). Scheduled cron `09ce2e0c` @ every 15 min. Pending: fold A
  findings → dispatch worktree fix agents; then audit B/C/D/E.
- **15:58 SAST · charter expansion** — User broadened scope: added B/C/D/E, three-persona vision,
  Fabric/SQLMesh/Prefect parity, process constraints (≤6 agents, worktrees, stay on charter).
- **~16:06 SAST · iter 2** — Audit wf `wuw7v8ekk` completed (10 agents, ~638k tok, 671-line report).
  Folded all findings into Status table + highest-severity list + fix sequencing above. Made 3 safe
  high-confidence fixes DIRECTLY (disjoint files, no worktree needed): (a) `_serialize_flow_run` now
  returns `env` + run-level `duration_s` (routes/flows.py); (b) AutomationsPage run list newest-first
  (`runs.slice(0,20)`, was `.reverse().slice` showing 20 oldest); (c) `/flows/preview` now returns the
  target cell's redacted `logs` so notebook print() debugging is visible. **Tests: backend pytest
  823 passed / 1 skip (97s); `npm run build` green.** Left in working tree (uncommitted, like the
  illustration work) — user commits when ready.
  **Next up (iter 3, ~16:11):** start fix-sequence step 2 — connector estimate scaffolding (A8) as a
  worktree fix agent (additive, default-None, isolated). Then step 3 (unify git serializers) which
  unblocks A3/A4/A6. Keep ONE workflow in flight; run tests each time.
- **16:14 SAST · iter 3** — Noted the working tree moved to `main` (user merged feat/flows-cells-and-connectors
  ~16:10); all prior WIP now committed, only iter-2 edits + .claude/audits/ uncommitted. Work intact. Updated
  merge-target → `main`. Shipped A8 connector-estimate scaffolding DIRECTLY (single coherent additive chunk;
  worktree reserved for genuine parallel fan-out — one agent doesn't need isolation): added `QueryEstimate`
  (connectors/plan.py), default-None `Connector.estimate()` (base.py, deliberately NOT an 8th capabilities key),
  BigQuery dry-run exact-bytes (bigquery.py), DuckDB EXPLAIN ~rows (duckdb_conn.py), + new
  tests/test_connector_estimate.py (3 tests). **Tests: new 3/3 pass; connector+conformance+plan suite 395 passed
  / 2 skip — no regressions.** No frontend touched. Deferred: per-connector EXPLAIN (Postgres/Snowflake/…),
  /query/estimate endpoint (needs query() helper extraction), UI estimate chip.
  **NEW user request → workstream F** (dashboard reactive filter-graph + circular-dep detection + dependent-filter
  refire + generic pre-agg) added to charter; sequenced AFTER A5 vars/templating so the edge syntax is settled.
  **Next up (iter 4, ~16:26):** fix-sequence step 3 — unify the three git serializers into one KindHandler
  registry (A2). This is the central-file refactor that unblocks A3/A4/A6; do as ONE focused worktree agent
  (not parallel — single hot file), then bring changes into the working tree and run pytest.
- **16:27 SAST · iter 4** — Did the A2 registry-unification slice DIRECTLY (A-stream refactors are strictly
  sequential serializer→flows-as-files→vars, so they don't parallelize; worktree fan-out is reserved for the
  independent B/C/D/E work). Added `_flow_spec_from_row/_flow_row_fields/_flow_validate` + a `flow` KindHandler
  and a `folder` field to KindHandler (portability.py); rewrote `pull_project` (routes/git.py) to iterate
  KIND_REGISTRY by folder — **FLOWS NOW IMPORT on pull (were push-only — the HIGH asymmetric-round-trip gap)**,
  with hard-error-only validation skip (soft `[warn]` forward-refs allowed) and flow-store upsert keyed on
  metadata.id. Added 3 unit tests (registry folder map + flow handler round-trip + validate). **Tests: new
  portability 9/9; portab+git+flow suite 787 passed / 1 skip — no regressions; route imports clean.** No frontend.
  **Next up (iter 5, ~16:41):** fix-step 4 — flows-as-files per-cell layout (A3) on this registry foundation:
  serialize_flow_files(spec)->[{path,content}] writing flows/<slug>__<id8>/flow.toml + cells/NN_<key>.{sql,py,md},
  load_flow_files round-trip, lossless via TaskSpec. Single focused change; add round-trip test; run pytest.
- **16:42 SAST · iter 5** — Shipped A3 flows-as-files DIRECTLY: new backend/app/git/flow_files.py
  (serialize_flow_files/load_flow_files; flows/<slug>__<id8>/flow.toml + cells/NN_<key>.{sql,py,md}; full cell dict
  stored minus source [→sidecar] and ui [→[layout]] for lossless round-trip via TaskSpec) + test_flow_files.py
  (4 tests, all pass — round-trip asserts model_dump equality). Uses tomllib/toml (both available).
  **User messages handled:** (a) confirmed cron `36c3a15a` (every 15m) is live; (b) found+folded the two root design
  docs into NEW workstreams G (CLIENT_COMPUTE_PLAN.md — scan/slice params, useWidgetData, run-location badges) and
  H (DASHBOARD_TABS_AND_FILTERS — Track T tabs + Track F filter overhaul; = concrete impl of workstream F);
  (c) honoured "use worktrees + multi-agents": LAUNCHED worktree fix Workflow `w7va0dkt2` (2 isolated BACKEND agents,
  disjoint files, pytest-gated, each commits on its own worktree branch): [1] dashboards/spec.py — Track-T tabs/tab_id
  + client-compute param `mode` (additive, legacy-safe) + tests; [2] file-view endpoints A7 (environments.py GET
  /git/files + /content with path-traversal guards) + tests. Backend-only because frontend can't build in a worktree
  (node_modules not per-tree). **Tests this turn: flow_files 4/4 pass.**
  **Next up (iter 6):** review `w7va0dkt2` results → merge the two worktree branches into the working tree (disjoint
  from my uncommitted set, so clean) → run full backend pytest + npm run build → update Status. Then continue:
  wire flow_files into env_sync write/read (A3 remainder), or fix-step 5 (variables A5).
- **17:07 SAST · iter 5b (worktree merge)** — Workflow `w7va0dkt2` completed: both worktree agents committed clean.
  Reviewed diffs, brought files into the working tree via `git checkout <branch> -- <paths>` (no merge commit — keeps
  the leave-uncommitted pattern). [1] dashboards/spec.py: `Tab` model + `spec.tabs` + `widget.tab_id` (dup-id /
  undeclared-tab_id hard errors, drawers exempt, tab_id None→first tab) + `Variable.mode` scan/slice (None≡scan,
  legacy-safe) — Track-T T1 + G-P1 backend seed. [2] environments.py: GET /git/files + /git/files/content with
  path-traversal guards — A7 backend done. Removed both worktrees + deleted branches + pruned. **Tests: merged-in
  suites 138 pass; broad regression (dashboard/spec/environment/git/portab) 467 passed — no regressions.** The
  worktree multi-agent round-trip works end-to-end (launch → isolated commit → review → checkout-merge → cleanup).
  **Next up (iter 6, ~17:11):** continue — either wire flow_files.py into env_sync write/read (A3 remainder) directly,
  OR launch another worktree batch for the next disjoint slices (e.g. A10 run spec-snapshot+pagination backend, and
  G/H frontend pieces merged-then-built). Keep ≤6 agents, one workflow at a time, tests gating.
- **17:08 SAST · iter 6** — A-stream fixes are well advanced and the 4 not-started workstreams B/C/D/E had no audit
  yet, so per the loop rule ("else audit the next not-started areas") launched read-only audit Workflow `wf457gthc`
  (4 parallel agents — no worktrees, read-only): B security/pentest (IDOR/cross-org, RLS/predicate-injection,
  path-traversal, SSRF, secret-in-argv), C frontend (a11y, dead code, dark-contrast, bundle, git-UI placement),
  D responsive (headless-chrome screenshots @390/768/1440 on landing/pricing, view crops), E docs (drift vs code +
  coverage of new features + three-persona story). Dev server confirmed up on :5173. No files touched this fire.
  **Next up (iter 7):** fold `wf457gthc` findings into Status + per-area gap lists; dispatch worktree FIX agents for
  the highest-severity (likely B criticals first — any cross-org/IDOR or secret leak), backend in worktrees /
  frontend merged-then-built. One workflow at a time; tests gate.
- **17:12 SAST · iter 7** — Audit `wf457gthc` STILL IN FLIGHT (0-byte output, no completion notice; responsive
  screenshots + multi-route security read take time). Constraint = one workflow at a time, so did NOT launch
  another. Made a small safe disjoint increment instead: added `listGitFiles` + `getGitFileContent` helpers to
  src/lib/gitenv.js for the A7 file-view endpoints shipped iter-5b (read=graceful-null, content=throws to surface
  400/404, mirroring house pattern) — unblocks the future GitFilesPanel. **`npm run build` green.** No backend
  touched. **Next up (iter 8):** when `wf457gthc` completes, fold B/C/D/E findings → Status + gap lists → dispatch
  worktree fix agents (security criticals first). If still running, do another small disjoint increment.
- **17:26 SAST · iter 8** — Audit `wf457gthc` completed (4 agents, ~267k tok). Folded ALL B/C/D/E findings into the
  Status table + a highest-severity digest. Headline = 2 HIGH security findings forming a cross-tenant exfiltration
  path in the flow executor (B1 SQL-injection via {{params}} string-interp; B2 RLS policies={} never applied to flow
  query cells). Org-scoping/IDOR and the /query path audited CLEAN. Verified executor.py/planner.py/runtime.py are
  NOT in my uncommitted set (routes/flows.py IS), so LAUNCHED worktree fix Workflow `wmje0a0ep` (2 agents): [1] B1
  SQL-injection — bind flow query-cell {{params}} positionally via the /query param engine instead of interpolating
  into sql, + regression test proving UNION injection is neutralised (executor.py + new test; told NOT to touch
  flows.py); [2] docs E — document file-view endpoints + flows-as-files (reconcile 3 layouts) + estimate +
  tabs/scan-slice + three-persona story (docs/*.md). Both disjoint from my uncommitted set → clean checkout-merge.
  **B2 DEFERRED (deliberate):** interactive-run RLS = safe fix (thread caller identity.policies; touches my dirty
  flows.py → I do it directly next fire after B1's approach is visible); SCHEDULED-run RLS (flows_tick claims=None)
  = genuine security DESIGN DECISION (flow-owner stored policies vs service identity) → flag for user. No files
  touched by me this fire. **Next up (iter 9):** review+merge `wmje0a0ep` (scrutinise B1 diff before merge); do B2
  interactive-RLS in flows.py directly; surface the scheduled-RLS design question. Queue: C code-splitting (main
  tree, build-verifiable), D pricing grid, B3-B7, then A5 variables + G/H frontend.
- **17:27 SAST · iter 9** — (a) D pricing-grid fix DIRECTLY in main tree (build-verifiable): PricingPage.jsx now
  lg:grid-cols-3 + 2xl:grid-cols-5 with 2xl:max-w-[110rem] container so cards never shrink as the screen grows.
  `npm run build` green. (b) Fix batch `wmje0a0ep` completed mid-fire — REVIEWED both diffs carefully and MERGED:
  B1 SQL-injection fix (executor.py bind_sql_params + 5 regression tests incl. UNION-payload-binds-to-zero-rows;
  read the core binder + the durable-path reroute, which reuses registry._resolve_flow_connector and mirrors
  _handle_query 1:1) and docs E (5 files). Brought both in via checkout-merge (clean — disjoint from my set), ran
  the FULL backend suite = **3167 passed / 21 skip / 0 fail**, removed worktrees + branches. **The B1 reroute is the
  one behavioural change to smoke-test against a live BYO connector (no in-repo live-connector test exists).**
  **Next up (iter 10):** B2 interactive-RLS (thread caller identity.policies into preview/run-cell/run-flow claims
  in flows.py — I own that file) + surface the scheduled-RLS design question; then C code-splitting (main tree),
  B3-B7 hardening, A5 variables, G/H frontend.
- **17:42 SAST · iter 10** — B2-INTERACTIVE RLS fix DIRECTLY in flows.py (mine): added `verified_identity` dep to
  all 4 interactive routes (create_blend/preview_cell/run_cell/run_flow) and replaced `"policies": {}` →
  `dict(identity.policies)` so flow query cells are RLS-row-filtered by the caller's verified token, exactly like
  /query (empty policies for admin/unscoped caller → unchanged behaviour). current_user only carries {id,email,...}
  with no policies, so verified_identity (same dep /query uses; accepts first-party+embed) was the clean source.
  Initially saw 19 failures under a `-k` subset but proved them fixture-ordering artifacts (B2 flow tests 123 pass;
  query_stream 25 pass in isolation); **FULL backend suite 3167 passed / 21 skip / 0 fail.** B2-SCHEDULED left as an
  explicit OPEN DESIGN QUESTION for the user (what identity a scheduled flow runs under). No frontend touched.
  **Next up (iter 11):** C frontend code-splitting (HIGH — React.lazy in App.jsx + vite manualChunks for echarts/
  reactflow/arrow/duckdb-wasm/monaco; do in main tree, build-verifiable) OR launch a worktree batch for B3-B7
  backend hardening (secret-escape, SSRF allowlist, git-token-argv, /git/restore path-guard, hmac tick-secret) —
  these are disjoint backend files, good parallel worktree candidates. Then A5 variables + G/H frontend.
- **17:57 SAST · iter 11** — B-hardening. Did B7 DIRECTLY (flows.py is mine/dirty): tick-secret now uses
  hmac.compare_digest (was timing-leaky !=), + `import hmac`. Tick/flows_api tests 42 pass. LAUNCHED worktree batch
  `wdsxubszn` (2 agents, CLEAN files only, to avoid colliding with my dirty set): [B4] new app/connectors/ssrf.py
  guard (blocks loopback/link-local/RFC1918/metadata 169.254.169.254, resolves ALL addrs vs DNS-rebind,
  NUBI_SSRF_ALLOW_PRIVATE escape hatch) applied to http_json.py + tests; [B6] harden GitSync.read() path-traversal
  in git/sync.py (resolve+confirm within repo_dir) + tests. Files split because duckdb_conn.py (B3/B4-httpfs) and
  routes/git.py (B6-restore) are in my DIRTY set → I'll do those parts myself. B5 (git-token-in-argv) DEFERRED:
  delicate (changes git auth mechanism), tests stub git so an agent could pass-but-break real auth — needs
  real-remote verification; flag for careful handling. **Next up (iter 12):** review+merge `wdsxubszn`; then do the
  dirty-file hardening myself — B3 (duckdb CREATE SECRET quote-escaping) + B4-httpfs (reuse ssrf.py for the s3
  endpoint host) in duckdb_conn.py, B6-restore (_validate_git_file_path on /git/restore) in routes/git.py. Then
  C code-splitting, A5 variables, G/H frontend. B5 + rate-limiting to user/later.
- **18:0x SAST · iter 12** — Reviewed+merged `wdsxubszn` (both agents committed clean): B4 SSRF guard
  (app/connectors/ssrf.py + http_json.py; fail-open only on genuine DNS failure — acceptable, can't be an SSRF
  target) and B6 GitSync.read path-traversal guard (git/sync.py). Verified guard placements before merge; merged via
  checkout (clean — disjoint from my dirty set), removed worktrees+branches. Then did B3 DIRECTLY in my dirty
  duckdb_conn.py: added _sq() single-quote-doubling to every CREATE SECRET interpolated value (key_id/secret/region/
  url_style/endpoint/scope) + a breakout test (fixed my own flawed first assertion — doubled `''` still contains `'`
  as a substring, so assert the fully-doubled form instead). **FULL backend suite 3203 passed / 21 skip / 0 fail**
  (was 3167; +SSRF/gitsync/B3 tests). Security now: B1,B2-interactive (HIGH) + B3,B4-http,B6-read,B7 (MED/LOW) all
  done. **Next up (iter 13):** C frontend code-splitting (HIGH, remaining; React.lazy in App.jsx + vite manualChunks,
  main tree build-verifiable) — OR A5 variables (the foundation G/H/F build on). OPEN for user: B2-scheduled identity,
  B5 git-token-argv (real-remote test), rate-limiting. B4-httpfs + /git/restore-guard = low-pri dirty-file follow-ups.
- **18:12 SAST · iter 13** — C frontend code-splitting (the remaining HIGH) DONE in main tree: App.jsx now
  lazy-loads every authed/heavy page + public Dashboard/Editor (28 routes) behind ONE top-level <Suspense>
  (RouteFallback); marketing routes stay eager so landing is unaffected. vite.config.js manualChunks splits
  echarts/reactflow/apache-arrow/@duckdb/duckdb-wasm/@monaco-editor into named vendor chunks. **Entry chunk
  4382kB → 1893kB (~57% smaller); echarts(1MB)/reactflow/duckdb/arrow + route chunks now lazy (OFF the landing
  critical path).** Build green. SMOKE-VERIFIED via headless Chrome: landing renders, /login renders, and the now-lazy
  /dev/illustrations route resolves through Suspense and renders (proves lazy+Suspense works at runtime, not just
  builds). No backend touched. **Next up (iter 14): START A5 VARIABLES** — the keystone the F/G/H + templating work
  builds on. Plan: migration 0007_variables.sql (core, NOT ee) + app/vars/store.py (InMemory+Pg dual, mirror
  flows/store.py) + CRUD routes + reserve `vars` name + inject {{vars.*}} into the /query render context (bound,
  cache-key-safe). Big → slice across fires: store+migration+CRUD first (worktree agent, backend-disjoint), then
  template wiring, then Python set_var + run-overlay. Then G/H frontend. OPEN for user unchanged.
- **18:27 SAST · iter 14** — Started A5 (variables — keystone for templating/F/G/H). Gathered patterns (flows/store.py
  dual-store + get/set_flow_store singleton; resources.py self-registering router via app.routes.api_router + main.py
  import; migration naming 0007 next; query.py/template.py confirmed CLEAN for the later wiring slice). LAUNCHED
  worktree agent `w7mcvawgr` for the FOUNDATION: 0007_variables.sql (CORE table: org_id, project_id nullable, key,
  value jsonb, unique index over (org_id, COALESCE(project_id,zero-uuid), key)) + app/vars/store.py (InMemoryVarStore
  + PgVarStore + get/set_var_store) + app/routes/variables.py (CRUD, org+project scoped, reader GET / writer mutate,
  cross-org 404) + main.py self-register import + tests. Run-overlay + Python set_var SDK EXPLICITLY deferred to a
  later slice. All new files + 1-line main.py import → disjoint from my dirty set → clean checkout-merge.
  **Next up (iter 15):** review+merge `w7mcvawgr`; then A5 slice 2 — wire {{vars.*}} into the /query render context
  (query.py clean; inject a `vars` sub-dict into the resolved template context so Jinja binds them positionally,
  cache-key-safe; reserve `vars` in _TOKEN_CLAIM_RESERVED_NAMES). Then set_var/run-overlay, then G/H frontend.
- **18:40 SAST · iter 15** — Reviewed+MERGED A5 foundation `w7mcvawgr`. Scrutinised the two correctness-critical
  parts before merge: PgVarStore.set_var ON CONFLICT targets the exact COALESCE(project_id,zero-uuid) expression of
  the unique index (InMemory mirrors via a sentinel scope key), and the route uses resolve_org_id + reader/writer
  gates with cross-org→404. Migration FKs are sound (orgs/projects ON DELETE CASCADE, users SET NULL). Merged via
  checkout (all new files + 1-line main.py import, registered BEFORE resources.py catch-all), **full backend suite
  3213 passed / 21 skip / 0 fail** (+10 var tests, no regressions), removed worktree+branch. Reviewer flagged a
  minor follow-up: conftest._reset_state doesn't reset the var-store singleton (harmless now — every var test injects
  its own; worth a reset hook for symmetry later). A5 persistent store + CRUD is DONE.
  **Next up (iter 16): A5 slice 2** — wire {{vars.*}} into the /query render context. Do directly in query.py (clean):
  after building the `resolved` named-params dict, inject a `vars` sub-dict (org+project store lookup) into the
  render_sql_template context so {{ vars.key }} binds positionally (cache-key-safe, same as named params); reserve
  `vars` in _TOKEN_CLAIM_RESERVED_NAMES so a param can't shadow it; add a test. Then set_var/run-overlay, then G/H.
- **18:42 SAST · iter 16** — A5 slice 2 DONE directly in query.py (clean, hot path — careful work). Discovered
  resolve_named_params already DELEGATES to the Jinja sandbox (template.py), so {{ vars.key }} dotted access works
  once `vars` is in the context. Added: `_load_query_vars(org, project)` (org-global ⊕ project overlay, project
  wins; best-effort → {} on store error); inject `_template_vars` into the render context guarded by `"{{" in sql`
  AND identity.org (org from the verified identity — org_id is resolved later in the handler); `resolved["vars"]=…`
  in the registered-params branch + a NEW templated-no-declared-params branch (renders with {vars}, undefined var →
  clean 400 unknown_template_var); reserved `vars` in _TOKEN_CLAIM_RESERVED_NAMES. Values BIND positionally via the
  finalize hook → cache-key-safe + injection-safe (same guarantee as named params + the B1 fix). Hit one
  UnboundLocalError (used org_id before it's assigned at L767) → switched to identity.org. +4 tests
  (test_query_vars.py: positional binding, injection-payload-bound, reserved-name, project-overlay merge). **Full
  suite 3217 pass / 21 skip / 0 fail.** **Next up (iter 17):** A5 slice 3 — Python set_var + run-scoped overlay in
  flows (executor/runtime: inject `vars` read + a set_var(name,value,persist=False) into the python cell wrapper;
  run-overlay carried on TaskContext, wins over store; persist=True flushes to the store) + add `vars` to flow SQL
  cell context. Then sync vars→variables.json (git), then G/H frontend (scan/slice + tabs + reactive filters F).
- **18:57 SAST · iter 17** — Deferred A5 flow-vars (read path = 4 TaskContext build sites incl. the security-sensitive
  durable runtime + one site missing org_id; too intricate to land cleanly with the window closing → deserves a
  dedicated fire). Instead completed the self-contained security item **B4-httpfs**: added guard_s3_endpoint() to
  app/connectors/ssrf.py (blocks ONLY cloud-metadata 169.254.169.254/fd00:ec2::254 + link-local; private/loopback
  stay LEGAL since self-host MinIO lives there) and called it in duckdb_conn.py setup_s3_httpfs before registering
  the ENDPOINT — closes the httpfs IMDS-credential-theft SSRF vector. +5 tests (metadata blocked w/ port+scheme,
  private MinIO allowed, public allowed, ipv6 link-local blocked, empty no-op). **ssrf+duckdb_storage 71 pass;
  connector/s3/datasets regression 353 pass / 13 skip — no regressions.** SSRF now closed on BOTH vectors
  (http_json + httpfs). **Next up (iter 18): A5 slice 3 (flow vars) as a dedicated fire** — add `vars` field to
  TaskContext + "vars" to _BOUND_NAMESPACES/_resolve_native (bound in flow SQL cells like /query) + a shared
  _load_flow_vars helper wired at the 4 TaskContext sites (preview route needs org_id added) + Python cell
  set_var/run-overlay (persist flag). Then vars→variables.json, then G/H frontend.
- **19:12 SAST · iter 18** — A5 slice 3 READ path DONE (flow vars). Executor: added `vars` field to TaskContext,
  "vars" to _BOUND_NAMESPACES, and a `vars` branch to both _resolve_native (SQL bind path) and _resolve_value
  (string config). Added shared `load_vars_namespace(org_id, project_id)` to app/vars/store.py (org-global ⊕ project
  overlay, best-effort → {} so a vars-load failure NEVER breaks a run — important for the durable/scheduled path).
  Wired `vars=await load_vars_namespace(...)` at the 2 durable runtime TaskContext sites (project from flow_dict) +
  the async preview route in flows.py (also ADDED the missing org_id there). The 4th site — the SYNC runtime
  preview_cell helper — can't await, so left vars={} (the async route covers the interactive path; reverted that one
  line after hitting `SyntaxError: await outside async function`). Flow SQL cells now bind {{ vars.* }} positionally
  via the B1 bind_sql_params path → cache+injection-safe, same as /query. +2 tests (flow SQL binds vars; injection
  payload stays bound). **Full backend suite 3224 pass / 21 skip / 0 fail.** {{vars.*}} now works in BOTH /query and
  flow SQL cells. **Next up (iter 19): A5 slice 3-WRITE** — Python cell `set_var(name,value,persist=False)` + read
  `vars` dict in the python wrapper (registry.py) + run-scoped overlay carried on TaskContext (set during a run,
  visible to later cells; persist=True flushes to the store). Then vars→variables.json (git sync), then G/H frontend.
- **19:27 SAST · iter 19** — A5 slice 3 READ-python DONE (small completable slice; deferred the multi-part WRITE
  path with the window closing). Injected a read-only `vars` dict into the python cell wrapper (registry.py)
  alongside inputs/params/secrets — python cells can now read `vars["key"]`, mirroring the {{ vars.* }} SQL namespace.
  +1 sandbox test (python cell reads vars + computes). **Full backend suite 3225 pass / 21 skip / 0 fail.**
  {{vars.*}} read access now complete across /query SQL, flow SQL cells, AND python cells. **Next up (iter 20):
  slice 3-WRITE** (one coherent piece, needs a full fire): python `set_var(name,value,persist=False)` helper in the
  wrapper that accumulates onto __FLOW_RESULT__; runtime merges each cell's set vars into a run-scoped overlay on
  TaskContext (visible to later cells, overlay wins over store); persist=True flushes to the var store. Then
  vars→variables.json git sync, then G/H frontend. If a fire lands after 19:58 → STOP (CronDelete + final summary).
- **19:42 SAST · iter 20** — Window nearly closed (~16 min); did NOT start the multi-part slice 3-WRITE (would risk
  half-done work at shutdown). Did the tidy completable follow-up the A5 foundation agent flagged: added an
  InMemoryVarStore reset to conftest._reset_state (between-test isolation symmetry with the flow store). Conftest-only,
  low-risk. **Full backend suite 3225 pass / 21 skip / 0 fail** (verified the global reset broke no cross-test state).
  **Next fire (~19:57/20:05) will be past 19:58 → STOP: CronList + CronDelete 36c3a15a + final summary.** Remaining
  for whoever continues: A5 slice 3-WRITE (python set_var + run-overlay + persist), vars→variables.json git sync, then
  G/H frontend; plus the OPEN user decisions (B2-scheduled RLS identity, B5 git-token-argv real-remote test, rate-limit).

## ═══════════════════ FINAL SUMMARY (loop ended 19:57 SAST, 2026-06-10) ═══════════════════
Loop ran 15:56→19:57 (~4h, 20 iterations). Cron job `36c3a15a` CronDelete'd. Tests gated every
change; final state GREEN (backend pytest 3225 pass / 21 skip / 0 fail; npm run build green).
All work left UNCOMMITTED in the working tree (47 files) per house rule — user commits when ready.

### SHIPPED & VERIFIED
**Security (B) — all HIGH closed + MED/LOW hardening:**
- B1 [HIGH] Flow SQL-injection: {{params/inputs/item}} now bound positionally (executor.bind_sql_params), not
  string-interpolated. Durable query cells rerouted via _execute_query_with_bridge. ⚠️ SMOKE-TEST a live BYO
  connector durable run (no in-repo live-connector test).
- B2-interactive [HIGH] RLS: 4 interactive flow routes thread caller identity.policies (was {}).
- B3 DuckDB CREATE SECRET quote-escaping · B4 SSRF guard (app/connectors/ssrf.py) on http_json AND httpfs s3-endpoint
  (metadata/link-local blocked, private/MinIO allowed) · B6 GitSync.read path-traversal · B7 hmac.compare_digest tick.
**A-stream:** A2 flows now IMPORT on git pull (registry-driven) · A3 flow_files.py (flows/<slug>__<id8>/flow.toml +
  cells/NN.{sql,py,md}, lossless) · A5 variables: 0007 migration + vars/store.py (dual) + CRUD routes + {{vars.*}}
  READ across /query SQL, flow SQL cells, AND python cells (bound, cache+injection-safe, org⊕project) · A7 read-only
  git file-view endpoints · A8 Connector.estimate (BigQuery dry-run + DuckDB EXPLAIN).
**C/D:** code-splitting (React.lazy + vite manualChunks; entry 4382kB→1893kB, ~57%↓; smoke-verified) · pricing-grid
  responsive fix · dark-contrast confirmed AA (non-finding) · gitenv.js file-view helpers.
**E:** docs for file-view endpoints, flows-as-files (3-layout reconcile), estimate, tabs/scan-slice, three-persona.
**H/G backend seeds:** dashboard Tab model + widget.tab_id; Variable.mode scan/slice.
Tooling: master charter + 2 read-only audit workflows (10+4 agents) + 4 worktree fix workflows, all reviewed/merged.

### REMAINING (priority order for whoever continues)
1. A5 slice 3-WRITE: python set_var(name,value,persist=False) + run-scoped overlay on TaskContext (set mid-run →
   visible to later cells; persist=True flushes to store). READ path is done; this is the write half.
2. vars→variables.json git sync; nubi.toml (A6); A4 output_schema; A9 logging table; A10 run spec-snapshot/pagination/
   cancel; A2 serializer convergence + deletion propagation; A1 RHS git sidebar; A7 frontend GitFilesPanel.
3. G (client-compute scan/slice frontend) + H (tabs TabBar/SpecRenderer/editor; filter overhaul) + F (reactive
   filter dependency graph) — the macro three-persona/dashboard work; all build on the A5 variables substrate.
4. C follow-ups: modal focus-trap, dead-file deletion (Chart/Playground/Dashboard/QueryCell.jsx).

### OPEN DECISIONS FOR THE USER (deliberately NOT auto-resolved)
- **B2-scheduled RLS identity:** what policy context a scheduled flow (flows_tick, claims=None) runs under —
  flow-owner stored policies (my lean) vs explicit service identity.
- **B5 git-token-in-argv:** fix changes git auth mechanism (credential-helper/askpass); tests stub git, so needs
  REAL-REMOTE verification before merge.
- **Rate limiting:** none exists; infra/middleware decision.

## ═══════════════════ VERIFICATION PASS (post-loop, user-requested) ═══════════════════
Fan-out: 5 adversarial read-only agents (wf wvz97qp5y) + orchestrator ground-truth.
GROUND TRUTH: backend pytest 3225 pass / 21 skip / 0 fail; npm run build green.
RESULT: ~28/30 claims VERIFIED against actual source. No overstated/contradicted claims.
TWO REAL GAPS (both already disclosed in charter as "remaining" — scaffolding, not lies):
  • A3 flow_files.py is UNWIRED — implemented + 4 tests pass, but NO production sync path imports it
    (grep: only its own test references it). Per-cell file layout exists but reaches no push/pull path yet.
  • A8 Connector.estimate() has ZERO callers anywhere — bigquery dry_run + duckdb EXPLAIN implemented +
    unit-tested, but no /query/estimate route and nothing invokes it. Pure scaffolding.
VERIFIER ERROR CAUGHT: the frontend agent claimed dashboard tab/mode has NO tests — FALSE. test_dashboard_spec.py
  has 16 passing Track-T/mode tests (dup-tab-id, undeclared-tab_id, valid tab_id, drawer-exempt, mode slice/scan/absent).
NITS: base.py estimate() docstring says DuckDB "EC:<n>" but the impl matches "~<n> rows" (test passes → code right,
  docstring stale). Dup-tab-id/undeclared-tab_id are emitted at the same severity as undeclared var-refs but
  validate_spec doesn't set parsed=None — so "hard" is consumer-dependent (ai/tools + git-import treat as blocking);
  this matches the existing variable-ref pattern by design.
