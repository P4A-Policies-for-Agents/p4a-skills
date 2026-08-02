---
name: p4a-document-policy
description: "Use when writing, filling in, or revising the per-policy documentation TABS a P4A (Policies 4 Agents) policy shows on its catalog page — via the set_policy_doc_tab / replace_policy_docs / get_policy_docs MCP tools. Covers the four standard tabs every policy should fill (overview, configuration, examples, faq), their fixed keys/titles/sortOrder, when a custom tab (security, deployment, …) is worth adding, the See Also convention for linking related P4A and official MuleSoft policies, and the two mechanical traps that silently corrupt a tab set — omitting bodyMd on an upsert (wipes the body) and sortOrder collisions. Do NOT use for driving the deploy/submit tools (see p4a-mcp-usage), for shaping the GitHub repo (see p4a-build-policy), or for the pre-submit metadata check (see p4a-verify-requirements)."
---

# P4A Document Policy

## Overview

A P4A policy's catalog page renders its documentation as an ordered set of
**tabs**, managed through the P4A MCP doc tools. Left ungoverned, every author
invents a different tab set with different titles and ordering, so the catalog
reads inconsistently and consumers can't predict where to find configuration or
examples.

This skill is the **uniform contract** for those tabs: which tabs to fill, what
each one holds, the fixed order, and the two mechanical traps that corrupt a tab
set. Follow it so every P4A policy page has the same shape.

**Core principle:** the four standard tabs are a fixed contract — same keys,
same titles, same order, on every policy. Deviate only by *adding* a custom tab,
never by renaming or reordering the standard four.

## When to Use

- Filling in a new policy's documentation tabs for the first time.
- Revising or extending an existing policy's tabs.
- Deciding whether a piece of content is a standard tab, a custom tab, or a
  cross-link.

**Do NOT use** for:

- Calling the deploy/submit tools — see [[p4a-mcp-usage]].
- Shaping the GitHub repo layout — see [[p4a-build-policy]].
- The pre-submit metadata check (name/description/category) — see
  [[p4a-verify-requirements]].

## The tool set

| Tool | Use |
| --- | --- |
| `mcp__p4a__get_policy_docs` | Read the current tab set (`{key, title, kind, bodyMd, sortOrder}` each, plus a completeness metric). **Always call first** to see what exists. |
| `mcp__p4a__set_policy_doc_tab` | Upsert **one** tab. Keys `overview`/`configuration`/`examples`/`faq` are predefined; any other key is custom. Non-destructive. |
| `mcp__p4a__replace_policy_docs` | Bulk-replace the **whole** set at once. Must include an `overview` tab. Use only for a full rebuild. |

## The four standard tabs — fixed contract

Every policy fills **all four**. Keys, titles, and sortOrder are fixed — do not
rename or reorder them:

| sortOrder | key | title | Holds |
| --- | --- | --- | --- |
| 0 | `overview` | `Overview` | What the policy does, its category/positioning, the one key caveat, empty-config behavior. The elevator pitch. |
| 1 | `configuration` | `Configuration` | Every config property: a top-level table (name, type, default, description), then a sub-table per nested object/array. Failure behavior. |
| 2 | `examples` | `Examples` | Worked, copy-pasteable config snippets — one per distinct capability, each with a one-line intent header. |
| 3 | `faq` | `FAQ & See Also` | Common questions (see below), then the **See Also** section linking related policies and specs. |

Fill each `bodyMd` in your own words from the policy's own source (its
`gcl.yaml` schema + README), **restructured** into these tabs — do not paste the
README verbatim into one tab. The `overview` tab often mirrors the policy's
catalog description; the other three are derived.

## Custom tabs — add, don't multiply

A custom tab (any non-predefined key) is worth adding **only** when consumer-facing
content doesn't fit the four and pulls real weight. Common, reusable ones:

| key | title | When |
| --- | --- | --- |
| `security` | `Security` | Security-category policies, or any policy with a trust-model / "this is not a control boundary" caveat worth its own page. |
| `deployment` | `Deployment & Ordering` | Policy chain-order matters (e.g. must run after an auth policy), or has non-obvious apply-to / rollout guidance. |

Place custom tabs **after** the standard ones (sortOrder ≥ 3, pushing `faq`
last). Prefer folding content into a standard tab over inventing a niche custom
tab — resist one-off tabs like `how-it-works`, `upstream-bindings`,
`a2a-protocol`; that content belongs in `overview`/`configuration`. Skip
contributor-only material (architecture internals, build/test) — it lives in the
repo, not the catalog page.

## The FAQ & See Also tab

**FAQ:** 4–8 Q&A covering the questions the config table can't answer on its own
— the central caveat, where identity/state comes from, empty-config behavior,
protocol/version limits, ordering constraints. Cross-link the custom tabs
("see the **Security** tab").

**See Also:** group the links so the page situates itself in the ecosystem:

- **Official spec** for the protocol the policy governs (A2A, MCP, …).
- **MuleSoft Omni/Flex Gateway built-in policies** that relate. Don't hand-type
  the doc slug — look the policy up in the authoritative catalog:
  `mcp__p4a__search_policies` with `source: "mulesoft"`, then
  `mcp__p4a__get_mulesoft_policy` for its `doc_url` and `summary` (so the
  description you write matches what the built-in actually does, and the A2A/MCP
  protocol version it targets).
- **Related P4A policies** — find them with `mcp__p4a__search_policies` (by
  category or keyword). **Only link a policy that is approved AND published**:
  `search_policies` returns `lifecycle_status` and `published_at` on each hit —
  include it only when `lifecycle_status` is `approved` **and** `published_at` is
  non-null. Skip anything `under_review`, `draft`, or unpublished; a catalog page
  must not point consumers at a policy they can't yet install. For each kept
  policy, **link its P4A portal page** (where a consumer installs it), not its
  GitHub repo: `https://www.p4a.ai/dashboard/policies/<id>`, where `<id>` is the
  policy `id` from `search_policies`. Name it and say how it composes.
- **Framework** — the PDK doc links.

## Mechanical traps — these silently corrupt the tab set

| Trap | What happens | Do this instead |
| --- | --- | --- |
| **Omitting `bodyMd` on `set_policy_doc_tab`** | `bodyMd` defaults to **empty string** — a "just reorder" call with title + sortOrder only **wipes the body**. | Always pass the full `bodyMd` on every upsert, even when your intent is only to change `sortOrder` or `title`. |
| **`sortOrder` collision** | Two tabs at the same sortOrder render in an undefined order. | Keep sortOrders unique and contiguous: 0/1/2 for the standard three, custom tabs next, `faq` last. |
| **Retitling / rekeying the standard four** | Breaks catalog uniformity; a predefined key with a custom title reads inconsistently. | Use the exact keys and titles in the contract table. Add capability via custom tabs, not by mutating the four. |
| **`replace_policy_docs` without `overview`** | Rejected — `overview` is the one mandatory tab. | Include `overview` in any bulk replace; prefer per-tab `set_policy_doc_tab` unless doing a full rebuild. |

## Workflow

1. `get_policy_docs` — see what exists and its completeness.
2. Gather source: the policy's `gcl.yaml` (config truth) + README (prose).
3. Fill the four standard tabs with `set_policy_doc_tab`, sortOrder 0–3, each
   with a complete `bodyMd`.
4. Add a `security` / `deployment` custom tab only if it pulls weight; bump
   `faq` to last so it stays the final tab.
5. Build the See Also section — `search_policies` for related P4A policies,
   keep only the approved-AND-published ones, and link each one's portal page
   (`https://www.p4a.ai/dashboard/policies/<id>`). Verify any MuleSoft/spec URL
   before linking.
6. `get_policy_docs` again — confirm completeness is full and sortOrders are
   unique and contiguous.

## Common Mistakes

- **Dumping the README into one tab.** Restructure into the four; the README is
  source, not the page.
- **Inventing a bespoke tab per topic.** Every extra tab is a uniformity cost —
  fold into a standard tab unless it's a reusable `security`/`deployment` slice.
- **Reordering with a bodyless upsert.** Wipes the body (see traps). Carry the
  full body every time.
- **Linking a MuleSoft/spec URL from memory.** Verify it resolves first — built-in
  policy doc slugs are easy to guess wrong.
- **Linking an unpublished P4A policy, or naming one with no link.** A See Also
  entry must be approved AND published (`lifecycle_status: approved` +
  non-null `published_at`) and link its portal page
  (`https://www.p4a.ai/dashboard/policies/<id>`). Naming an `under_review` policy
  points consumers at something they can't install; a GitHub link or a bare name
  isn't the install surface.
