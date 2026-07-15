# Contribution 3: Fold tensor operation

**Contribution Number:** 3  
**Student:** Charitarth  
**Issue:** [Fold tensor operation](https://github.com/tracel-ai/burn/issues/786)  
**Status:** Phase I — In Progress (issue selected, fork set up, environment verified; blocked on resolving an overlap with [#4519](https://github.com/tracel-ai/burn/issues/4519))

---

## Why I Chose This Issue

[Burn](https://github.com/tracel-ai/burn) is a mature deep learning framework written in Rust. It emphasizes a backend-agnostic tensor API where a single operation is implemented once at the tensor level and dispatched to many backends (ndarray, tch, wgpu, cuda, candle, …). 

I chose this since it gives me an opputinity to contribute to a core library in the ecosystem and exposure to working with Rust. 

---

## Understanding the Issue

### Problem Description

Burn supports `unfold` (im2col — extracting sliding local blocks from a batched tensor) but not the inverse `fold` (col2im — summing overlapping blocks back into an image). PyTorch exposes both as a paired API; Burn only has half of it.

### Expected Behavior

A public `fold` tensor operation, mirroring `torch.nn.Fold`: given a `[batch, channels * kernel_h * kernel_w, num_blocks]` tensor plus output size, kernel size, stride, padding, and dilation, it produces `[batch, channels, height, width]` by accumulating overlapping blocks.

### Current Behavior

No `fold` operation exists. Users who need col2im have to hand-roll it with `slice`/`slice_assign` loops, which is slow and not backend-agnostic.

### Affected Components

Based on how `unfold` was wired in PR #819 (paths since moved under `crates/`), a `fold` implementation would touch:

- **`crates/burn-tensor`** — the public API in `tensor/module.rs` and the backend trait in `tensor/ops/modules/base.rs`, plus tests under `tensor/src/tests/module/`.
- **Each backend** — `burn-ndarray`, `burn-candle`, `burn-tch`, `burn-autodiff` (backward pass), and the cubecl/wgpu backends.
- Note: `burn-cubecl` already has an internal `col2im` kernel used by `conv_transpose2d` (`crates/burn-cubecl/src/kernel/conv/conv_transpose2d/col2im.rs`) that a backend impl could reuse.

---

## Reproduction Process

### Environment Setup

Forked and cloned per the [Burn Contributor Book](https://burn.dev/contributor-book/):

```bash
gh repo fork tracel-ai/burn --clone   # origin = my fork, upstream = tracel-ai
cd burn
git checkout -b feat/tensor-fold
```

- **Toolchain:** rustc/cargo 1.96.1 (stable). No `rust-toolchain.toml` pin, so stable is fine.
- **No extra system deps** for the crates `fold` touches — `cargo run-checks` auto-installs `typos-cli` on first use; GPU backends are optional.
- **Build verified:** `cargo check -p burn-tensor -p burn-ndarray` compiles clean (~29s), confirming the toolchain and the fold-relevant crates build.

Pre-commit workflow from the book:

```bash
cargo fmt --all
cargo clippy --fix        # add --allow-dirty if tree is dirty
cargo run-checks          # required before merge (slow: full fmt+lint+test)
```

### Steps to Reproduce

Since this is a missing feature rather than a bug, "reproduction" = confirming the gap:

1. Search the tensor API for a `fold` counterpart to `unfold4d` — none exists in `crates/burn-tensor/src/tensor/module.rs`.
2. Confirm PyTorch exposes `torch.nn.Fold` as the documented inverse of `torch.nn.Unfold`.

### Reproduction Evidence

- **Fork:** `charitarthchugh/burn`, branch `feat/tensor-fold`.
- **My findings:** `unfold` (PR #819) is the file-by-file template to mirror. The internal cubecl `col2im` kernel means a performant backend impl already partly exists, just not exposed as a public op.

---

## Solution Approach

### Analysis

`fold` is the mathematical inverse of `unfold`: where `unfold` gathers sliding blocks, `fold` scatters them back with **addition** at overlaps. The natural, backend-agnostic default implementation is a `scatter(dim, indices, values, Add)` over precomputed index mappings — which is exactly the approach the maintainer endorsed on the related issue #4519.

### Proposed Solution

Mirror the structure of PR #819 (`unfold`): define the op once in the `burn-tensor` backend trait with a default `scatter`-add implementation, expose it in the public tensor module, add the autodiff backward pass, and let backends override with the existing cubecl `col2im` kernel for performance. Keep it to one focused PR.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Add a public `fold` op that reconstructs an image from overlapping column blocks (col2im), the inverse of `unfold4d`.

**Match:** `unfold` (PR #819) is the direct template for the API + per-backend wiring; the internal `col2im` kernel in `burn-cubecl` is the reference for the accumulation math.

**Plan:**

1. Add `fold` to the backend trait in `crates/burn-tensor/src/tensor/ops/modules/base.rs` with a default `scatter`-add implementation.
2. Expose it in the public API (`crates/burn-tensor/src/tensor/module.rs`).
3. Add the autodiff backward pass in `burn-autodiff` (the gradient of fold is unfold).
4. Add tests under `crates/burn-tensor/src/tests/module/` (mirroring `unfold4d.rs`), validating against known PyTorch outputs.
5. Optionally wire a backend override reusing the cubecl `col2im` kernel.

**Implement:** branch `feat/tensor-fold` (work not yet started — see blocker below).

**Review:** follow `CONTRIBUTING.md` — small focused PR, idiomatic Rust, documented public API, regression tests, `cargo run-checks` green.

**Evaluate:** unit tests comparing `fold` output to reference values, and a round-trip sanity check where applicable.

---

## Testing Strategy

### Unit Tests

- [ ] `fold` output matches known reference values (from `torch.nn.Fold`) for a basic case.
- [ ] Overlapping blocks accumulate by addition (stride < kernel size).
- [ ] Stride / padding / dilation variants behave correctly.

### Integration Tests

- [ ] Autodiff: gradient of `fold` equals `unfold` of the upstream gradient.
- [ ] Runs across at least the ndarray and tch backends via Burn's shared test macros.

### Manual Testing

- [ ] Small script folding a hand-computed unfolded tensor, compared against PyTorch.

---

## Implementation Notes

### Week 1 Progress

- Selected #786 after verifying it's open and unclaimed by anyone else
- Forked `tracel-ai/burn`, set up `origin`/`upstream` remotes, created branch `feat/tensor-fold`, and verified the environment builds (`cargo check` on the fold-relevant crates passes).
- **Discovered an overlap:** issue **[#4519 "feat: add fold4d (col2im) operation as inverse of unfold4d"](https://github.com/tracel-ai/burn/issues/4519)** (opened 2026-02-13) is effectively the **same operation** as #786, but scoped in detail (proposed API mirroring `torch.nn.Fold`, `scatter`-add default). It is **actively claimed** by contributor `Capataina`, who has maintainer (`laggui`) buy-in on the approach and commented that they are still working on it. #786 is the vague 2023-era ask; #4519 is the concrete implementation of it.
- **Action taken:** commented on #4519 to flag the overlap with #786 and ask whether Capataina is still active / wants a collaborator, rather than open a competing PR.


---

## Pull Request

**PR Link:** [not yet submitted]

**Status:** Blocked — awaiting response on the #4519 overlap before writing code.

---

## Learnings & Reflections

### Technical Skills Gained

---

## Resources Used

- [Burn Contributor Book](https://burn.dev/contributor-book/) — environment setup and "adding a new operation" guide.
- [PR #819 — "Feat/tensor unfold"](https://github.com/tracel-ai/burn/pull/819) — the reference implementation to mirror.
- [PyTorch `torch.nn.Fold`](https://pytorch.org/docs/stable/generated/torch.nn.Fold.html) — target semantics.
- [Issue #4519](https://github.com/tracel-ai/burn/issues/4519) — the overlapping, more-detailed fold4d issue.
