---
name: p4a-build-policy
description: "Use when assembling a GitHub-hosted policy repository so the P4A (Policies 4 Agents) platform can accept it as a submission. Covers the two accepted layouts — the unified model (one root with Makefile, Cargo.toml, src/lib.rs) and the split model (a definition root with gcl.yaml + exchange.json + Makefile and a separate implementation root with Cargo.toml + Makefile + src/lib.rs) — the pdk dependency floor (version >= 1.8, including Cargo workspace inheritance via pdk = { workspace = true }), the public-repo requirement, subPath addressing for policies nested in a monorepo, optional files (definition/gcl.yaml, icon.png/icon.svg), the catalog metadata (name, description, category) a submission carries, and the split-model implementation Makefile build contract (table-form definition_asset_id, cargo-anypoint >= 1.8, GAV-fetch build flow) that the deploy pipeline requires. Explains the two-phase gate — the submit-time shape/PDK check vs the build pipeline that compiles and publishes. Do NOT use for writing the policy's Rust logic (see the omni-gateway-pdk-skills repo), for driving the MCP submit_policy tool (see p4a-mcp-usage), or for the pre-submit self-check (see p4a-verify-requirements)."
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

## The gcl.yaml config definition

`gcl.yaml` is the policy's **configuration schema** (GCL / `PolicyDefinition`) —
it declares the parameters a platform admin fills in when applying the policy,
and drives the config form Anypoint renders. It's a PDK-owned schema (see the
PDK docs for the full spec). Property *names* matter: a name that collides with
a Rust reserved keyword breaks the build pipeline's codegen (see the gotcha
below). Two authoring conventions matter for a well-formed policy:

- **Give each config property a `description`.** Every parameter under
  `spec.properties` should carry a `description` — it's the help text shown next
  to the field in the Anypoint config UI. A parameter with no description ships
  as an unlabelled input, which reads as unfinished. (This is the *property*
  description, distinct from the submission `description` that becomes the
  Exchange doc page — see below.)
- **Mark secrets with the `security:sensitive` characteristic.** Any property
  holding a password, token, key, or certificate must carry the
  `security:sensitive` characteristic so the platform masks it in the UI and
  handles it as a secret rather than plaintext config. Omitting it leaks the
  value into logs and the config form in the clear.

Consult the PDK documentation for the exact GCL keys and characteristic syntax —
the schema lives with the PDK, not with P4A.

### Gotcha: property names must not collide with Rust keywords

A `gcl.yaml` config property named after a Rust reserved keyword (`type`,
`match`, `move`, `ref`, `impl`, `fn`, `loop`, `mod`, …) breaks the **build
pipeline**, not just the submit-time name check. The pipeline runs
`cargo anypoint config-gen` to regenerate `src/generated/config.rs` from the
definition, which emits an invalid struct field (`pub type: ...`) and aborts
codegen. Exact deploy error:

```
gcl.yaml (definition) has a config property whose name collides with a Rust
reserved keyword: 'type'. cargo anypoint config-gen emits an invalid Rust
struct field for it and aborts codegen. Rename the property (e.g. 'type' →
'fieldType') in definition/gcl.yaml and re-deploy.
```

- **Local `make build` will NOT catch this** if the repo checks in a
  hand-maintained `config.rs` (some split-model policies do, when `config-gen`
  can't handle nested objects / DataWeave selectors). The pipeline regenerates
  from `gcl.yaml` regardless, so the local build passes and the deploy fails.
- **Fix:** rename the property (e.g. `type` → `fieldType`) in `gcl.yaml`. The
  offending key is often nested inside an array `items` object — a discriminator
  like `type: selector | value` — so grep the whole file, not just top-level
  properties. Update the matching serde `#[serde(alias = "...")]` and any docs /
  tests that use the old key.

## Gotcha: the split-model implementation Makefile build contract

The build pipeline drives a split-model deploy by publishing the **definition**
to Exchange as a `-dev` asset, patching the **implementation** `Cargo.toml`'s
`definition_asset_id` to point at that published dev version, then running the
implementation's `make publish`. That handshake only works if the implementation
Makefile uses the **GAV-fetch build flow**, and the metadata is shaped the way
`cargo anypoint` expects. Three deploy-time failures come from getting this
wrong — none is caught by a local `make build` (the local build has the
unpatched metadata and a stale checked-in `config.rs`), so they surface only in
the pipeline.

### 1. `definition_asset_id` must be a table, not a bare string

Under `[package.metadata.anypoint]` in the implementation `Cargo.toml`,
`definition_asset_id` must be a `{ name, version }` **table**:

```toml
[package.metadata.anypoint]
group_id = "00000000-0000-0000-0000-000000000000"
definition_asset_id = { name = "my-policy", version = "1.0.0" }
implementation_asset_id = "my-policy-flex"
```

A bare string (`definition_asset_id = "my-policy"`) fails at the
`build-asset-files` wiring step:

```
Implementation Cargo.toml does not declare definition_asset_id.{name,version}
under [package.metadata.anypoint]; cannot wire the definition publish into
make build-asset-files
```

### 2. `cargo-anypoint` must be ≥ 1.8.0

The Makefile's `install-cargo-anypoint` target must install **1.8.0**, not an
older pin (1.6.0 was a common scaffold default):

```makefile
install-cargo-anypoint:
	@cargo install cargo-anypoint@1.8.0
```

cargo-anypoint 1.6.0 cannot parse the table-form `definition_asset_id`; it
stringifies the table to `[object Object]`, so the derived policy-ref name comes
out as `[object Object]-v1-0` and `make publish` aborts:

```
make publish exited 2 during phase 'publish_dev': error: unexpected argument
'Object]-v1-0' found
```

### 3. Use the GAV-fetch build flow, not the old `policy-project` flow

The implementation Makefile must fetch the (pipeline-published) definition from
Exchange by GAV and build against it — the flow working split-model policies
use:

```makefile
DEFINITION_GAV = $(shell cargo anypoint get-policy-definition-gav)

build-asset-files:  ## Retrieves the definition from exchange
	@anypoint-cli-v4 pdk fetch-policy --gav=$(DEFINITION_GAV)
	@cargo anypoint config-gen --output ./src/generated/config.rs

build:
	@cargo build --target $(TARGET) --release
	@cargo anypoint build-policy --target=$(BUILD_POLICY_DIR) --wasm=$(TARGET_DIR)/$(CRATE_NAME).wasm --min-runtime-version=$(MIN_FLEX_VERSION)
	@anypoint-cli-v4 pdk install-policy --target $(TARGET_DIR) --path $(BUILD_POLICY_DIR)/$(CRATE_NAME)/implementation/

publish: build-asset-files build
	@anypoint-cli-v4 pdk policy-wasm publish --wasm=$(TARGET_DIR)/$(CRATE_NAME).wasm --target=$(BUILD_POLICY_DIR)/$(CRATE_NAME) --dev
```

The **old** monolithic flow — where `build` calls
`cargo anypoint gcl-gen -d $(DEFINITION_NAME) ...` and `publish` runs
`anypoint-cli-v4 pdk policy-project publish` — does not resolve `DEFINITION_NAME`
in the split pipeline (the definition lives in a separate root), so the
policy-ref name degrades to `[object Object]-v1-0` and publish fails the same way
as (2). Set `MIN_FLEX_VERSION := 1.9.3` when the policy uses
`enable_stop_iteration` or other ≥1.9 features.

**Why it passes locally but fails in the pipeline:** locally you build against
the unpatched `definition_asset_id` and a checked-in `config.rs`; the pipeline
patches the metadata to the just-published dev version and regenerates, so the
Makefile flow and metadata shape are exercised only on deploy. When migrating a
policy out of a monorepo, port the Makefiles from a known-good split-model repo
rather than trusting the ones that came with the source.

## Recommended: a how-to doc in the repo

Write a consumer-facing how-to in the repo (e.g. `how-to.md`, or a `## Usage`
section of the `README.md`): what the policy does, its config parameters and
their defaults, a minimal apply example, and any caveats. It surfaces on P4A in
**two** places at submission — neither is auto-extracted from a repo file, so
you carry the content across yourself:

- **The submission `description`** (2–120 chars name; **10–20,000-char
  description**) becomes the policy's Anypoint Exchange **asset doc page** —
  P4A generates that page from `description` on deploy (there is no repo
  `home.md` ingestion; the description *is* the doc). Paste the how-to's body
  into `description` so the Exchange/catalog page reads as finished rather than
  empty.
- **The "Examples / how-to URL" field** (optional) takes a **URL**, not a file
  — point it at the GitHub-rendered how-to (e.g. the repo's how-to.md page) or a
  hosted guide. It shows up as a documentation link on the policy detail page.

So: keep the doc in the repo for contributors, mirror its body into
`description` for the Exchange page, and link its URL in the how-to field.

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
- <https://docs.mulesoft.com/pdk/latest/> — the PDK / GCL `PolicyDefinition` schema: config property `description` and the `security:sensitive` characteristic.

_Snapshot: 2026-07-03 (rev. split-model Makefile build-contract findings from a policy migration)._
