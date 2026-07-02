---
name: p4a-build-policy
description: Use when assembling a GitHub-hosted policy repository so the P4A (Policies 4 Agents) platform can accept it as a submission. Covers the canonical PDK policy layout the platform validates for — a Makefile, Cargo.toml, and src/lib.rs — the pdk dependency floor (version >= 1.8, including Cargo workspace inheritance via `pdk = { workspace = true }`), the public-repo requirement, subPath policies inside a monorepo, and the catalog metadata (name, description, category) the submission expects. Do NOT use for writing the policy's Rust logic (see the omni-gateway-pdk-skills repo) or for the pre-submit self-check (see p4a-verify-requirements).
---

# P4A Build Policy

## Overview

<!-- Sub-issue: flesh out. This skill is about the *shape* of a submittable repo,
     not the policy logic. P4A validates the repo via the public GitHub REST API. -->

## When to Use

- Laying out a new policy repo (or a policy subdirectory in a monorepo) for P4A submission.
- Choosing the `pdk` dependency declaration so it clears the version floor.
- Preparing the catalog metadata that accompanies a submission.

**Do NOT use** for:

- Writing the policy's Rust code — see `omni-gateway-pdk-skills`.
- The final pre-submit check — see [[p4a-verify-requirements]].

## Required repo shape

<!-- Sub-issue: the policy directory (repo root or a subPath) must contain
     Makefile, Cargo.toml, src/lib.rs. The repo must be PUBLIC. -->

## The pdk dependency floor

<!-- Sub-issue: Cargo.toml must declare pdk >= 1.8.0. Inline (`pdk = "1.8.0"`,
     `pdk = { version = "..." }`, `[dependencies.pdk]`) or workspace inheritance
     (`pdk = { workspace = true }` resolving to `[workspace.dependencies] pdk`). -->

## Policies inside a monorepo (subPath)

<!-- Sub-issue: a policy nested inside a repo is a first-class citizen; explain the
     tree/<ref>/<subPath> URL form. -->

## Catalog metadata

<!-- Sub-issue: name, description, category (from the platform's category enum). -->

## Source Ref

<!-- Sub-issue: link the public P4A "submitting a policy" docs guide + OpenAPI, with date. -->
