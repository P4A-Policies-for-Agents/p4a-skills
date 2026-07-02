---
name: p4a-mcp-usage
description: Use when connecting an AI agent to the P4A (Policies 4 Agents) platform over the P4A MCP server, or when driving its tools to discover, inspect, install, or deploy a policy. Covers connecting the Streamable-HTTP MCP endpoint, authenticating with a Personal Access Token (PAT) and the api:read / api:write scope model, and the catalog/status/deploy tool set — search_policies, get_policy, get_install_command, list_deployments, get_deployment, get_deployment_logs, list_my_connections, deploy_policy — including the elicitation confirmation on destructive tools. Do NOT use for writing PDK/Rust policy code (see the omni-gateway-pdk-skills repo) or for assembling a submittable policy repo (see p4a-build-policy).
---

# P4A MCP Usage

## Overview

<!-- Sub-issue: flesh out. The P4A MCP server is a hosted, stateful Streamable-HTTP
     endpoint. An agent authenticates with a Personal Access Token (bearer), and
     every tool runs RLS-scoped to that token's user — never service-role. -->

## When to Use

- Connecting an agent to the P4A MCP server and confirming the handshake.
- Searching the policy catalog, inspecting a policy, or getting its install command.
- Checking deployment status / logs, or deploying a policy to a connected Anypoint org.

**Do NOT use** for:

- Writing PDK/Rust policy code — see `omni-gateway-pdk-skills`.
- Shaping a GitHub repo for submission — see [[p4a-build-policy]].

## Connecting

<!-- Sub-issue: MCP endpoint URL, transport (Streamable-HTTP), client config example. -->

## Authentication — PAT + scopes

<!-- Sub-issue: how to mint a PAT in the portal, the api:read vs api:write scopes,
     how to attach the bearer, what fails closed when a scope is missing. -->

## Tools

<!-- Sub-issue: per-tool reference. Read-only (api:read): search_policies, get_policy,
     get_install_command, list_deployments, get_deployment, get_deployment_logs,
     list_my_connections, get_connection, search_docs, get_doc. Write (api:write):
     deploy_policy, submit_policy, submit_idea. -->

## Confirming a destructive action (elicitation)

<!-- Sub-issue: deploy_policy / delete_* confirm via MCP elicitation when the client
     advertises the capability; otherwise proceed under api:write + audit. -->

## Source Ref

<!-- Sub-issue: link the public P4A docs MCP page + OpenAPI, with snapshot date. -->
