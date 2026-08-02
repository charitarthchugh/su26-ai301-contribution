# Contribution 2: Explicitly show ResultBuilder Node as part of execution in the UI

> Contribution 1 ([vllm-omni #195](contribution_1_vllm_omni_195.md)) was discontinued after the feature turned out to be already resolved upstream. Its writeup lives in [contribution_1_vllm_omni_195.md](contribution_1_vllm_omni_195.md).

**Contribution Number:** 2  
**Student:** Charitarth  
**Issue:** [Explicitly show ResultBuilder Node as part of execution in the UI](https://github.com/apache/hamilton/issues/1150)  
**Status:** Phase III — In Progress (SDK and backend landed; frontend, docs and verification remain)  
**Branch:** [`feature/ResultBuilder-node-in-ui`](https://github.com/charitarthchugh/hamilton/tree/feature/ResultBuilder-node-in-ui)  
**PR:** not opened yet, gated on end-to-end verification in the UI.

---

## Why I Chose This Issue

[Apache Hamilton](https://github.com/apache/hamilton) is a mature ML/data dataflow framework (~2.5k★, maintained by DAGWorks) for expressing data and feature pipelines as a DAG of Python functions. After my first pick ([vllm-omni #195](contribution_1_vllm_omni_195.md)) turned out to be already resolved upstream, I wanted a genuinely available, code-substantive issue, and this one checks every box:

- **Skill match:** I'm comfortable in Python and experienced at wrangling dev environments and deployment setups, which is what standing up Hamilton's docker-compose UI stack and working in the tracking SDK demands — the bulk of the change lives in Python SDK code.
- **Learning goal:** it touches Hamilton's lifecycle-hook / tracking-adapter architecture and spans the full stack (Python capture → React render), which is exactly the kind of end-to-end feature I want experience with.
- **Understanding:** I traced why the node is missing — the `ResultBuilder` is a lifecycle adapter, not a graph node, so the tracking hooks never see it — and mapped the fix across the SDK, backend, and frontend (see [Solution Approach](#solution-approach)). It also has real project impact: users would finally see the final artifact a run produced and compare outputs across runs.

---

## Understanding the Issue

### Problem Description

In Apache Hamilton, a `ResultBuilder` assembles the final result of a DAG run, e.g. compiling node outputs into a Pandas DataFrame, a dictionary, or a custom object. It runs as part of the execution lifecycle, but the UI never shows it — the execution graph just ends at the terminal nodes, so the actual artifact a run produced is invisible. This matters because the whole point of the Hamilton UI is run observability, and the final output is the one thing every user cares about. I chose it because the fix plays to my Python and environment-setup strengths while stretching me across the full stack.

This omission makes it difficult for users to:

1. Explicitly see what final artifact a run produced.
2. Understand how the final output was constructed from the terminal nodes of the DAG.
3. Compare the final outputs of different execution runs side-by-side.

### Expected Behavior

The UI should explicitly show the `ResultBuilder` node as the final node in the execution graph, with:

- Incoming edges from all the terminal nodes (final variables) that fed into it.
- Its output type, schema, and summary statistics (data observability) displayed when selected.
- The ability to compare the final built result across different executions.

### Current Behavior

The `ResultBuilder` node is omitted from the execution graph visualization in the UI. The graph simply ends at the terminal nodes, and the final built result itself is not explicitly shown as a node with its own data observability metadata.

### Affected Components

- **SDK (Python):** `hamilton/ui/sdk/src/hamilton_sdk/adapters.py` (specifically `HamiltonTracker` and `AsyncHamiltonTracker`) and `hamilton/ui/sdk/src/hamilton_sdk/driver.py` (where node templates are extracted).
- **Backend (Python/Django):** `hamilton/ui/backend/server/trackingserver_template/models.py`, where the `NodeType` enum backs the `classifications` field, plus its migrations. No table changes, but a new classification value has to be a listed choice.
- **Frontend (React/TypeScript):** The DAG visualization components (`DAGViz.tsx`) which dynamically render nodes and edges based on the logged execution data.

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

OpenAPI endpoint output:

```json
{"openapi":"3.1.0","info":{"title":"NinjaAPI",...},
 "paths":{"/api/v1/metadata/attributes/schema":{"get":{...}}}}
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
- `graph_utils.py:39` excludes `_`-prefixed functions from becoming nodes, so the name `_result_builder` is collision-proof against any user transform.

### Proposed Solution

Synthesize a `_result_builder` node in the tracking SDK: register it as a node template when the tracker sees the graph, and report the combined result as that node's output when the run finishes, with the run's `final_vars` as its realized dependencies. The backend gains one `NodeType` choice so the classification validates; the frontend gains one member in its `Classification` union and nothing else. Core Hamilton stays untouched.

There were two ways to do this: a synthetic node added by the SDK, or a real graph node injected in core the way `materialize()` does it. Core would change `execute()` semantics for every user, so I asked the maintainer on the issue before proceeding. He approved the SDK-side approach ("option A") on 2026-07-15.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** the ResultBuilder combines `execute()` outputs into the single object the caller gets back. It runs as a lifecycle adapter rather than a graph node, so the tracker never registers it and the UI has nothing to draw.

**Match:** materializers are the closest thing in the codebase to "show a build step as a node", since `materialize()` injects a real `_build_result` node into the graph. On the tracking side, `_extract_node_templates_from_function_graph` already registers nodes and `runs.process_result` + `update_tasks` already capture per-run output, which is everything this node needs. For tests, `ui/sdk/tests/test_adapters.py` has an established mock-client pattern.

**Plan.** I settled the design questions and wrote the answers down before writing any code, so each one got decided once rather than re-argued halfway through an implementation. The choices the code depends on:

- The node is emitted on every tracked run, on all `post_graph_execute` paths. The second analysis fact above forces this. It makes `raw_execute()` and `materialize()` runs slightly wrong, which the PR documents rather than guards against.
- The node template carries no dependencies. `realized_dependencies` is set per run from `final_vars`, because the requested outputs change from run to run.
- Reuse `runs.process_result` for the result summary instead of writing a second profiler.
- Add a `result_builder` choice to the backend `NodeType`, with one no-op postgres migration and an in-place edit to the squashed sqlite `0001_initial`.
- Failed runs emit no task run at all, so the node falls back to the UI's existing "not executed" rendering.
- One PR, SDK and UI only. Living under `ui/sdk/` does not put a change in scope, and `hamilton/` core is off-limits.
- Name the node `_result_builder`. Template output type is `typing.Any`; the real per-run type goes in the result summary.
- The frontend gets the `Classification` union member and nothing else: no styling, no icons, no filtering.
- Document the rollout wrinkle below rather than trying to fix it.
- Add a short section to `docs/hamilton-ui/ui.rst`, and keep the tests SDK-side only.

That rollout wrinkle is the one I expect to generate support questions. `hash_dag` hashes only the real `FunctionGraph`, and `trackingserver_template/api.py:183` short-circuits template registration on `(project_id, dag_hash, name, code_hash)`. Existing DAG templates therefore never pick up the node. It shows up only on newly registered versions, which in practice means after the user's own code changes.

That gives the build order:

1. Kill the risky assumptions first. Is the backend migration really zero-SQL? Does reusing `process_result` on the combined result cost anything worth caring about?
2. SDK template: append the synthetic node in `_extract_node_templates_from_function_graph`.
3. SDK task run: stash `final_vars` in `pre_graph_execute`, then emit one `update_tasks` call from `post_graph_execute`. Sync tracker first, async after.
4. Backend: the `NodeType` choice and both migration paths.
5. Frontend: one union member.
6. Tests and docs, end-to-end verification in the UI, then the PR.

**Implement:** [`feature/ResultBuilder-node-in-ui`](https://github.com/charitarthchugh/hamilton/tree/feature/ResultBuilder-node-in-ui), currently commits [`d554b294`](https://github.com/charitarthchugh/hamilton/commit/d554b294) (backend) and [`421ff29b`](https://github.com/charitarthchugh/hamilton/commit/421ff29b) (SDK template). The task-run emission is written and verified but not yet committed. Details in [Implementation Notes](#implementation-notes).

**Review:** pre-commit (ruff check, ruff format, python-ast, ASF license headers) is green on every changed file. Every step records its verification evidence before I move on, and I check `git status` against the intended file list every time, because `uv run` inside `ui/backend` quietly rewrites `uv.lock` and that must not ship.

**Evaluate:** red/green unit tests asserting the exact `update_tasks` payloads, `sqlmigrate` output for the migration, `makemigrations --check` for sqlite consistency, the full SDK suite against baseline, and a real tracked run rendering in the local UI before the PR opens.

---

## Testing Strategy

### Unit Tests

SDK only. I added these to the existing suites rather than starting a new parallel file: `test_driver.py` covers node-template synthesis, `test_adapters.py` covers tracker emission.

- [x] `test_extract_node_templates_appends_result_builder` (`test_driver.py`). Runs against a real `FunctionGraph` built from the repo's `tests/resources/basic_dag_with_config.py` fixture. It asserts the template list grows by exactly one, checks the appended template's fields, and confirms `_result_builder` never enters `fg.nodes`. Red-green checked: comment out the `append` and you get `1 failed, 7 passed`; put it back and `8 passed`.
- [x] `test_result_builder_node_does_not_affect_dag_hash` (`test_driver.py`). Pins the guarantee that the synthetic node stays out of `hash_dag`. Worth being honest about this one: it passes with or without my change, by design. It exists to fail if some future change starts feeding the node into the hash.
- [x] `test_result_builder_task_run_is_emitted` (`test_adapters.py`). Exactly one `_result_builder` task update, `realized_dependencies == ["a", "b", "c"]`, status SUCCESS. This needed a `RecordingClient` that keeps every task update, since the existing `MockHamiltonClient` only keeps the latest. Red-green checked: drop the three-line `elif` arm and it fails.
- [x] `test_result_builder_task_run_not_emitted_on_failure` (`test_adapters.py`). A `should_fail=True` run emits nothing. This is the failed-run guard, and the thing most likely to break quietly later.
- [ ] Async parity tests, once the async tracker lands.

### Integration Tests

- [x] Full SDK suite: 137 passed, 7 skipped, against a 135 / 7 baseline. The two extra are the new adapter tests. Four modules fail collection on `ModuleNotFoundError: polars / pydantic`, which is a missing optional dep in my venv and has nothing to do with the diff.
- [x] The migration really does emit no SQL. `sqlmigrate trackingserver_template 0003` gives `BEGIN; -- Alter field classifications on nodetemplate -- (no-op) COMMIT;`, run against the generated migration rather than the throwaway I used to check the assumption up front.
- [x] Sqlite consistency: `makemigrations --check --dry-run` under `server.settings_mini` reports `No changes detected`. So the in-place edit to the squashed `0001_initial` genuinely resynchronises Django's model state, and Django does not want a `0002`.
- [ ] A tracked run round-tripping a `result_builder` classification through the API. That needs the task-run emission plus the frontend union member, so it is still ahead of me.

### Manual Testing

- [x] Baseline: a tracked DAG run renders fully in the local UI (run 3, project 2). That is what makes the missing node observable in the first place.
- [ ] Before the PR opens. Does `_result_builder` appear in the run view with a populated result summary? Does it render sensibly given it has no template dependencies? Does a failed run show it as not-executed rather than broken? Can two runs of the same DAG be compared on it, which is the whole point of the issue?
- [ ] Still to check: how a template node with no task run actually renders. I am assuming it inherits sane "not executed" styling. If it looks confusing, or breaks when `realized_dependencies` is missing, the failed-run choice has to be revisited and the fallback is an explicit FAILURE or SKIPPED task run.

---

## Implementation Notes

### Phase III Progress

Landed so far: the local stack verified against a live SDK checkout, both risky assumptions measured, the SDK node template, the sync tracker's task-run emission, and the backend `NodeType` with both migration paths.

Still ahead: async tracker parity, the frontend union member, async tests, the docs section, end-to-end verification in the UI, and the PR itself.

### Challenges Faced

1. **Checking a documented behaviour instead of trusting it.** Hand-editing an already-applied squashed sqlite migration is only safe if adding a `choices` value emits no SQL. Django has treated `choices` as a non-DB attribute since 4.1, but I had reasoned that from the docs and never run it. Running it confirmed the no-op on Django 5.2.15, and corrected my own mental model on the way: there is no scalar `node_type` column at all, so the generated migration is `0003_alter_nodetemplate_classifications`. Skip this step and I would have hand-edited an applied migration on faith.
2. **A measurement that changed nothing, and one that nearly changed the scope.** Reusing `process_result` on a `DictResult` re-profiles outputs that were already profiled individually. Measured, it costs exactly twice the time and twice the payload, flat across three orders of magnitude, and it is bounded: payload tracks column count rather than rows (1k rows gives 6,517 B, 1M rows gives 6,798 B), and `MAX_DICT_LENGTH_CAPTURE = 100` really does cap it (100 DataFrames at 189,846 B, 300 at 189,735 B). So no size guard, and the reuse survives. What the benchmark did turn up was a pre-existing bug. `compute_stats_dict:106-109` tries `json.dumps` first and, if that works, stores the value verbatim, skipping every truncation guard. A 1M-element list costs 1,182 B as a bare node output and 20,630,689 B inside a dict. That is 17,454× amplification, measured rather than extrapolated.
3. **Deciding not to fix that bug.** Harder call than it sounds. `compute_stats_list:207-210` has the identical two lines with the identical comment, so it is a consistent design choice by the maintainers rather than an oversight. And the worst case you can already reach today, a list of 100 × 100k lists at 103 MB, is worse than anything my change causes. Fixing it would mean reverting a maintainer decision in a file #1150 never mentions, inside a PR about something else. It goes upstream as its own issue instead, and the docs section points at the knobs (`HAMILTON_MAX_LIST_LENGTH_CAPTURE`, `HAMILTON_CAPTURE_DATA_STATISTICS`) so anyone with large list outputs can find them.
4. **The sync/async hook-ordering asymmetry, still open.** The sync driver fires `post_graph_execute` after `do_build_result`, so `results` is the combined object. The async driver fires it inside `raw_execute`'s `finally`, before the build. The async tracker cannot see the built output without core changes, and core is off the table. Async will therefore land as a status-only update, with the limitation documented rather than papered over.
5. **Toolchain traps worth writing down.** Generated Django migrations have no ASF license header, so `pre-commit` fails once and the `insert-license` hook writes it for you; re-run to go green. `uv run` inside `ui/backend` quietly rewrites `uv.lock` with re-resolution churn and a package version bump, which the repo reserves for release commits, so it has to be reverted every time. And `FunctionGraph.from_modules(mod, config=...)` must be called with no adapter at all: passing `base.DefaultAdapter()` dies at `graph.py:78` with `AttributeError: 'DefaultAdapter' object has no attribute 'does_method'`, because it wants a `LifecycleAdapterSet`.

### Code Changes

Committed, [`d554b294`](https://github.com/charitarthchugh/hamilton/commit/d554b294) `Add result_builder node classification to the tracking server`:

- `ui/backend/server/trackingserver_template/models.py` adds `result_builder = "result_builder", _("ResultBuilder")` to `NodeType`.
- `.../migrations/0003_alter_nodetemplate_classifications.py` was generated by Django rather than hand-written. One `AlterField`, zero SQL.
- `.../migrations_sqlite/0001_initial.py` gains one line in the inline `choices` list, following the squashed convention.

Committed, [`421ff29b`](https://github.com/charitarthchugh/hamilton/commit/421ff29b) `Synthesize a _result_builder node template in the SDK`:

- `ui/sdk/src/hamilton_sdk/driver.py` gets a `RESULT_BUILDER_NODE_NAME` constant, so the emission code imports the name instead of re-typing the string, and `_result_builder_node_template()`, appended in `_extract_node_templates_from_function_graph`. It builds the dict directly instead of extending `_convert_classifications`, because there is no `Node` object to hand that function; bypassing it was the smaller diff.
- `ui/sdk/tests/test_driver.py` gets two tests.

Written and verified, not yet committed:

- `ui/sdk/src/hamilton_sdk/adapters.py` stashes `final_vars` per `run_id` in `pre_graph_execute`, and calls a new public `emit_result_builder_task_run()` from the success arm of `post_graph_execute`. Emitting nothing on failure falls out of the existing control flow, so it needs no second condition. Timestamps are a zero-width span at the real `finally`-block instant: no hook brackets the builder, so its duration is not observable, and a real instant beats a made-up duration. `process_result` wants a `Node`, so it gets a minimal stand-in, since it only reads `.name` and `.tags`. To keep the diff inside the issue's scope I worked that seam from the calling side instead of touching `data_observation.py`.
- The same file's `post_node_execute` had a 25-line attribute-shaping block that was already duplicated in the async twin. Emitting the same shape a third time would have made three copies of it, so it now lives in three module-level helpers shared by all three call sites.
- `ui/sdk/tests/test_adapters.py` gets two tests.

Approach decisions:

- SDK-side synthesis over core node injection. The maintainer approved it, and injecting in core would change `execute()` semantics for everyone.
- The node is unconditional, which is not a shortcut. `call_lifecycle_method_sync` unpacks exactly one `do_build_result` adapter and no hook receives the adapter set, so the tracker cannot detect a builder. The resulting `raw_execute()` and `materialize()` inaccuracy gets documented in the PR, because the check I would want to write cannot be written correctly.
- The two measurements scheduled ahead of the code they de-risk, so an unsafe migration edit or a bad profiling reuse would surface before implementation instead of after.


---

## Learnings & Reflections

### Technical Skills Gained

- Hamilton's lifecycle-adapter architecture: how `post_graph_construct`, `pre_graph_execute`, `post_node_execute` and `post_graph_execute` compose into a tracking pipeline, and how the ordering difference between the sync and async drivers turns into a hard product constraint rather than an implementation detail.
- Django migration internals. `Field.non_db_attrs` is why a `choices` change is metadata-only, and `sqlmigrate` is how you prove it instead of asserting it.
- Profiling a change against the cost the user already pays, so "is this too expensive?" gets an answer with numbers attached.
- Threading a feature across the stack: a Python classification string travelling through a Django enum and a TypeScript union to reach the UI. The more useful half of that was finding out where the codebase made work unnecessary.

### Challenges Overcome

See [Challenges Faced](#challenges-faced): verifying the migration no-op instead of trusting the docs, measuring the double-profiling cost, deciding against fixing an out-of-scope bug I found while measuring, the sync/async hook asymmetry, and the license-header, `uv.lock` and `FunctionGraph` traps.

### What I'd Do Differently Next Time

- Write the design choices down before the code, not alongside it. Every question then got answered once, and weeks later I could still tell why. It also shrank the frontend work from four files to one, because writing it down forced me to go and check how the UI actually styles nodes instead of assuming it needed help.
- Front-load the research that kills assumptions. Both measurements ran before the code that depended on them, and both changed something: one fixed my model of the backend schema, the other found a bug and forced an explicit scope call. Doing them afterwards would have meant rework in the first case and a much more tempting scope creep in the second.
- Commit per unit of work rather than in one push at the end. It keeps progress reviewable and each change bisectable.

---

## Resources Used

- [Issue #1150 discussion](https://github.com/apache/hamilton/issues/1150) — maintainer approval and scope decisions.
- [Hamilton lifecycle adapters documentation](https://hamilton.apache.org/concepts/customizing-execution/) — hook semantics.
- `hamilton/io/materialization.py` — the in-repo precedent for injecting a build-result node.
- `hamilton/lifecycle/base.py` (`call_lifecycle_method_sync`) — where the constraint comes from that the tracker cannot detect the builder.
- [Django migration docs](https://docs.djangoproject.com/en/5.2/topics/migrations/) — the basis for the zero-SQL claim I then verified with `sqlmigrate`.
- `ui/sdk/tests/test_adapters.py` and `test_driver.py` — the existing mock-client and fixture-graph patterns the new tests extend.
- `ui/test_tracking_simple.py` — the repo's own tracking harness, reused for reproduction instead of writing my own.
