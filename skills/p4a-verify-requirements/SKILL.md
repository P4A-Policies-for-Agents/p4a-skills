---
name: p4a-verify-requirements
description: "Use when self-checking a P4A (Policies 4 Agents) policy submission against the platform's requirements BEFORE calling submit_policy or the submit form, so the first attempt passes instead of bouncing. Covers the submission metadata rules (name 2-120 chars, description 10-20000 chars, valid githubUrl — plus implementationGithubUrl for split projects, category from the 12 storable values, and the optional videoTutorialUrl / examplesUrl / iconUrl / iconDisabled / deliversIdeaId / projectType fields), the caveat that category is NOT enum-validated server-side (a quality convention, not a hard gate), the repo-shape + pdk>=1.8 checks it defers to p4a-build-policy, and a runnable pre-submit checklist plus a failed-submission diagnosis map. Run this as the last step before submitting. Do NOT use for assembling the repo layout in the first place (see p4a-build-policy), for driving the submit_policy tool (see p4a-mcp-usage), or for writing policy Rust code (see the omni-gateway-pdk-skills repo)."
---

# P4A Verify Requirements

## Overview

The pre-flight check an author (or their agent) runs **immediately before**
submitting, so a failure surfaces here instead of as a rejected submission. It
mirrors what the platform validates: the **metadata fields** (this skill's
focus) plus the **repo shape + `pdk` floor** (owned by [[p4a-build-policy]] —
this skill defers there rather than duplicating).

Two things gate a submission, in order: the synchronous submit-time check
(public repo, required files, `pdk` ≥ 1.8, and the metadata field rules below),
then the async build pipeline on deploy. This skill is about clearing the first.

## When to Use

- The final gate before calling `submit_policy` or the submit form.
- Diagnosing which requirement a rejected submission missed.

**Do NOT use** for:

- Assembling the repo layout initially — see [[p4a-build-policy]].
- Calling the `submit_policy` tool — see [[p4a-mcp-usage]].
- Writing policy Rust code — see `omni-gateway-pdk-skills`.

## Submission metadata rules

Every field is trimmed before validation. Required unless marked optional.

| Field | Rule |
| --- | --- |
| `name` | **Required.** 2–120 chars. |
| `description` | **Required.** 10–20,000 chars. Markdown allowed — this becomes the Anypoint Exchange asset doc page on deploy (see [[p4a-build-policy]]), so write it for the consumer. |
| `githubUrl` | **Required.** Valid URL to a **public** repo (see repo-shape checks below). |
| `implementationGithubUrl` | **Required iff `projectType` is `split`** — the implementation root's URL. Omit for unified. |
| `projectType` | Optional, `unified` (default) or `split`. |
| `category` | Optional string. **Recommend** one of the 12 storable values below. |
| `videoTutorialUrl` | Optional. Valid URL or empty. |
| `examplesUrl` | Optional. Valid URL or empty — the "Examples / how-to URL" link on the policy page. |
| `iconUrl` | Optional. Valid URL override; otherwise an `icon.png`/`icon.svg` in the repo is auto-discovered. |
| `iconDisabled` | Optional boolean. `true` forces the default glyph. |
| `deliversIdeaId` | Optional. Must reference an **approved** Policy Idea; a non-approved id is rejected. |

### Category — recommended, not enforced

Storable values: `Security`, `Quality of Service`, `Transformation`,
`Compliance`, `GraphQL`, `Troubleshooting`, `A2A`, `A2A and MCP`, `MCP`, `LLM`,
`SSE`, `Other`. (`All Categories` is a UI filter token, not a storable value.)

**Caveat:** `category` is *not* enum-validated server-side today — an arbitrary
or empty string is accepted at submit time. So picking a correct category (and a
clear name + description) is a **quality convention that gets the policy found
and reviewed well, not a hard gate**. Treat it as required for a good
submission even though the server won't bounce a bad one.

## Repo-shape + pdk checks (defer to p4a-build-policy)

These are hard gates enforced at submit time; the rules and accepted forms live
in [[p4a-build-policy]]. Confirm before submitting:

- [ ] Repo is **public** on GitHub.
- [ ] Required files present at the policy root or subPath — unified:
      `Makefile`, `Cargo.toml`, `src/lib.rs`; split: definition root
      (`gcl.yaml`, `exchange.json`, `Makefile`) + implementation root
      (`Cargo.toml`, `Makefile`, `src/lib.rs`).
- [ ] `Cargo.toml` declares `pdk` resolving to **≥ 1.8.0** (inline or workspace
      inheritance).

## Pre-submit checklist

Run top to bottom; every box must be checked before submitting.

- [ ] `name` is 2–120 chars.
- [ ] `description` is 10–20,000 chars and reads as consumer-facing doc.
- [ ] `githubUrl` is a valid URL to a **public** repo.
- [ ] For a split project: `projectType: split` **and** `implementationGithubUrl` set.
- [ ] `category` is one of the 12 storable values (quality convention).
- [ ] Optional URLs (`videoTutorialUrl`, `examplesUrl`, `iconUrl`) are valid URLs or omitted.
- [ ] `deliversIdeaId`, if set, points at an **approved** idea.
- [ ] Repo-shape + `pdk ≥ 1.8` checks above pass (per [[p4a-build-policy]]).

## Diagnosing a failed submission

| Rejection signal | Failed check |
| --- | --- |
| "Repository must be public" / "not found or not public" | Repo isn't public, or URL/ref/subPath is wrong. |
| Missing-file error (Makefile / Cargo.toml / src/lib.rs, or split-root files) | Required file absent at the resolved path — recheck the subPath. |
| "Cannot parse pdk version" / version-too-low | `pdk` declaration missing, unparseable, or below 1.8.0; workspace inheritance not resolving (see [[p4a-build-policy]] caveat). |
| Field length / URL validation error | `name`, `description`, or a URL field is out of range or malformed. |
| Idea-not-approved error | `deliversIdeaId` references an idea that isn't in the approved state. |

A bad or missing `category` will **not** appear here — it's accepted silently
(see the caveat above), so it can't be diagnosed as a rejection; fix it for
quality regardless.

## Source Ref

Derived from the public P4A documentation:

- <https://docs.p4a.ai/docs/guides/submitting-a-policy> — submission requirements, required files, `pdk` ≥ 1.8, public-repo rule, categories.

_Snapshot: 2026-07-02._
