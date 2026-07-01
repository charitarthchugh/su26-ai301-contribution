# Contribution 2: Explicitly show ResultBuilder Node as part of execution in the UI

> Contribution 1 ([vllm-omni #195](contribution_1_vllm_omni_195.md)) was discontinued after the feature turned out to be already resolved upstream. Its writeup lives in [contribution_1_vllm_omni_195.md](contribution_1_vllm_omni_195.md).

**Contribution Number:** 2  
**Student:** Charitarth  
**Issue:** [Explicitly show ResultBuilder Node as part of execution in the UI](https://github.com/apache/hamilton/issues/1150)  
**Status:** Phase III — In Progress

---

## Why I Chose This Issue

[Apache Hamilton](https://github.com/apache/hamilton) is a mature ML/data dataflow framework (~2.5k★, maintained by DAGWorks) for expressing data and feature pipelines as a DAG of Python functions. After my first pick ([vllm-omni #195](contribution_1_vllm_omni_195.md)) turned out to be already resolved upstream, I wanted a genuinely available, code-substantive issue, and this one checks every box:

- **Real project impact:** it improves observability of pipeline runs in the Hamilton UI — users would be able to see the final artifact a run produced and compare outputs across runs.
- **Good learning surface:** it touches Hamilton's lifecycle-hook / tracking-adapter architecture and spans the full stack (Python capture → React render), which is exactly the kind of end-to-end feature I want experience with.

---

## Understanding the Issue

### Problem Description

In Apache Hamilton, a `ResultBuilder` assembles the final result of a DAG run, e.g. compiling node outputs into a Pandas DataFrame, a dictionary, or a custom object. It runs as part of the execution lifecycle, but the UI never shows it.

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
- **Backend (Python/Django):** Django models/API capturing and storing the execution runs (no schema changes needed, as the backend already supports arbitrary nodes and dependencies).
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

---

## Solution Approach (Phase 2)

### Analysis

The `ResultBuilder` is a lifecycle method/adapter in Hamilton, not a standard function/node in the DAG. Therefore, it is not included in `FunctionGraph.nodes` and is not captured by the `pre_node_execute` and `post_node_execute` hooks of the tracking SDK (`HamiltonTracker` and `AsyncHamiltonTracker`).

So to show the `ResultBuilder` in the UI, I need to:

1. **Detect it:** figure out which `ResultBuilder` the current graph is using.
2. **Register a node template:** add one for the `ResultBuilder` to the DAG template registered in `post_graph_construct`.
3. **Track the execution:** log when the `ResultBuilder` starts and finishes, capturing its inputs (the graph's terminal nodes / final variables) and its output (the built result).
4. **Add observability:** compute summary stats and schema for the built result and hang them off the `ResultBuilder` node.

### Proposed Solution

I'd do this entirely from the SDK and the frontend, and not touch core Hamilton. Changing execute() itself would change behavior for every user.

- Reproduce it first: a tracked pipeline using plain .execute([...]) with a few outputs, then check the UI and confirm the combined result has no node. 
- Ask Stefan on the issue whether the synthetic-node approach is fine. He described the builder as already part of the graph, but the execute() path doesn't actually work that way.
- SDK is where most of the work lives, in three spots: add the builder node in post_graph_construct (its deps are the requested final vars), capture the final results object in post_graph_execute  and log it through the existing update_tasks call, and teach _convert_classifications about result_builder so it doesn't get tagged a plain transform.
- Frontend: add result_builder to the Classification union, then give the node an icon and border in the two node components. Reusing the materializer artifact styling is the least effort. Add a legend entry too. The output panel likely needs nothing, since it just renders whatever the SDK logged.
- Backend: I think nothing. Node storage is generic enough already. The one optional bit is adding result_builder to the NodeType enum, and that's hygiene, not a blocker.
- Verify by rerunning the repro and watching the node and its output appear.

---
Currently waiting on maintainer in order to have correct approach. Have created a basic issue reproduction pipeline locally (do not want to commit so it does not create conflicts). 