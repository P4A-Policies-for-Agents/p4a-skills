---
name: p4a-mcp-usage
description: Use when driving the P4A (Policies 4 Agents) MCP server's tools from an agent — discovering, deploying, and managing policies, connections, workspaces, and ideas. Covers which api:read / api:write scope gates which tools, how to chain the read tools into a deploy, defaulting deployments to the personal workspace, and what to gate for human approval — confirming inputs before any submit/deploy, and the destructive-tool confirmation flow (deploy_policy, delete_deployment, delete_failed_deployments, delete_connection, remove_workspace_member) including the confirm:true bypass and the no-elicitation fallthrough. Assumes the server is already connected and authenticated — MCP server / client configuration is out of scope (see the P4A docs). Do NOT use for writing PDK/Rust policy code (see the omni-gateway-pdk-skills repo), for assembling a submittable policy repo (see p4a-build-policy), or for the pre-submit self-check (see p4a-verify-requirements).
---

# P4A MCP Usage

## Overview

The P4A MCP server lets an agent search the policy catalog, inspect and deploy
policies, follow a deployment's build, manage Anypoint connections and
workspaces, and file or vote on ideas. This skill is about **using and chaining
those tools well, and knowing what to gate for human approval** — not about
standing up the connection.

**Out of scope: MCP server / client configuration** — the endpoint, transport,
session model, Personal Access Token minting, and client config. Assume the
server is already connected and authenticated. For setup, see the public P4A docs
at <https://docs.p4a.ai/docs/mcp> (`connecting`) and
<https://docs.p4a.ai/docs/api/personal-access-tokens>.

## When to Use

- Searching the catalog, inspecting a policy, or getting its install command.
- Deploying a policy and following the build to completion via logs.
- Managing Anypoint connections, workspaces + members, or ideas from the agent.

**Do NOT use** for:

- MCP/PAT setup — that's configuration; see the docs linked above.
- Writing PDK/Rust policy code — see `omni-gateway-pdk-skills`.
- Shaping a GitHub repo for submission — see [[p4a-build-policy]].
- The final pre-submit requirements check — see [[p4a-verify-requirements]].

## Scopes gate which tools you can call

Every tool runs **RLS-scoped to your token's user** — you only see and mutate
your own (or your workspace's) resources — and each tool needs one of two scopes.
Each tool's own description (surfaced at connect time) is the source of truth for
its purpose and parameters; what this skill adds is the scope + destructive
grouping, since that drives *how* you call them.

- **`api:read`** — every `search_*` / `get_*` / `list_*` tool (catalog,
  deployments, connections, submissions, workspaces, ideas) plus the docs tools
  (`search_docs`, `get_doc`).
- **`api:write`** — the mutating tools: `submit_policy`, `submit_idea`,
  `vote_idea`, `deploy_policy`, `rename_connection`, `delete_connection`,
  `create_workspace`, `rename_workspace`, `add_workspace_member`,
  `update_workspace_member`, `remove_workspace_member`, `delete_deployment`,
  `delete_failed_deployments`.

A tool called without its scope **fails closed** — the server rejects it rather
than silently no-op'ing. If a write tool returns a scope error, the token is
`api:read`-only and needs re-minting (a setup step; see the docs).

## Chaining tools — discover → deploy

The read tools resolve the ids the write tools need. Typical chain:

1. `search_policies` (or `list_ideas`) to find what you want.
2. `get_policy` to inspect it; `get_install_command` if you'd rather run the
   manual steps instead of deploying through the server.
3. `list_my_connections` → pick a `connectionId`; then **always call
   `list_connection_business_groups` and have the human confirm which business
   groups are the deploy targets** — present the reachable groups by name and let
   them choose the `targetOrganizationIds`. Never infer the targets or default to
   "all reachable"; a deploy lands in exactly the orgs chosen here, so this is a
   required approval step, not a convenience listing.
4. `list_my_workspaces` → pick a `workspaceId`. **Default to the personal
   workspace** when the chosen connection is reachable in more than one workspace,
   unless the human explicitly names a shared one. A shared-workspace deployment
   is visible to that workspace's other members, so don't pick one implicitly just
   because the connection happens to be reachable there.
5. `deploy_policy` — gates on confirmation (see below).
6. `get_deployment` / `get_deployment_logs` to follow the build to completion
   (paginate logs with `afterId`).

## What to gate for human approval

### Confirm inputs before any submit or deploy

**Restate the exact inputs and get the human's explicit go-ahead before calling
`submit_policy`, `submit_idea`, `vote_idea`, or any destructive tool.** These
create or change records other people see — a submission enters the review queue,
an idea is filed publicly, a deployment reaches Anypoint — and the agent is
usually assembling the fields from context, so a wrong `githubUrl`, `category`,
target business group, or workspace should be caught *before* the call, not walked
back after. For `deploy_policy` the highest-stakes fields are the target business
groups (`targetOrganizationIds`, resolved via `list_connection_business_groups` in
step 3) and the workspace — restate both by name.

1. Restate the full payload (every field; spell resolved ids out as their
   human-readable names where you can).
2. Wait for the human to confirm or correct it.
3. Only then call the tool.

This is the **agent's** responsibility even for the non-destructive filing tools
(`submit_policy` / `submit_idea` never prompt on their own). For the destructive
tools, this same restate-and-confirm step is how you decide whether to pass
`confirm: true`.

### Destructive-tool confirmation flow

Five tools are destructive or outward-facing: `deploy_policy`,
`delete_deployment`, `delete_failed_deployments`, `delete_connection`,
`remove_workspace_member`. The server decides how to confirm, in order:

1. **`confirm: true` passed** → proceeds, no prompt. Use only when you've *already*
   confirmed with the human (or in headless/scripted clients that can't prompt).
2. **Client has no `elicitation` capability** → proceeds under `api:write` +
   audit. No interactive channel exists, so the scope grant + immutable audit
   trail are the boundary.
3. **Client has `elicitation`** → the server prompts the human: accept → proceed;
   decline → `{ cancelled: true }`, no action; prompt fails/times out → explicit
   error telling you to re-call with `confirm: true` (never silently proceeds).

**Guidance:** prefer letting the human confirm — via the client's elicitation UI,
or by asking in your own turn and re-calling with `confirm: true`. Don't
blanket-set `confirm: true` to skip the human; the confirmation exists because
these actions reach Anypoint or delete data. Elicitation is a UX affordance *on
top of* the scope + audit boundary, not the boundary itself.

## Source Ref

Derived from the public P4A documentation:

- <https://docs.p4a.ai/docs/mcp/tools> — full tool list, scopes, destructive-tool confirmation.
- <https://docs.p4a.ai/docs/mcp/overview> — server model (endpoint, transport, sessions, elicitation).
- <https://docs.p4a.ai/docs/api/personal-access-tokens> — PAT scopes (`api:read` / `api:write`).

_Snapshot: 2026-07-02._
