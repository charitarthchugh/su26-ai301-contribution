# Contribution 1: [Feature]: Support Container Image Format -> ARM64

**Contribution Number:** 1  
**Student:** Charitarth
**Issue:** [[Feature]: Support Container Image Format -> ARM64](https://github.com/vllm-project/vllm-omni/issues/195)
**Status:** Discontinued — issue already resolved upstream (see [Week 2 Progress](#week-2-progress)). Pivoted to [Contribution 2: apache/hamilton #1150](README.md).

## Why I Chose This Issue

This issue is in a project that interests me - I would like to test out high performance serving for omni-modal models - especially because I have the hardware that is impacted by this issue. Additionally it does not seem paticularly hard and I have a lot of experience solving environment problems.

## Understanding the Issue

### Problem Description

The project is missing an arm64 container build, which is useful for Nvidia edge devices (which are all arm) and potentially even to those on GB100/200/300.

### Expected Behavior

Users are able to run the project with docker on arm64

### Current Behavior

Image does not exist

### Affected Components

Would need to create a new Dockerfile in `docker/`

---

## Reproduction Process

### Environment Setup

Setup docker + nvidia container toolkit on DGX Spark

### Steps to Reproduce

1. Pull latest vllm-omni image
2. Run vllm-omni with a model, e.g Qwen3-Omni
3. Server starts sucessfully, inference works

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

## Implementation Notes

### Week 1 Progress

Claimed the issue upstream ([comment, 2026-06-04](https://github.com/vllm-project/vllm-omni/issues/195)) and started setting up to reproduce. The plan was to confirm that `vllm-omni` ships no arm64 (aarch64) container image and then add a Dockerfile / multi-arch build under `docker/` so the project can run on Nvidia arm64 hardware (Jetson, GB10, DGX Spark).

### Week 2 Progress

**Outcome: this issue was already fixed upstream — there is nothing left to build — so I am discontinuing this contribution and pivoting to a new one.**

While reproducing, I found that aarch64 (arm64) image builds had already been added to the release pipeline by **[PR #3428 — "[CI/Build] Unify release pipeline with NIGHTLY=1 option, add x86_64/aarch64 image builds"](https://github.com/vllm-project/vllm-omni/pull/3428)**, which **merged on 2026-05-17** — roughly three weeks _before_ I claimed the issue (2026-06-04). The feature requested in #195 (official arm64 container images for arm64 devices like Jetson / GB10 / DGX Spark) is therefore already delivered on `main`; the issue had simply never been closed, because the merged PR didn't reference it with a closing keyword.

I [commented on the issue (2026-06-08)](https://github.com/vllm-project/vllm-omni/issues/195) recommending it be closed and that a short documentation update be made to clear up the confusion in the thread (several users were still reporting the missing image).

**Decision:** rather than spend a contribution on a docs-only cleanup of an already-resolved feature request, I selected a new, code-substantive issue — **[apache/hamilton #1150](https://github.com/apache/hamilton/issues/1150)** ("Explicitly show ResultBuilder Node as part of execution in the UI") — and documented its writeup in **[README.md](README.md)**.

**Lesson learned:** a merged PR doesn't always auto-close its related issue, so an open issue can look claimable when the work is already done. Before starting, verify against _merged PRs_, not just the issue's open/closed state.
