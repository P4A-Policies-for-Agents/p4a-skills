---
name: p4a-verify-requirements
description: Use when self-checking a P4A (Policies 4 Agents) policy submission against the platform's minimum requirements BEFORE submitting, so the first submission passes validation instead of bouncing. Covers the repo-shape checklist (public GitHub repo containing Makefile, Cargo.toml, src/lib.rs at the policy root or subPath), the pdk >= 1.8 version gate (inline or Cargo workspace inheritance), and the metadata gating (name, description, category from the platform's enum). Run this as the last step before calling submit_policy. Do NOT use for building the repo in the first place (see p4a-build-policy) or for writing policy code (see the omni-gateway-pdk-skills repo).
---

# P4A Verify Requirements

## Overview

<!-- Sub-issue: flesh out. This is the pre-flight checklist an author (or agent)
     runs before submitting, mirroring the checks the platform performs so failures
     surface locally, not at submission time. -->

## When to Use

- As the final gate before submitting a policy to P4A.
- When a submission was rejected and you need to diagnose which requirement failed.

**Do NOT use** for:

- Assembling the repo initially — see [[p4a-build-policy]].
- Writing policy code — see `omni-gateway-pdk-skills`.

## The checklist

<!-- Sub-issue: turn each into a verifiable step.
     1. Repo is PUBLIC on GitHub.
     2. Policy directory (root or subPath) has Makefile, Cargo.toml, src/lib.rs.
     3. Cargo.toml declares pdk >= 1.8.0 (inline or workspace inheritance).
     4. Metadata present: name, description, category (from the enum). -->

## Diagnosing a failed submission

<!-- Sub-issue: map each validation failure to the checklist item that caused it. -->

## Source Ref

<!-- Sub-issue: link the public P4A submission-requirements docs + OpenAPI, with date. -->
