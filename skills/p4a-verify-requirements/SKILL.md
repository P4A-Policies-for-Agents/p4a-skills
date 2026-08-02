---
name: p4a-verify-requirements
description: "Use when self-checking a P4A (Policies 4 Agents) policy submission against the platform's requirements BEFORE calling submit_policy or the submit form, or when writing the submission description field, so the first attempt passes and reads as a real doc page instead of bouncing or shipping empty. Covers the submission metadata rules (name 2-120 chars, description 10-20000 chars, valid githubUrl — plus implementationGithubUrl for split projects, category from the 12 storable values, and the optional videoTutorialUrl / examplesUrl / iconUrl / iconDisabled / deliversIdeaId / projectType fields), how to write the description (it IS the catalog + Exchange doc page — a recipe: lead sentence, key caveat, config summary, worked example, positioning), the caveat that category is NOT enum-validated server-side (a quality convention, not a hard gate), the repo-shape + pdk>=1.8 checks it defers to p4a-build-policy, and a runnable pre-submit checklist plus a failed-submission diagnosis map. Run this as the last step before submitting. Do NOT use for assembling the repo layout in the first place (see p4a-build-policy), for driving the submit_policy tool (see p4a-mcp-usage), or for writing policy Rust code (see the omni-gateway-pdk-skills repo)."
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
| `description` | **Required.** 10–20,000 chars of Markdown. This *is* the policy's doc page — P4A renders it as the catalog detail page and generates the Anypoint Exchange asset doc page from it on deploy (there is no repo `home.md` ingestion). Write it for the consumer; see [Writing the description](#writing-the-description) below. |
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

## Writing the description

The `description` is the policy's **doc page**, not a blurb — it's what a
consumer reads to decide whether to install and how to configure. Length passes
validation at 10 chars; a *good* submission fills the field with a real page.
Because it renders as Markdown on both the catalog and (on deploy) the Exchange
asset page, write structured Markdown, not one paragraph.

If the policy already has **catalog documentation tabs** (see
[[p4a-document-policy]]), the description and the `overview` tab cover the same
ground — write the overview well and mirror its body (plus a config summary)
into `description` rather than authoring twice.

**Recipe — a good description contains, in this order:**

1. **Lead sentence** — what the policy does and for whom, in one line. Name the
   gateway (Omni/Flex), the surface it governs (A2A / MCP / LLM / HTTP), and the
   action (detect, block, transform, rate-limit…). This line is also what a
   reader skims first, so it must stand alone.
2. **The one key caveat** — the trust-model or scope limit a consumer must know
   before relying on it (e.g. "detection, not redaction"; "fails open on
   unknown versions"; "disclosure ≠ authorization"). State it early, not buried.
3. **Configuration summary** — the properties, their defaults, and empty-config
   behavior. A short table or list; enough to configure without leaving the
   page. Don't reproduce the full schema — summarize and point to the repo/tabs.
4. **A minimal worked example** — one copy-pasteable config snippet with a
   one-line intent. Fenced code block.
5. **Positioning** — how it composes with other policies and where it sits in
   the request lifecycle (ordering constraints, what to pair it with).

**Shape rules:**

- Open with the lead sentence — the catalog may truncate a preview, so the
  first line has to carry the value.
- Use headings, tables, and fenced code blocks; the renderer honors them. A
  wall of prose reads as unfinished.
- Keep it accurate to the shipped version — a description claiming a capability
  the code doesn't have is worse than omitting it.

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
- [ ] `description` is 10–20,000 chars and reads as a consumer-facing doc page —
      lead sentence, key caveat, config summary, a worked example, positioning
      (see [Writing the description](#writing-the-description)).
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

_Snapshot: 2026-08-02 (added description-writing recipe)._
