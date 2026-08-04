# Contribution 2: Explicitly show ResultBuilder Node as part of execution in the UI

> Contribution 1 ([vllm-omni #195](contribution_1_vllm_omni_195.md)) was discontinued after the feature turned out to be already resolved upstream. Its writeup lives in [contribution_1_vllm_omni_195.md](contribution_1_vllm_omni_195.md).

**Contribution Number:** 2  
**Student:** Charitarth  
**Issue:** [apache/hamilton#1150 — Explicitly show ResultBuilder Node as part of execution in the UI](https://github.com/apache/hamilton/issues/1150)  
**Status:** Phase IV — Complete. PR submitted upstream, awaiting maintainer review.  
**Branch:** [`feature/ResultBuilder-node-in-ui`](https://github.com/charitarthchugh/hamilton/tree/feature/ResultBuilder-node-in-ui)  
**Pull Request:** [apache/hamilton#1678 — Show the result builder as a node in the Hamilton UI](https://github.com/apache/hamilton/pull/1678) 

| | |
|---|---|
| Commits on the PR branch | 5 |
| Diff | 10 files, +670 / −16 |
| New tests | 15 |
| SDK suite | 148 passed, 7 skipped (baseline on `main`: 133 / 7) |
| Upstream repo | [apache/hamilton](https://github.com/apache/hamilton) |

---

## Why I Chose This Issue

[Apache Hamilton](https://github.com/apache/hamilton) is a mature ML/data dataflow framework for expressing data and feature pipelines as a DAG of Python functions. After my first pick ([vllm-omni #195](contribution_1_vllm_omni_195.md)) turned out to be already resolved upstream, I wanted a genuinely available, code-substantive issue, and this one checks every box:

- **Skill match:** I'm comfortable in Python and experienced at wrangling dev environments and deployment setups, which is what standing up Hamilton's docker-compose UI stack and working in the tracking SDK demands — the bulk of the change lives in Python SDK code.
- **Learning goal:** it touches Hamilton's lifecycle-hook / tracking-adapter architecture and spans the full stack (Python capture → Django model → React render), which is exactly the kind of end-to-end feature I want experience with.
- **Understanding:** I traced why the node is missing — the `ResultBuilder` is a lifecycle adapter, not a graph node, so the tracking hooks never see it — and mapped the fix across the SDK, backend, and frontend (see [Solution Approach](#solution-approach)). It also has real project impact: users would finally see the final artifact a run produced and compare outputs across runs.

---

## Understanding the Issue

### Problem Description

In Apache Hamilton, a `ResultBuilder` assembles the final result of a DAG run, e.g. compiling node outputs into a Pandas DataFrame, a dictionary, or a custom object. It runs as part of the execution lifecycle, but the UI never shows it — the execution graph just ends at the terminal nodes, so the actual artifact a run produced is invisible. This matters because the whole point of the Hamilton UI is run observability, and the final output is the one thing every user cares about.

This omission makes it difficult for users to:

1. Explicitly see what final artifact a run produced.
2. Understand how the final output was constructed from the terminal nodes of the DAG.
3. Compare the final outputs of different execution runs side-by-side.

### Expected Behavior

The issue asks that the UI explicitly show the `ResultBuilder` node as part of the execution, with:

- Incoming edges from all the terminal nodes (final variables) that fed into it.
- Its output type, schema, and summary statistics (data observability) displayed when selected.
- The ability to compare the final built result across different executions.

The shipped change delivers (2) and (3) in full, and (1) partially: the node records the outputs it combined on every run, but renders unconnected in the DAG view rather than with drawn edges. That divergence is deliberate, explained under [Approach Decisions](#approach-decisions), and disclosed in both the PR and the user-facing docs.

### Current Behavior

The `ResultBuilder` node is omitted from the execution graph visualization in the UI. The graph simply ends at the terminal nodes, and the final built result itself is not shown as a node with its own data observability metadata.

### Affected Components

- **SDK (Python):** `ui/sdk/src/hamilton_sdk/driver.py` (node-template extraction and `hash_dag`) and `ui/sdk/src/hamilton_sdk/adapters.py` (`HamiltonTracker` and `AsyncHamiltonTracker`).
- **Backend (Python/Django):** `ui/backend/server/trackingserver_template/models.py`, where the `NodeTemplate.NodeType` enum backs the `NodeTemplate.classifications` array field, plus its migrations. No table changes, but a new classification value has to be a listed choice.
- **Frontend (React/TypeScript):** `ui/frontend/src/state/api/friendlyApi.ts`, which holds the `Classification` union the frontend mirrors that server field with.

> Earlier phases of this writeup named `DAGViz.tsx` as the frontend touchpoint. That turned out to be wrong once I read how the UI actually styles nodes: nothing there hard-codes a classification, so the node renders through the existing paths and the union member is the entire frontend change. See [Challenges Faced](#challenges-faced).

---

## Reproduction Process

### Environment Setup

I got the local dev environment running with docker compose and checked each container manually to confirm it was working, applying patches as needed.

```bash
# 1. Frontend deps
npm install --prefix ui/frontend

# 2. Build + start backend stack
cd ui
HAMILTON_TELEMETRY_ENABLED=false docker compose -f docker-compose.yml up -d --build db backend

# 3. Frontend dev server (proxies /api -> :8241)
PORT=3000 npm run dev --prefix ui/frontend

# 4. Commit gate (husky is broken on v9 → use Python pre-commit)
pre-commit install --install-hooks
```

### Steps to Reproduce

1. Run a Hamilton DAG that uses a `ResultBuilder` (e.g. `PandasDataFrameResult` or `DictResult`) with the `HamiltonTracker` adapter enabled.
2. Open the Hamilton UI at `http://localhost:3000`.
3. Navigate to the project dashboard and select the run.
4. Observe the execution graph: the graph ends at the terminal nodes, and there is no node representing the `ResultBuilder` or the final compiled output.

### Reproduction Evidence

Containers up:

```
ui_db_1     Up   5432/tcp
ui-backend  Up   0.0.0.0:8241->8241/tcp
```

Endpoints responding:

```
backend direct :8241   GET /api/openapi.json -> 200 application/json
                       GET /api/docs         -> 200
vite dev    :3000      GET /                 -> 200 (React SPA)
                       GET /api/openapi.json -> 200 application/json (proxied to :8241)
```

- **My findings:** the graph in the UI ends at the terminal nodes; the combined object `execute()` returns appears nowhere. Materializers *do* show up, which became the key comparison for the analysis below.

---

## Solution Approach

### Analysis

The UI shows every transform node and its output, but the combined object `dr.execute([...])` actually returns never appears as a node. Users can't see what a whole execution produced, or compare it across runs. The reason is that the `ResultBuilder` isn't a graph node: the sync driver runs it as a post-processing step after `__raw_execute()` (`hamilton/driver.py:570-577`). Materializers do show up in the UI, because `materialize()` injects a real `_build_result` node into the graph before running (`hamilton/io/materialization.py`).

Nothing in the UI hard-codes what a "materializer" is, though. Nodes are special purely through their `classifications` string array plus tags, and the frontend branches on classification the same way for `isInput` and `hasArtifact`. Also, `post_graph_execute` already receives the final combined result; it just ignores it today. So the problem reduces to something Hamilton's tracking already knows how to do: register one extra node template and emit one extra task update, reusing `_extract_node_templates_from_function_graph`, `runs.process_result`, and the existing `update_tasks` client call.

Three facts I verified in the source turned out to constrain the whole design, so I recorded them before writing any code:

- `execute()` (`hamilton/driver.py:585`), `raw_execute()` (680) and `materialize()` (1617) **all** fire `post_graph_execute`, but only `execute()` has run `do_build_result` first. The other two pass a raw output dict.
- No lifecycle hook receives the adapter set, and `call_lifecycle_method_sync` does `(adapter,) = self.sync_methods[method_name]`. **The tracker cannot learn which result builder ran, or whether one ran at all.** So the node cannot be made conditional on a builder existing — that option simply isn't reachable from the SDK.
- `graph_utils.py:39` excludes `_`-prefixed functions from becoming nodes, so the name `_result_builder` is collision-proof against user *transforms* — though not against names that don't come from a function, which is why the shipped code detects collisions anyway.

### Proposed Solution

Synthesize a `_result_builder` node in the tracking SDK: register it as a node template when the tracker sees the graph, and report the combined result as that node's output when the run finishes, with the outputs that actually reached the result as its realized dependencies. The backend gains one `NodeType` choice so the value validates; the frontend gains one member in its `Classification` union and nothing else. Core Hamilton stays untouched.

There were two ways to do this: a synthetic node added by the SDK, or a real graph node injected in core the way `materialize()` does it. Core would change `execute()` semantics for every user, so I asked the maintainer on the issue before proceeding. He approved the SDK-side approach ("option A") on 2026-07-15.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** the ResultBuilder combines `execute()` outputs into the single object the caller gets back. It runs as a lifecycle adapter rather than a graph node, so the tracker never registers it and the UI has nothing to draw.

**Match:** materializers are the closest thing in the codebase to "show a build step as a node", since `materialize()` injects a real `_build_result` node into the graph. On the tracking side, `_extract_node_templates_from_function_graph` already registers nodes and `runs.process_result` + `update_tasks` already capture per-run output, which is everything this node needs. For tests, `ui/sdk/tests/test_adapters.py` has an established mock-client pattern and `ui/sdk/tests/resources/` an established fixture-module pattern.

**Plan.** I settled the design questions and wrote the answers down before writing any code, so each one got decided once rather than re-argued halfway through an implementation. The choices the code depends on:

- The node is emitted on every tracked run, on all `post_graph_execute` paths. The second analysis fact above forces this. It makes `raw_execute()` and `materialize()` runs summarize a raw output dict rather than a built result, which the PR and docs state rather than guard against.
- The node template carries no dependencies. They're recorded per run, because the requested outputs change from run to run.
- Reuse `runs.process_result` for the result summary instead of writing a second profiler.
- Add a `result_builder` choice to the backend `NodeTemplate.NodeType`, with one no-op postgres migration and an in-place edit to the squashed sqlite `0001_initial`.
- Failed runs emit no task run at all, so the node falls back to the UI's existing "not executed" rendering.
- One PR, SDK and UI only. `hamilton/` core is off-limits.
- Name the node `_result_builder`. Template output type is `typing.Any`; the real per-run type goes in the result summary.
- The frontend gets the `Classification` union member and nothing else: no styling, no icons, no filtering.
- Add a section to `docs/hamilton-ui/ui.rst` documenting the behaviour and its four caveats.

That gives the build order:

1. Kill the risky assumptions first. Is the backend migration really zero-SQL? Does reusing `process_result` on the combined result cost anything worth caring about?
2. Backend: the `NodeType` choice and both migration paths.
3. Frontend: one union member.
4. SDK template: append the synthetic node in `_extract_node_templates_from_function_graph`.
5. SDK task run: stash `final_vars` in `pre_graph_execute`, then emit one `update_tasks` call from `post_graph_execute`. Sync tracker, then async.
6. Docs, end-to-end verification in the UI, then the PR.

**Implement:** [`feature/ResultBuilder-node-in-ui`](https://github.com/charitarthchugh/hamilton/tree/feature/ResultBuilder-node-in-ui), five commits, one per layer. See [Code Changes](#code-changes).

**Review:** pre-commit (ruff check, ruff format, python-ast, ASF license headers) is green on every changed file — re-verified 2026-08-04 against the final branch. I ran the SDK suite at each of the five commits, so the branch bisects. I check `git status` against the intended file list every time, because `uv run` inside `ui/backend` quietly rewrites `uv.lock` and that must not ship.

**Evaluate:** red/green unit tests asserting the exact `update_tasks` payloads, `sqlmigrate` output for the migration, `makemigrations --check` for sqlite consistency, the full SDK suite against baseline, and real tracked runs rendering in the local UI with each result confirmed in postgres. All of it is recorded under [Testing Strategy](#testing-strategy).

---

## Testing Strategy

All automated tests live SDK-side, added to the existing suites rather than in a new parallel file, following the mock-client pattern already in `test_adapters.py` and the fixture-graph pattern already in `test_driver.py`. One new fixture module, `ui/sdk/tests/resources/dag_with_reserved_node_name.py`, follows the convention of the other modules in that directory.

### Unit Tests

**`ui/sdk/tests/test_driver.py`** — node-template synthesis (5 tests):

- [x] `test_extract_node_templates_appends_result_builder` — runs against a real `FunctionGraph`, asserts the template list grows by exactly one, checks the appended template's fields, and confirms `_result_builder` never enters `fg.nodes`. Red-green checked: comment out the `append` and it fails.
- [x] `test_extract_node_templates_omits_result_builder_by_default` — pins that the node is opt-in, so the legacy `Driver` in that module keeps behaving as it always has.
- [x] `test_result_builder_node_is_skipped_when_the_name_is_taken` — a dataflow that already defines `_result_builder` gets a logged warning and no synthetic node, instead of a failed registration.
- [x] `test_result_builder_node_changes_the_dag_hash` — the node is folded into `hash_dag` when registered, so a template registered before the node existed isn't silently reused.
- [x] `test_dag_hash_is_unchanged_when_the_name_is_taken` — the collision path leaves the hash alone, so the two decisions can't disagree.

**`ui/sdk/tests/test_adapters.py`** — tracker emission (10 tests):

- [x] `test_result_builder_task_run_is_emitted` — exactly one `_result_builder` task update, status SUCCESS, correct realized dependencies. Needed a `RecordingClient` that keeps every task update, since the existing `MockHamiltonClient` only keeps the latest. Red-green checked.
- [x] `test_result_builder_task_run_carries_a_result_summary` — the combined object is profiled and attached, which is the whole point of the issue.
- [x] `test_no_result_builder_task_run_when_the_name_is_taken` — the collision path emits nothing, so nothing is recorded against the user's own node.
- [x] `test_result_builder_task_run_not_emitted_on_failure` — a failed run emits nothing and the node renders not-executed. The guard most likely to break quietly later.
- [x] `test_result_builder_failure_does_not_break_the_run` — emission errors are logged and swallowed. Without this, an exception here would leave a successful run rendering as still-running forever.
- [x] `test_result_builder_task_run_is_emitted_when_the_builder_returns_none` — a side-effect-only builder still counts as having run.
- [x] `test_result_builder_dependencies_under_materialize` — `materialize()` asks `pre_graph_execute` for `final_vars + materializer_vars` but hands `post_graph_execute` only the `final_vars` slice, so the unfiltered list would credit the node with materializers it never saw. Runs a real `materialize()` against `tmp_path`.
- [x] `test_result_builder_dependencies_keeps_the_requested_list_for_a_builder_of_its_own_keys` — narrowing is only sound when the result is keyed by node name; a custom builder's own dict keeps the full list rather than be credited with nothing.
- [x] `test_async_result_builder_task_run_is_emitted` — async tracker parity.
- [x] `test_async_result_builder_task_run_not_emitted_on_failure` — async failure parity.

### Integration Tests

- [x] **Full SDK suite, branch:** `pytest ui/sdk/tests/ -q` → **148 passed, 7 skipped**.
- [x] **Full SDK suite, baseline on `apache/hamilton:main`:** same command in a clean worktree → **133 passed, 7 skipped**. The 15-test delta is exactly the 15 tests listed above; no pre-existing test changed state.
- [x] **The two touched files:** `pytest ui/sdk/tests/test_driver.py ui/sdk/tests/test_adapters.py -q` → **22 passed, 3 skipped**. The 3 skips are pre-existing Ray adapter tests that skip on `No module named 'ray'`, an optional dep absent from my venv and untouched by this diff.
- [x] **Per-commit:** the suite was run at each of the five commits, so the branch bisects cleanly.
- [x] **The migration really does emit no SQL.** `sqlmigrate trackingserver_template 0003` gives `BEGIN; -- Alter field classifications on nodetemplate -- (no-op) COMMIT;`, run against the generated migration rather than the throwaway I used to check the assumption up front.
- [x] **Sqlite consistency:** `makemigrations --check --dry-run` under `server.settings_mini` reports `No changes detected`. So the in-place edit to the squashed `0001_initial` genuinely resynchronises Django's model state, and Django does not want a `0002`.
- [x] **Style gate:** `pre-commit run --files <10 changed files>` → all hooks Passed or Skipped, nothing Failed.

### Manual Testing

End-to-end against a local UI backend with the migration applied, tracking real runs through the Django API and confirming each result in postgres — not just in the rendered page.

| Case | What I checked | Result |
|---|---|---|
| `DictResult` | Node renders; summary is the combined dict; deps narrowed to the requested outputs | Pass |
| `PandasDataFrameResult` | Full DataFrame profile in the node summary, via the existing `process_result` path | Pass |
| `materialize()` | Deps recorded correctly, materializer id excluded | Pass |
| Failed run | Nothing emitted; node renders not-executed rather than broken | Pass |
| Name collision | Warning logged, run succeeded, nothing recorded against the user's node | Pass |
| Baseline before the change | A tracked DAG run renders fully in the local UI, with no result-builder node | Pass |
| Comparing two runs | The node's summary is visible per run, which is the comparison the issue asks for | Pass |

**Before/after evidence** (full-size images are embedded in [PR #1678](https://github.com/apache/hamilton/pull/1678)): same dataflow, same requested outputs, same result builder, run twice against the same local UI, with only the SDK on the path differing.

| | nodes with recorded runs |
|---|---|
| [before](https://github.com/user-attachments/assets/7015e96a-1667-4c77-94ec-151c531e3528) | `average_squared, input_numbers, squared, sum_squared` |
| [after](https://github.com/user-attachments/assets/2102cf9c-8fb8-4ad8-b82e-ca2248b93e96) | `average_squared, input_numbers, _result_builder, squared, sum_squared` |

Opening the node shows [the object the caller received](https://github.com/user-attachments/assets/5d0b7e35-1063-4bb7-b31c-2a9442b9befc):

```
_result_builder — Result summary
{ 2 items
  "sum_squared":     int 55
  "average_squared": int 11
}
```

---

## Implementation Notes

### Development Timeline

| Date | What happened |
|---|---|
| 2026-06-10 | Forked and cloned; local UI stack stood up with docker compose |
| 2026-06-15 | Commented on [#1150](https://github.com/apache/hamilton/issues/1150) claiming the issue and asking for pointers |
| 2026-06-16 | Environment work; reproduction confirmed |
| 2026-07-01 | Posted my source-level analysis on the issue with two candidate approaches (A: SDK-side, B: core-side) and asked which the maintainer preferred |
| 2026-07-15 | Maintainer (`skrawcz`) replied: "A sounds reasonable." Implementation unblocked |
| 2026-07-29 | First implementation complete on `feature/ResultBuilder-node-in-ui-#1150`; self-reviewed it the same session and judged it not reviewable enough to submit — see below |
| 2026-08-01 → 08-04 | Rebuilt on `feature/ResultBuilder-node-in-ui`: ~14 working commits across four days, rebased into 5 reviewable per-layer commits |
| 2026-08-04 | Opened PR [#1678](https://github.com/apache/hamilton/pull/1678) against `apache/hamilton:main` |

Two honest notes on cadence, since both are visible to anyone reading the git history:

1. **The 06-16 → 07-29 gap is real, and it is not idle time.** That window is the issue-analysis phase: reading `driver.py`, `materialization.py` and the lifecycle hooks to work out *why* the node was missing, writing that up on the issue on 07-01, and then waiting on the maintainer's answer, which arrived on 07-15. Approach B would have thrown the implementation away, so starting to code before that reply would have been the wrong call. The analysis output is the 07-01 issue comment rather than a commit.
2. **All five PR commits carry a 2026-08-04 date** because I rebased the branch into one commit per layer immediately before opening the PR. The working history behind them runs 08-01 → 08-04, which the reflog shows. I did that because the honest working history was full of `Only narrow the result builder's dependencies by node name`-style corrections that make a five-file change hard to review; splitting by layer means each commit is independently checkable and the suite passes at every one. The earlier attempt's commits are still on the fork under [`feature/ResultBuilder-node-in-ui-#1150`](https://github.com/charitarthchugh/hamilton/tree/feature/ResultBuilder-node-in-ui-%231150).

### Phase III → Phase IV Progress

**Phase III (implementation)** landed everything the plan called for: the backend `NodeType` choice with both migration paths, the frontend union member, the SDK node template, the sync and async trackers' task-run emission, the docs section, and 15 tests. The three items still listed as open at the end of Phase III — async parity, the frontend change, and the docs section — are all done and on the branch.

**Phase IV (submission)** opened [PR #1678](https://github.com/apache/hamilton/pull/1678) after the end-to-end verification the PR was gated on. The PR follows the project's template (Changes / How I tested this / Notes / Checklist), leads with why rather than what, embeds before/after screenshots, and states all four known limitations rather than leaving a reviewer to find them.

### Challenges Faced

1. **Rejecting my own first implementation.** The branch worked on 2026-07-29, as two large `feat(sdk)` / `feat(ui)` commits, and I was ready to submit it that session. Reading it back as a reviewer would, I couldn't answer "why is this line here?" for several of them, and it also touched four frontend files that I had not verified were necessary. Rebuilding instead of submitting cost four days and was the right call: what shipped is 10 files instead of 13, one commit per layer, and every non-obvious decision has a reason written into the commit body. The cheap lesson is that "it works" and "it can be reviewed" are different finish lines.

2. **The DAG-hash bug I nearly shipped.** `register_dag_template_if_not_exists` matches on `(project_id, dag_hash, code_hash, name)` and returns the existing template *without inspecting the nodes posted with it*. My first version left `hash_dag` alone, reasoning that the synthetic node isn't part of the real graph so it shouldn't affect the graph's identity. That is wrong in the way that matters: any dataflow already registered before the upgrade keeps its old template, the tracker then logs task runs against a node that template doesn't contain, and the node silently never appears. Folding the name into the hash when the node is registered fixes it, at the cost of one new DAG version per dataflow on first run after upgrading — which the docs now warn about.

3. **A name collision that the docs said couldn't happen.** `graph_utils.py:39` excludes `_`-prefixed *functions* from becoming nodes, which is what made `_result_builder` look collision-proof. But node names that don't come from a function — an external input, a decorator-generated name — bypass that filter entirely, and `NodeTemplate` is unique on `(name, dag_template)`, so registering both would fail the user's run outright. The fix is to detect the collision and step aside with a warning: skipping costs that run its node, failing would cost the user their run. It has its own fixture module and two tests.

4. **Dependency narrowing under `materialize()`.** `Driver.materialize` asks `pre_graph_execute` for `final_vars + materializer_vars` but hands `post_graph_execute` only the `final_vars` slice. Recording the requested list unfiltered would credit the node with materializers it never saw. Narrowing against the result's own keys fixes that — but only when the result is keyed by node name, which `DictResult` and the raw dict are and a DataFrame or a custom builder's dict aren't. Getting this wrong in the other direction (narrowing always) credits a DataFrame-producing run with zero dependencies. Two tests pin both halves.

5. **Checking a documented behaviour instead of trusting it.** Hand-editing an already-applied squashed sqlite migration is only safe if adding a `choices` value emits no SQL. Django has treated `choices` as a non-DB attribute since 4.1, but I had reasoned that from the docs and never run it. Running it confirmed the no-op on Django 5.2.15, and corrected my own mental model on the way: there is no scalar `node_type` column at all, so the generated migration is `0003_alter_nodetemplate_classifications`. Skip this step and I would have hand-edited an applied migration on faith.

6. **A measurement that changed nothing, and one that nearly changed the scope.** Reusing `process_result` on a `DictResult` re-profiles outputs that were already profiled individually. Measured, it costs exactly twice the time and twice the payload, flat across three orders of magnitude, and it is bounded: payload tracks column count rather than rows (1k rows gives 6,517 B, 1M rows gives 6,798 B), and `MAX_DICT_LENGTH_CAPTURE = 100` really does cap it (100 DataFrames at 189,846 B, 300 at 189,735 B). So no size guard, and the reuse survives. What the benchmark did turn up was a pre-existing bug. `compute_stats_dict:106-109` tries `json.dumps` first and, if that works, stores the value verbatim, skipping every truncation guard. A 1M-element list costs 1,182 B as a bare node output and 20,630,689 B inside a dict. That is 17,454× amplification, measured rather than extrapolated.

7. **Deciding not to fix that bug.** Harder call than it sounds. `compute_stats_list:207-210` has the identical two lines with the identical comment, so it is a consistent design choice by the maintainers rather than an oversight. And the worst case you can already reach today, a list of 100 × 100k lists at 103 MB, is worse than anything my change causes. Fixing it would mean reverting a maintainer decision in a file #1150 never mentions, inside a PR about something else. It goes upstream as its own issue instead, and the docs section points at the knobs (`HAMILTON_MAX_LIST_LENGTH_CAPTURE`, `HAMILTON_CAPTURE_DATA_STATISTICS`) so anyone with large list outputs can find them.

8. **The sync/async hook-ordering asymmetry.** The sync driver fires `post_graph_execute` after `do_build_result`, so `results` is the combined object. The async driver fires it inside `raw_execute`'s `finally`, before the build. The async tracker therefore cannot see the built output without core changes, and core was off the table. Async ships with the node emitted and summarizing the raw output dict, with the limitation stated in the PR and the docs rather than papered over.

9. **Toolchain traps worth writing down.** Generated Django migrations have no ASF license header, so `pre-commit` fails once and the `insert-license` hook writes it for you; re-run to go green. `uv run` inside `ui/backend` quietly rewrites `uv.lock` with re-resolution churn and a package version bump, which the repo reserves for release commits, so it has to be reverted every time. And `FunctionGraph.from_modules(mod, config=...)` must be called with no adapter at all: passing `base.DefaultAdapter()` dies at `graph.py:78` with `AttributeError: 'DefaultAdapter' object has no attribute 'does_method'`, because it wants a `LifecycleAdapterSet`.

### Code Changes

Five commits, one per layer, each with the suite passing.

| Commit | Title | Files |
|---|---|---|
| [`c4a2755d`](https://github.com/apache/hamilton/pull/1678/commits/c4a2755d9432a47951ea566d240f1da6c75017e1) | Add a result_builder node classification to the tracking server | 3 files, +50 |
| [`e7e9d9d5`](https://github.com/apache/hamilton/pull/1678/commits/e7e9d9d51e7703ad5ea7ea6ff7fe24f01f9c91f8) | Add result_builder to the frontend Classification union | 1 file, +2 / −1 |
| [`059e09ba`](https://github.com/apache/hamilton/pull/1678/commits/059e09ba86ac85f016bf5457ffc98cc31d42add5) | Synthesize a _result_builder node template in the SDK | 3 files, +180 / −3 |
| [`0e166995`](https://github.com/apache/hamilton/pull/1678/commits/0e166995079b2807a6c85d4162a8f2bb0a450d9b) | Emit the result-builder task run from the trackers | 2 files, +401 / −12 |
| [`12799ae8`](https://github.com/apache/hamilton/pull/1678/commits/12799ae8fc1f93a64f2c26b7f5bb52e81dff712c) | Document the result builder node in the UI docs | 1 file, +37 |

Files modified, by layer:

**Backend** — [`c4a2755d`](https://github.com/apache/hamilton/pull/1678/commits/c4a2755d9432a47951ea566d240f1da6c75017e1)

- `ui/backend/server/trackingserver_template/models.py` — adds `result_builder = "result_builder", _("ResultBuilder")` to `NodeTemplate.NodeType`.
- `.../migrations/0003_alter_nodetemplate_classifications.py` — generated by Django rather than hand-written. One `AlterField`, zero SQL.
- `.../migrations_sqlite/0001_initial.py` — one line added to the inline `choices` list, matching how `0002`'s `unique_together` is already carried there.

**Frontend** — [`e7e9d9d5`](https://github.com/apache/hamilton/pull/1678/commits/e7e9d9d51e7703ad5ea7ea6ff7fe24f01f9c91f8)

- `ui/frontend/src/state/api/friendlyApi.ts` — one member added to the `Classification` union. No styling, icon or filter: the node renders through the existing paths.

**SDK, node template** — [`059e09ba`](https://github.com/apache/hamilton/pull/1678/commits/059e09ba86ac85f016bf5457ffc98cc31d42add5)

- `ui/sdk/src/hamilton_sdk/driver.py` — adds `RESULT_BUILDER_NODE_NAME`, `_should_register_result_builder()` and `_result_builder_node_template()`, an `include_result_builder` flag on both `hash_dag()` and `_extract_node_templates_from_function_graph()`. The template dict is built directly rather than by extending `_convert_classifications`, because there is no `Node` object to hand that function; bypassing it was the smaller diff.
- `ui/sdk/tests/resources/dag_with_reserved_node_name.py` — new fixture module for the collision case.
- `ui/sdk/tests/test_driver.py` — five tests.

**SDK, task run** — [`0e166995`](https://github.com/apache/hamilton/pull/1678/commits/0e166995079b2807a6c85d4162a8f2bb0a450d9b)

- `ui/sdk/src/hamilton_sdk/adapters.py` — stashes `final_vars` per `run_id` in `pre_graph_execute`; adds `_emit_result_builder_task_run()` to both trackers, called from the success arm of `post_graph_execute`. The payload is built once in a shared, I/O-free `_result_builder_payload()` so the two trackers differ only in how they reach their client. Timestamps are a zero-width span at the real `finally`-block instant: no hook brackets the builder, so its duration is not observable, and a real instant beats a made-up duration. `process_result` wants a `Node`, so it gets a minimal stand-in, since it only reads `.name` and `.tags`.
- The same file's `post_node_execute` had a 25-line attribute-shaping block already duplicated in the async twin. Emitting the same shape a third time would have made three copies, so it now lives in `_result_attribute` / `_result_attributes` shared by all three call sites.
- `ui/sdk/tests/test_adapters.py` — ten tests plus a `RecordingClient`.

**Docs** — [`12799ae8`](https://github.com/apache/hamilton/pull/1678/commits/12799ae8fc1f93a64f2c26b7f5bb52e81dff712c)

- `docs/hamilton-ui/ui.rst` — a "The result builder node" section covering what the node is, that nothing is needed to enable it, that it renders unconnected, and the four caveats (double profiling, which driver paths see a built result, failed runs, name collisions).

### Approach Decisions

- **SDK-side synthesis over core node injection.** The maintainer approved it, and injecting in core would change `execute()` semantics for everyone.
- **The node is unconditional.** Not a shortcut: `call_lifecycle_method_sync` unpacks exactly one `do_build_result` adapter and no hook receives the adapter set, so the tracker cannot detect whether a builder ran. The resulting `raw_execute()` / `materialize()` imprecision is documented, because the check I would want to write cannot be written correctly.
- **No drawn edges, against the issue's literal wording.** The issue asks for incoming edges from the terminal nodes. The DAG view builds edges from *template* dependencies, and the template can't have them — which nodes feed the result varies per run, so a fixed list would be wrong for any subset run. Per-run dependencies are recorded and shown, so the information the issue wants is there; the edges are not. Stated plainly in the PR and the docs rather than quietly omitted.
- **Registration is opt-in.** A caller that registers the node without also emitting a task run would render it never-executed on every run, which is what the legacy `Driver` in that module would do. The flag defaults off.
- **Emission failures are logged and swallowed.** This runs immediately before `log_dag_run_end`, so an exception escaping here would leave a successful run rendering as still-running forever.
- **The two measurements were scheduled ahead of the code they de-risk**, so an unsafe migration edit or a bad profiling reuse would surface before implementation instead of after.

---

## Pull Request

**PR Link:** [apache/hamilton#1678 — Show the result builder as a node in the Hamilton UI](https://github.com/apache/hamilton/pull/1678)

**Target:** `apache/hamilton:main` ← `charitarthchugh:feature/ResultBuilder-node-in-ui`. Open, not a draft. Opened 2026-08-04.

**Status:** Awaiting maintainer review. No review, comment or CI verdict has come back yet as of 2026-08-04.

### Summary

The PR opens with `Closes #1150` and then explains the *why* before the *what*: the combined result `execute()` returns is assembled by a result builder after the last node finishes, outside the dataflow, so there is no node to attach a data summary to — every individual output is profiled in the UI, but the object the caller actually receives is not. It then covers:

- **Before / after evidence** — two screenshots of the same dataflow run against the same local UI with only the SDK differing (4 nodes → 5), plus a screenshot of the node's result summary.
- **Changes** — one subsection per commit, each explaining why that layer had to change: `NodeTemplate.classifications` is a constrained choice field so the SDK can't send an unknown value; the frontend union mirrors that field; the template can't carry fixed dependencies; the combined result only exists at `post_graph_execute` time.
- **How I tested this** — the test commands and counts, that the suite passes at each of the five commits, and a five-row table of end-to-end cases verified against a local backend with results confirmed in postgres.
- **Notes** — the four limitations, stated up front: dependency narrowing under `materialize()`, that not every driver path sees a built result, that the node is part of the DAG hash so the first run after upgrading registers a new dataflow version, that it renders unconnected, and that the result is profiled twice. Plus one flagged non-refactor with an offer to do it as a follow-up.
- **Checklist** — the project's own template checklist, all seven boxes checked: informative title, single goal / no scope creep, pre-commit passed, changed functionality tested, new functions documented, TODOs flagged, project documentation updated.

### Acceptance Criteria

| Criterion | Status | Evidence |
|---|---|---|
| Tests added | Done | 15 new tests across `test_driver.py` and `test_adapters.py` |
| All tests passing | Done | 148 passed / 7 skipped vs a 133 / 7 baseline on `main`; suite green at each of the 5 commits |
| Follows style guide | Done | `pre-commit` (ruff check, ruff format, python-ast, ASF license headers) green on all 10 changed files |
| No breaking changes | Done | The node is additive and opt-in; the legacy `Driver` path is unchanged by default, covered by `test_extract_node_templates_omits_result_builder_by_default`. One behavioural side effect — a new DAG template version on first tracked run after upgrading — is documented in the PR and the user docs. Existing versions and their runs are untouched. |
| Documentation updated | Done | `docs/hamilton-ui/ui.rst`, [`12799ae8`](https://github.com/apache/hamilton/pull/1678/commits/12799ae8fc1f93a64f2c26b7f5bb52e81dff712c) |
| Scoped to the issue | Done | 10 files, all in `ui/`, `docs/`; no core `hamilton/` changes, no unrelated reformatting |

### Maintainer Feedback

| Date | From | Feedback | My response |
|---|---|---|---|
| 2026-06-15 | — | Claimed the issue on [#1150](https://github.com/apache/hamilton/issues/1150) and asked for pointers into the codebase. No reply. | Went and read the code myself; posted the result on 07-01 rather than waiting. |
| 2026-07-01 | — | Posted my own analysis on the issue: why the builder never becomes a node, why `materialize()` differs, and two candidate approaches — A (SDK-side synthesis) and B (core-side node injection) — with a stated lean toward A. | Held off implementing until the maintainer picked one, since B would have thrown A's work away. |
| 2026-07-15 | `skrawcz` (maintainer) | "A sounds reasonable." | Took it as approval for the SDK-side approach and started implementing. Every commit on the branch stays inside `ui/`; core `hamilton/` is untouched, which is the constraint that answer imposed. Delivered in [`059e09ba`](https://github.com/apache/hamilton/pull/1678/commits/059e09ba86ac85f016bf5457ffc98cc31d42add5) and [`0e166995`](https://github.com/apache/hamilton/pull/1678/commits/0e166995079b2807a6c85d4162a8f2bb0a450d9b). |
| 2026-07-29 | self (pre-review) | Reviewed my own diff as a maintainer would and found it unreviewable: two oversized commits, four frontend files I hadn't justified. Held it back rather than spend maintainer time on it. | Rebuilt over 08-01 → 08-04 as five per-layer commits and 10 files, submitted as [#1678](https://github.com/apache/hamilton/pull/1678). |
| 2026-08-04 | — | PR [#1678](https://github.com/apache/hamilton/pull/1678) opened, `skrawcz` cc'd. | Awaiting review. Will log any feedback here with the date and the commit that addresses it. |

---

## Learnings & Reflections

### Technical Skills Gained

- **Hamilton's lifecycle-adapter architecture:** how `post_graph_construct`, `pre_graph_execute`, `post_node_execute` and `post_graph_execute` compose into a tracking pipeline, and how the ordering difference between the sync and async drivers turns into a hard product constraint rather than an implementation detail.
- **Content-addressed identity as a correctness concern.** The DAG-hash bug taught me the general shape of it: when a server dedupes on a hash, anything the hash doesn't cover can change without the server noticing, and the failure is silent. Asking "what does this hash *not* cover?" is now a question I'd ask of any caching or dedup layer.
- **Django migration internals.** `Field.non_db_attrs` is why a `choices` change is metadata-only, and `sqlmigrate` is how you prove it instead of asserting it.
- **Profiling a change against the cost the user already pays**, so "is this too expensive?" gets an answer with numbers attached.
- **Threading a feature across the stack:** a Python classification string travelling through a Django enum and a TypeScript union to reach the UI. The more useful half of that was finding out where the codebase made work unnecessary — the frontend went from four files to one because I checked how the UI actually styles nodes instead of assuming it needed help.
- **Writing commits for a reviewer rather than for myself.** One commit per layer, each with a body explaining why the change is there and passing tests on its own, is a different artifact from a working branch, and producing it is a real skill I did not have before this.

### Challenges Overcome

See [Challenges Faced](#challenges-faced) in full. The three that taught me the most: closing my own PR and rebuilding it, catching the DAG-hash bug that would have made the feature silently do nothing for existing users, and finding the name-collision hole in an invariant the docs had convinced me was airtight.

### What I'd Do Differently Next Time

- **Review my own diff as a stranger would, before I think it's done.** The four days I spent rebuilding the first implementation were work the first pass should have absorbed. The trigger is cheap: read the diff top to bottom as a stranger and try to answer "why is this line here?" for each hunk. Anywhere I can't, either the code or the commit message is wrong.
- **Ask the "what does this identity not cover?" question up front.** I found the DAG-hash problem by end-to-end testing against a pre-existing template, which is the expensive way. Reading `register_dag_template_if_not_exists` before writing the node would have found it in five minutes.
- **Don't trust an invariant just because a comment states it.** `_`-prefixed functions don't become nodes, so the name looked safe. The gap was that not every node name comes from a function — visible in the code, invisible in the sentence I had read.
- **Write the design choices down before the code, not alongside it.** This one worked and I would keep it: every question got answered once, and weeks later I could still tell why.
- **Front-load the research that kills assumptions.** Both measurements ran before the code that depended on them, and both changed something: one fixed my model of the backend schema, the other found a bug and forced an explicit scope call.
- **Overlap analysis with something committable.** The 06-16 → 07-29 gap was defensible — I was blocked on an architectural answer — but the tests and fixtures for the reproduction case didn't depend on that answer, and building them during the wait would have given the branch a steadier history and a head start.

---

## Resources Used

- [Issue #1150 discussion](https://github.com/apache/hamilton/issues/1150) — maintainer approval and scope decisions.
- [Hamilton lifecycle hooks reference](https://hamilton.apache.org/reference/lifecycle-hooks/) — hook semantics.
- [Hamilton result builders reference](https://hamilton.apache.org/reference/result-builders/) — what the node this change adds actually stands in for.
- [Hamilton `CONTRIBUTING.md`](https://github.com/apache/hamilton/blob/main/CONTRIBUTING.md) and the repo's PR template — the structure PR #1678 follows.
- `hamilton/io/materialization.py` — the in-repo precedent for injecting a build-result node.
- `hamilton/lifecycle/base.py` (`call_lifecycle_method_sync`) — where the constraint comes from that the tracker cannot detect the builder.
- `ui/backend/server/trackingserver_template/api.py` (`register_dag_template_if_not_exists`) — the hash-matching behaviour behind challenge 2.
- [Django migration docs](https://docs.djangoproject.com/en/5.2/topics/migrations/) — the basis for the zero-SQL claim I then verified with `sqlmigrate`.
- `ui/sdk/tests/test_adapters.py` and `test_driver.py` — the existing mock-client and fixture-graph patterns the new tests extend.
- `ui/test_tracking_simple.py` — the repo's own tracking harness, reused for reproduction instead of writing my own.
