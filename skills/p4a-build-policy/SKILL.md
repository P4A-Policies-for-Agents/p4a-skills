---
name: p4a-build-policy
description: Use when assembling a GitHub-hosted policy repository so the P4A (Policies 4 Agents) platform can accept it as a submission. Covers the two accepted layouts — the unified model (one root with Makefile, Cargo.toml, src/lib.rs) and the split model (a definition root with gcl.yaml + exchange.json + Makefile and a separate implementation root with Cargo.toml + Makefile + src/lib.rs) — the pdk dependency floor (version >= 1.8, including Cargo workspace inheritance via `pdk = { workspace = true }`), the public-repo requirement, subPath addressing for policies nested in a monorepo, optional files (definition/gcl.yaml, icon.png/icon.svg), and the catalog metadata (name, description, category) a submission carries. Explains the two-phase gate: the submit-time shape/PDK check vs the build pipeline that compiles and publishes. Do NOT use for writing the policy's Rust logic (see the omni-gateway-pdk-skills repo), for driving the MCP submit_policy tool (see p4a-mcp-usage), or for the pre-submit self-check (see p4a-verify-requirements).
---

# P4A Build Policy

## Overview

This skill is about the **shape of a submittable policy repo** — the files P4A
looks for, where they live, and the `pdk` version floor — not the policy's Rust
logic. Get the shape right and the submission clears validation; get the Rust
right (a separate concern) and the build pipeline compiles and publishes it.

Two things gate a submission, in order:

1. **Submit-time check** — synchronous, over the public GitHub API. Confirms the
   repo is public, the required files exist at the expected paths, and the `pdk`
   dependency meets the floor. A failure here rejects the submission immediately.
2. **Build pipeline** — asynchronous, after the submission is accepted and a
   deploy is requested. Clones the ref, compiles to WASM, and publishes to
   Anypoint Exchange. Shape problems that slip past step 1 (a Makefile target
   that doesn't build, an unparseable `gcl.yaml`) surface here.

This skill covers step 1 — what to put in the repo so it passes. See
[[p4a-verify-requirements]] for the pre-submit self-check that mirrors it.

## When to Use

- Laying out a new policy repo (or a policy subdirectory in a monorepo) for P4A.
- Choosing between the unified and split layouts.
- Declaring the `pdk` dependency so it clears the version floor.
- Preparing the catalog metadata that accompanies a submission.

**Do NOT use** for:

- Writing the policy's Rust code — see `omni-gateway-pdk-skills`.
- Calling the `submit_policy` MCP tool — see [[p4a-mcp-usage]].
- The final pre-submit check — see [[p4a-verify-requirements]].

## The repo must be public

The submit-time check fetches the repo over the public GitHub API. A **private
repo is rejected** ("Repository must be public"); a missing/unreachable repo
reads the same as private ("not found or not public"). Make the repo public
before submitting — there's no token-based private-repo path.

## Two accepted layouts

Pick one. The submission declares which via its `projectType` (`unified` is the
default; `split` requires a second GitHub URL for the implementation root).

### Unified model — one root

All three files live together, at the **repo root or a subPath**:

- `Makefile`
- `Cargo.toml`
- `src/lib.rs`

### Split model — definition + implementation roots

Two roots (two URLs), typically in the same repo at different subPaths:

- **Definition root:** `gcl.yaml`, `exchange.json` (must carry an `assetId`), `Makefile`
- **Implementation root:** `Cargo.toml`, `Makefile`, `src/lib.rs`

Use split when the policy's Exchange definition is authored and versioned
separately from its Rust implementation; use unified otherwise.

## subPath addressing (monorepos)

A policy nested inside a larger repo is first-class. Address it with a GitHub
tree URL:

```
https://github.com/<owner>/<repo>/tree/<ref>/<subPath>
```

- `<ref>` is a branch or tag and **may itself contain slashes** (e.g. a
  `release/1.8.0` branch) — the platform disambiguates ref-vs-subPath against
  the repo's actual refs.
- `<subPath>` is the directory holding the required files (the unified root, or
  either split root).
- Omitting `/tree/<ref>/...` targets the repo root on the default branch.

## The pdk dependency floor

`Cargo.toml` must declare a `pdk` dependency resolving to **version ≥ 1.8.0**.
Accepted declaration forms:

```toml
# inline string
pdk = "1.8.0"

# inline table
pdk = { version = "1.8.0" }

# section form
[dependencies.pdk]
version = "1.8.0"

# workspace inheritance — resolved from the workspace root's
# [workspace.dependencies] pdk entry
pdk = { workspace = true }
```

Version constraints are read leniently — `1.8`, `^1.8.0`, `~1.8`,
`>=1.8.0,<2` all parse; only the effective major.minor.patch is compared to the
floor. A version that can't be parsed, or resolves below 1.8.0, is rejected.

**Workspace inheritance caveat:** `pdk = { workspace = true }` only works when
the `Cargo.toml` sits in a **subPath** with an ancestor `Cargo.toml` that
defines `[workspace.dependencies] pdk`. A `workspace = true` declaration at the
repo root has nowhere to inherit from and is rejected — declare `pdk` inline
there instead.

## Optional files

- **`definition/gcl.yaml`** (unified model) — warned-if-absent, not a hard
  blocker at submit time, but the policy's config schema comes from it, so ship
  it. (In the split model the definition root's `gcl.yaml` is required, above.)
- **`icon.png` or `icon.svg`** at the policy root — auto-discovered and used as
  the catalog icon (probed in that order). Absent is fine; the catalog falls
  back to a default glyph.

## Catalog metadata

A submission carries, alongside the GitHub URL(s):

- **name** — 2–120 chars.
- **description** — 10–20,000 chars.
- **category** (optional) — one of: `Security`, `Quality of Service`,
  `Transformation`, `Compliance`, `GraphQL`, `Troubleshooting`, `A2A`,
  `A2A and MCP`, `MCP`, `LLM`, `SSE`, `Other`. ("All Categories" is a UI filter
  token, not a storable category.)

Optional extras a submission may include: a video-tutorial URL, an examples URL,
an icon-URL override (or a flag forcing the default glyph), and a link to an
approved Policy Idea it delivers.

## Source Ref

Derived from the public P4A documentation:

- <https://docs.p4a.ai/docs/guides/submitting-a-policy> — required files, unified vs split, `pdk` ≥ 1.8, workspace inheritance, public-repo requirement, categories.

_Snapshot: 2026-07-02._
