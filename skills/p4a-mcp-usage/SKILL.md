---
name: p4a-mcp-usage
description: Use when connecting an AI agent to the P4A (Policies 4 Agents) platform over its MCP server, or when driving that server's tools to discover, deploy, or manage policies, connections, workspaces, and ideas from an agent. Covers connecting the Streamable-HTTP MCP endpoint, authenticating with a Personal Access Token (PAT) bearer, the api:read / api:write scope split, which tools each scope gates, the destructive-tool confirmation flow (deploy_policy, delete_deployment, delete_failed_deployments, delete_connection, remove_workspace_member) including the confirm:true bypass and no-elicitation fallthrough, confirming inputs with the human before any submit/deploy, and defaulting deployments to the personal workspace. Do NOT use for writing PDK/Rust policy code (see the omni-gateway-pdk-skills repo), for assembling a submittable policy repo (see p4a-build-policy), or for the pre-submit self-check (see p4a-verify-requirements).
---

# P4A MCP Usage

## Overview

The P4A platform exposes its capabilities to AI agents through a hosted **MCP
(Model Context Protocol) server**. Once your agent is connected and
authenticated with a Personal Access Token, it can search the policy catalog,
inspect a policy, deploy one to your Anypoint org, watch a deployment's build
logs, manage connections and workspaces, and file or vote on ideas — all without
leaving the agent.

Everything the server does runs **scoped to your account** (row-level security),
using the identity behind your token. The token's scopes (`api:read` /
`api:write`) decide which tools you can call; a small number of outward-facing or
destructive tools additionally ask for confirmation before they act.

> **Canonical reference:** the public P4A docs at
> <https://docs.p4a.ai/docs/mcp> — `overview` (endpoint, transport, session
> model), `connecting` (token + client setup), and `tools` (full tool list and
> scopes). This skill distills those pages into an agent-facing playbook; when
> they disagree, the docs win.

## When to Use

- Connecting an agent to the P4A MCP server and confirming the handshake.
- Searching the catalog, inspecting a policy, or getting its install command.
- Deploying a policy and following the build to completion via logs.
- Managing Anypoint connections, workspaces + members, or ideas from the agent.

**Do NOT use** for:

- Writing PDK/Rust policy code — see `omni-gateway-pdk-skills`.
- Shaping a GitHub repo for submission — see [[p4a-build-policy]].
- The final pre-submit requirements check — see [[p4a-verify-requirements]].

## Connecting

The MCP endpoint is your portal origin + `/mcp` — in production:

```
https://www.p4a.ai/mcp
```

Transport is **Streamable HTTP** (MCP spec): the agent POSTs JSON-RPC requests,
and the server can open a server→client SSE stream (used for progress
notifications and the confirmation prompts described below). The session is
**stateful** — the server mints an `Mcp-Session-Id` on `initialize` and pins it
to your token's user; every subsequent request must carry the same bearer. The
server runs single-replica, so a session lives on one process; if the connection
drops, re-`initialize`.

During `initialize`, advertise the **`elicitation`** client capability if your
client can render a confirmation prompt — that unlocks the interactive
confirm-before-destroy UX below. Clients that can't (e.g. some headless setups)
still work; see [Confirming a destructive action](#confirming-a-destructive-action).

Example MCP client entry (shape varies by client — this is the Claude Code
`mcpServers` form):

```json
{
  "mcpServers": {
    "p4a": {
      "url": "https://www.p4a.ai/mcp",
      "headers": { "Authorization": "Bearer p4a_your_token_here" }
    }
  }
}
```

## Authentication — PAT + scopes

The server authenticates every request with a **Personal Access Token (PAT)**
passed as a bearer:

```
Authorization: Bearer p4a_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

- **Format:** tokens are prefixed `p4a_`. The full token is shown **once** at
  creation and never again — only a hash is stored, so copy it immediately.
- **Where to mint one:** the portal **Settings** page → *Personal Access Tokens*.
  See <https://docs.p4a.ai/docs/api/personal-access-tokens#creating-a-token>.
- **Same token, every surface:** the same PAT authenticates the MCP server, the
  REST API, and CI scripts. Scopes are transport-neutral.

### Scopes

Pick the narrowest scope the task needs when you mint the token:

| Scope | Grants | Use when |
| --- | --- | --- |
| `api:read` | All read/discovery tools (catalog, deployments, connections, submissions, workspaces, ideas) + the docs tools | The agent only inspects / discovers. |
| `api:write` | Everything in `api:read`, **plus** all mutating tools (submit, deploy, connection + workspace management, idea voting) | The agent submits, deploys, or manages resources. |

A tool called without the required scope **fails closed** — the server rejects it
rather than silently no-op'ing. If a write tool returns a scope error, the token
was minted `api:read`-only; mint a new `api:write` token.

## Tools

Each tool's own description (surfaced to your client at connect time) is the
source of truth for its purpose and parameters — don't rely on this skill to
re-list them. What matters here is the **scope each tool needs** and **which ones
are destructive**, since those drive how you call them.

All tools are **RLS-scoped to your token's user** — you only see and mutate your
own (or your workspace's) resources.

- **`api:read`** — every `search_*` / `get_*` / `list_*` tool (catalog,
  deployments, connections, submissions, workspaces, ideas). The docs tools
  (`search_docs`, `get_doc`) read public docs and need only a valid token.
- **`api:write`** — the mutating tools: `submit_policy`, `submit_idea`,
  `vote_idea`, `deploy_policy`, `rename_connection`, `delete_connection`,
  `create_workspace`, `rename_workspace`, `add_workspace_member`,
  `update_workspace_member`, `remove_workspace_member`, `delete_deployment`,
  `delete_failed_deployments`.
- **Destructive / outward-facing** (gate on confirmation — see below):
  `deploy_policy`, `delete_deployment`, `delete_failed_deployments`,
  `delete_connection`, `remove_workspace_member`.

### Confirm inputs before submitting

**Always show the human the exact inputs and get their explicit go-ahead before
calling any submission or filing tool** — `submit_policy`, `submit_idea`,
`vote_idea` — and before the destructive/outward-facing tools covered in
[Confirming a destructive action](#confirming-a-destructive-action)
(`deploy_policy`, `delete_*`, `remove_workspace_member`).

These tools create or change records other people see (a submission enters the
review queue; an idea is filed publicly; a deployment reaches Anypoint), and the
agent is often assembling the fields from context — a wrong `githubUrl`,
`category`, or `targetOrganizationIds` should be caught *before* the call, not
walked back after. So:

1. Restate the full payload you're about to send (every field, resolved ids
   spelled out as their human-readable names where you can).
2. Wait for the human to confirm or correct it.
3. Only then call the tool.

This applies even when the tool itself wouldn't otherwise prompt (e.g.
`submit_policy` / `submit_idea` are non-destructive and never elicit) — the
confirmation is the agent's responsibility, not the server's. For the tools that
*do* elicit, this restate-and-confirm step is also how you decide whether to pass
`confirm: true`.

### A typical discover → deploy flow

1. `search_policies` (or `list_ideas`) to find what you want.
2. `get_policy` to inspect it; `get_install_command` if you'd rather run the
   manual steps.
3. `list_my_connections` → pick a `connectionId`; `list_connection_business_groups`
   → pick the `targetOrganizationIds`.
4. `list_my_workspaces` → pick a `workspaceId`. **Default to the personal
   workspace** when the chosen connection is available in more than one workspace,
   unless the human explicitly names a shared workspace. Deploying into a shared
   workspace exposes the deployment to that workspace's other members, so don't
   pick one implicitly just because the connection happens to be reachable there —
   confirm the target workspace as part of the input-confirmation step above.
5. `deploy_policy` with those ids (confirms — see below).
6. `get_deployment` / `get_deployment_logs` to follow the build to completion.

## Confirming a destructive action

Five tools are **destructive or outward-facing** and gate on confirmation:
`deploy_policy`, `delete_deployment`, `delete_failed_deployments`,
`delete_connection`, `remove_workspace_member`.

The server decides how to confirm, in this order:

1. **`confirm: true` was passed** → proceed immediately, no prompt. Use this when
   *you* (the agent) have already confirmed with the human, or in scripted /
   headless clients that can't render a prompt.
2. **Client didn't advertise `elicitation`** → proceed under `api:write` + audit.
   There's no interactive channel, so the scope grant + the immutable audit trail
   are the security boundary.
3. **Client advertised `elicitation`** → the server sends a confirmation prompt
   over the SSE stream:
   - human accepts → the tool proceeds;
   - human declines → the tool returns `{ cancelled: true }` and does nothing;
   - the prompt fails or times out → the server returns an explicit error telling
     you to re-call with `confirm: true` (it never silently proceeds on a failed
     prompt).

**Guidance for agents:** prefer letting the human confirm — either through your
client's elicitation UI, or by asking in your own turn and then re-calling with
`confirm: true`. Don't blanket-set `confirm: true` on every destructive call to
skip the human; the confirmation exists because these actions reach Anypoint or
delete data. Elicitation is a **UX affordance on top of** the scope + audit
boundary, not the boundary itself.

## Source Ref

Derived from the public P4A documentation:

- <https://docs.p4a.ai/docs/mcp/overview> — endpoint, Streamable-HTTP transport, stateful session model, elicitation capability.
- <https://docs.p4a.ai/docs/mcp/connecting> — PAT bearer auth, client setup, minting a token.
- <https://docs.p4a.ai/docs/mcp/tools> — full tool list, scopes, destructive-tool confirmation.
- <https://docs.p4a.ai/docs/api/personal-access-tokens> — PAT format, scopes (`api:read` / `api:write`), creating and revoking tokens.

_Snapshot: 2026-07-02._
