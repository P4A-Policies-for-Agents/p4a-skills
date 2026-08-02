---
name: p4a-test-mcp-policies-with-a2d
description: Use when testing an MCP gateway policy end-to-end against a realistic server — building a mock MCP server in A2D (Agent-to-Discovery), publishing its manifest to Anypoint Exchange as a type=mcp asset, fronting it with a Flex/Omni Gateway API instance, applying the policy under test, and diffing the governed tools/list against the ungoverned mock to prove the policy fired. Covers satisfying A2D's "Issues Found" quality gate up front (required provider org+URL, tool description ≥200 chars, property description ≥25 chars, property name ≥5 chars); the confirm-transport-first step and the fact that MCP has no v0.3/v1.0 split — the multipart -F type=mcp form field (not a properties.protocol) makes the asset governable, and the transport lives INSIDE the mcp-metadata.json manifest, NOT as a queryable Exchange attribute; the working publish method (A2D's publish is unreliable — hand-author mcp-metadata.json + raw multipart POST, verified live); the gateway instance (endpoint.type=mcp, rest rejected; technology=flexGateway; isCloudHub null; deploy via apimanager/xapi/v1 because api/v1 silently leaves deployment null on managed/Omni Flex GW); the MCP-Support-must-be-first-in-chain rule (it preserves SSE framing + Mcp-Session-Id downstream); and verifying over a JSON-RPC/SSE handshake (initialize → notifications/initialized → tools/list), NOT a static GET. Do NOT use for writing the policy's PDK/Rust code (see omni-gateway-pdk-skills), the PDK-local cargo test harness, testing an A2A Agent Card policy (see p4a-test-a2a-policies-with-a2d), or driving the P4A MCP server to deploy a catalog policy (see p4a-mcp-usage).
---

# P4A: Test MCP Policies with A2D

## Overview

An MCP gateway policy (e.g. a tool-list governor, an MCP access-control policy, a
tool-description rewriter) can only be *fully* proven against a real MCP server
speaking JSON-RPC over the wire. The PDK-local `cargo test` harness exercises the
policy in isolation; this skill covers the **integration loop** — a live mock MCP
server behind a real Flex/Omni Gateway instance:

```
A2D mock MCP  ──►  Exchange (type=mcp asset)  ──►  Flex GW API instance  ──►  policy applied
     │                                                    │
     └─ ungoverned tools/list (baseline)      governed tools/list (verify the diff)
```

The **proof** is the diff: fetch `tools/list` directly from the A2D mock
(ungoverned baseline), fetch it again through the gateway (policy applied), and
confirm the policy transformed it as configured.

**MCP has no protocol-version split** (unlike A2A's v0.3/v1.0). There is one
transport that matters here — **streamable-HTTP** — so there is no dual-publish
dance. The lever that makes the asset governable is **`type=mcp`**; the transport
is declared *inside* the `mcp-metadata.json` manifest (`transport.kind`), not as an
Exchange attribute you can query.

## When to Use

- Testing any MCP policy against a live MCP server over HTTP, not just unit tests.
- Demonstrating a policy's effect (tool hiding, description rewrite, tool-list
  filtering, access gating) with a real before/after `tools/list` diff.
- Standing up a disposable mock MCP server when you don't have a real one to point at.

**Do NOT use** for:

- Writing the policy's PDK/Rust code — see the `omni-gateway-pdk-skills` repo.
- The PDK-local `cargo test` / `make test` harness — that tests the policy in
  isolation with a stubbed request, not against a live server. Run it first; this
  skill is the layer above it.
- Testing an A2A **Agent Card** policy — see [[p4a-test-a2a-policies-with-a2d]]
  (the publish shape, gateway `endpoint.type`, and verification surface all differ).
- Deploying a catalog policy through the P4A MCP server — see [[p4a-mcp-usage]].
- Publishing a *non-mcp* asset (REST/agent/LLM) — the publish shape differs.

## Step 0 — Confirm the transport and surface FIRST

**Before building anything, ask the human what to test.** MCP has no v0.3/v1.0
version to pick, but three things still shape the whole loop:

- **Transport.** This skill covers **streamable-HTTP** (`transport.kind:
  streamableHttp` in the manifest; gateway `endpoint.type=mcp`). SSE-only servers
  behave differently at the wire; confirm the mock and the policy target
  streamable-HTTP.
- **Surface the policy acts on.** `tools/list` is the common target (tool hiding /
  rename / description rewrite). But a policy might instead gate `tools/call`,
  filter `resources/list`, or act on the `initialize` capabilities. Confirm which
  MCP method the policy transforms — that decides what you diff.
- **Inbound vs outbound.** A policy that transforms a *response* (rewriting the
  `tools/list` a server returns) is **outbound** and binds to the upstream
  (`--upstreamId`); one that gates a *request* is **inbound**. **Infer it from the
  policy's `gcl.yaml` `metadata/capabilities/injectionPoint`, then confirm the
  reading with the human** before applying (Step 4) — don't apply on the inferred
  value alone.

**Also ask the human for a connected-app `client_id` and `client_secret`.** Every
raw Exchange and API Manager REST call in this skill (publish, poll, delete,
instance create/PATCH, upstream fetch, policy apply) authenticates with a bearer
token minted from these credentials via `client_credentials` (Step 2). The
connected app must have the Exchange and API Manager scopes for the target
org/env. Do NOT hardcode or echo the secret — take it from the human, keep it in
an env var (e.g. `$CLIENT_ID` / `$CLIENT_SECRET`), and reuse the token for all
calls below.

## Step 1 — Build the mock MCP server in A2D

Use the A2D MCP tools (`design_mcp_server`, then `add_mcp_tool` per tool) to
design a mock server whose tool list gives the policy something to act on.

**Name the server `<Policy Name> Test Server`** (e.g. `Tool List Governor Test
Server`) so the mock is instantly traceable to the policy under test across A2D,
the Exchange catalog, and the gateway instance. Carry the same name through the
Exchange `assetId`/`name` (Step 2) and the `instanceLabel` (Step 3).

For a tool-list/visibility policy, the tool set should be a deliberate mix:

- a **public** tool that should stay (control),
- a tool targeted for **description/annotation rewrite** (verify the text changes),
- a tool targeted for **exact-name deny** (verify it disappears),
- a tool matching a **glob/pattern deny** (verify pattern matching).

Give each tool a real `inputSchema`/`outputSchema` and `mockScenarios` so
`tools/call` returns deterministic data (`===` / `exists` operators; append the
`exists` catch-all **last** so specific matches win).

> **Satisfy A2D's quality gate as you build — don't discover it at publish.** A2D
> runs a validator ("Issues Found") over the server + every tool, and a failing
> asset blocks the loop. Set these up front so the report comes back clean:
>
> - **Server provider info is required** — set both the provider **organization**
>   AND a provider **URL** on the server (missing either is a hard failure).
> - **Tool `description` ≥ 200 characters** — every tool, including trivial ones
>   like `health_ping`. Write a real sentence or two on what it does, its inputs,
>   and what it returns; a one-liner (≈120–170 chars) fails.
> - **Each tool property `description` ≥ 25 characters** — every field in the
>   `inputSchema`/`outputSchema` needs a substantive description, not a stub like
>   `"the topic"` (12 chars).
> - **Each tool property `name` ≥ 5 characters** — rename short fields (`text` →
>   `documentText`, `id` → `recordId`) so no property name is under 5 chars.
>
> These same minimums apply to the hand-authored `mcp-metadata.json` in Step 2 —
> the manifest is derived from the mock, so fixing them here fixes both.

Confirm the live mock serves them all with the JSON-RPC handshake (Step 5 shows
the full recipe): `initialize` → `notifications/initialized` → `tools/list`
should list every tool. This is your **ungoverned baseline** — save it.

The live mock endpoint is `POST https://a2d-ai.com/api/platform/<mock-id>/mcp`
(streamable-HTTP; responses are `text/event-stream`).

## Step 2 — Publish the MCP manifest to Exchange as `type=mcp`

> **Publish by hand — do not rely on A2D's `publish_to_exchange_mcp_server`.** As
> with the A2A card publisher, the A2D Exchange builder is unreliable for a
> governable asset (it serializes from the draft spec, which omits mock scenarios
> and can emit a manifest Exchange's mcp-domain validator rejects). The live mock
> endpoint serves tools correctly; the Exchange builder is the weak link.
> Hand-author the manifest and raw-POST it, exactly as [[p4a-test-a2a-policies-with-a2d]]
> does for the Agent Card.

1. **Hand-author the MCP manifest** (`mcp-metadata.json`). It carries
   `protocolVersion` (e.g. `2025-03-26`), `transport: {kind: "streamableHttp",
   path: "/mcp"}`, `capabilities`, and the `tools[]` array (each with
   `name`/`description`/`annotations`/`inputSchema`/`outputSchema`). This is the
   MCP analogue of the A2A card — it is the payload the policy will govern, and
   **the transport lives here** (`transport.kind`), not in an Exchange attribute.

2. **Author the Exchange descriptor.** `assetId`, `groupId`, `version`, `name`,
   `description`, `tags`, and the properties block. The `-F type=mcp` form field
   (Step 2.4) is what makes the asset governable and unlocks the `endpoint.type=mcp`
   gateway instance (Step 3) — there is **no `properties.protocol`** for MCP.
   `properties.platform` surfaces as a queryable Exchange attribute; other
   `properties.*` (e.g. a `transport` hint) may be silently dropped — **verified:**
   a published MCP asset shows only `{platform: …}` under `attributes[]`, so do not
   rely on a `transport` attribute existing (it lives in the manifest).

3. **Get a token** — `POST /accounts/api/v2/oauth2/token`, `grant_type=client_credentials`,
   form-encoded `client_id`/`client_secret`. Use the connected-app credentials the
   human supplied in Step 0 (`$CLIENT_ID` / `$CLIENT_SECRET`); mint once and reuse
   the bearer token for every raw REST call in Steps 2–4.

   ```bash
   TOKEN=$(curl -sS -X POST https://anypoint.mulesoft.com/accounts/api/v2/oauth2/token \
     -d grant_type=client_credentials \
     -d "client_id=$CLIENT_ID" -d "client_secret=$CLIENT_SECRET" | jq -r .access_token)
   ```

4. **Raw multipart POST.** Endpoint carries **org AND full GAV** in the path (the
   CLI's `exchange asset upload` has no `mcp` type — it won't work):

   ```
   POST /exchange/api/v2/organizations/{ORG}/assets/{groupId}/{assetId}/{version}
     -F type=mcp                               # ← the MCP lever (NOT a properties.protocol)
     -F name=...
     -F status=published
     -F description=...                        # keep concise
     -F properties.platform=a2d                # surfaces as an attributes[] entry
     -F "files.mcp-metadata.json=@mcp-metadata.json;type=application/json"
   ```

   The manifest rides as a file field named **`files.mcp-metadata.json`** (it lands
   under the `mcp-metadata` file classifier). The bare `/assets` and org-only routes
   404 or hit the wrong facade. **Verified end-to-end** against a live EU org:
   `type=mcp` + this file field → 202 → `completed`, asset published as `type: mcp`.

5. **Poll** — response is **202** + a `publicationStatusLink`. Poll it (~2s) until
   `status: completed`. `409` = already published (bump version to re-publish).
   Confirm the publish stuck:
   `GET /exchange/api/v2/assets/{ORG}/{assetId}/{version}` → the asset should show
   `type: mcp` and a `mcp-metadata` file classifier. (Its `attributes[]` will carry
   only `platform` — the transport is inside the manifest, not an attribute.)

6. **Add a `home.md` portal home page — via the v1 `portal/draft` API, NOT the
   v2 `pages` route.** A freshly published asset has a blank catalog page until a
   home page exists. Author a `home.md` describing the mock server (what it is,
   its tools + sensitivity, the transport/path, the raw + governed endpoints),
   then PUT it into the portal **draft** and publish the draft:

   ```
   # 1) write the home page into the editable draft (page name in the path)
   PUT /exchange/api/v1/organizations/{ORG}/assets/{ORG}/{assetId}/{version}/portal/draft/pages/home
     Authorization: Bearer $TOKEN
     Content-Type: text/markdown
     --data-binary @home.md                 # → 204

   # 2) publish the draft so the page goes live on the catalog card
   PATCH /exchange/api/v1/organizations/{ORG}/assets/{ORG}/{assetId}/{version}/portal
     Authorization: Bearer $TOKEN
     Content-Type: application/json
     -d '{}'                                 # → 204

   # 3) (optional) confirm it rendered
   GET /exchange/api/v1/organizations/{ORG}/assets/{ORG}/{assetId}/{version}/portal/pages/home
   ```

   > **The v2 `pages/home` route does NOT exist for `type=mcp` assets — it 404s,
   > and so does `GET .../{version}/pages`.** This is the A2A flow's route; it is
   > simply not wired for mcp. **Verified live:** every published mcp asset in a
   > real org (e.g. `sf-customer-asset-mcp`, `talent-pool-mcp-server`, `jiraMcp`)
   > has no v2 pages route and no `pages` field. The Anypoint UI edits the home
   > page through the **v1 `portal/draft`** surface above — a fresh asset starts
   > with one `synthetic: true` placeholder `home` page; the `PUT` replaces it
   > with real content (drops `synthetic`), and the `PATCH …/portal` publishes the
   > draft. Note the v1 path repeats the org twice (`organizations/{ORG}/assets/{ORG}/…`)
   > and takes the page name **in the path** (`.../pages/home`), not `pages/home`
   > as a slug. Re-run both steps after any hard-delete + republish (that drops
   > portal pages back to the synthetic placeholder).

7. **Hard-delete** a mistaken asset via `DELETE /exchange/api/v2/assets/{group}/{asset}/{version}`
   + header `X-Delete-Type: hard-delete` (→ 204). The CLI has no hard-delete flag.
   Hard-delete drops the asset's portal pages too — re-run the v1 `portal/draft`
   PUT + `portal` PATCH (step 6) after republishing.

> **The Exchange app artifact is NOT the mcp asset.** If your mock is a deployed
> Mule app, Exchange already holds an **app**-typed artifact
> (`<name>-mule-application`) — the packaged app, not a governable mcp asset. The
> gateway instance needs a **`type=mcp`** asset with the manifest. Publish it
> separately (this step); don't point the instance at the app artifact.

## Step 3 — Front the server with a Flex Gateway API instance

The `create_and_manage_api_instances` MCP tool only does `mule4`/`basicEndpoint`
— it **cannot** create a Flex `mcp` instance. The mock's public URL is a valid
**external upstream** — no deployed backing app required. The CLI's `--type` enum
has no `mcp` either, so create the instance via **raw API Manager REST** against
the `type=mcp` asset from Step 2.

### Create the instance — `endpoint.type=mcp`

```bash
POST /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis
{
  "technology": "flexGateway",                 # else the mcp endpoint is rejected
  "endpoint": {
    "type": "mcp",                             # ← MCP endpoint type ("rest" is rejected for mcp assets)
    "deploymentType": "HY",
    "isCloudHub": null,                        # MUST be null (not false) and MUST be present
    "proxyUri": "http://0.0.0.0:8081/<label>/",
    "uri": "<mock-mcp-base>/"                  # the prefix such that +"/mcp" = the real surface — see note
  },
  "spec": {"groupId":"<ORG>","assetId":"<assetId>","version":"<version>"},
  "instanceLabel": "<label>"
}
```

> **Upstream URI is the prefix onto which MCP Support appends `/mcp` — NOT
> necessarily the host root.** MCP Support maps the gateway route onto
> `<upstream-uri>mcp`, so the upstream must be exactly the server's MCP surface
> **minus** the trailing `mcp`, with a trailing `/`. Do NOT put `/mcp` on the
> upstream yourself (that double-paths to `…/mcp/mcp`).
> - Server whose MCP surface is at the host root (`https://host/mcp`) → upstream
>   is the **host root**: `https://host/`.
> - **A2D mock** — its surface is `https://www.a2d-ai.com/api/platform/<serverId>/mcp`,
>   so the upstream is the **platform prefix**
>   `https://www.a2d-ai.com/api/platform/<serverId>/` (host root alone 404s). The
>   rule is "surface minus `mcp`", not "host root". **Verified live:** with this
>   prefix the `initialize`/`tools/list`/`tools/call` handshake succeeds through the
>   gateway.

### Deploy via the `xapi/v1` PATCH surface (managed / Omni Flex GW)

On a managed or **Omni** Flex Gateway target, the `api/v1` create+PATCH accept a
`deployment{}` block, return 200, but **silently leave `deployment: null`** — the
instance never actually deploys. Deploy through the **`apimanager/xapi/v1`** PATCH
surface instead. First resolve the auto-created upstream id (xapi rejects a bare
weight — routing must reference the upstream by id):

```bash
# API Manager auto-creates one upstream on instance-create; get its id
GET /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis/{id}/upstreams
#   → .upstreams[0].id

PATCH /apimanager/xapi/v1/organizations/{ORG}/environments/{ENV_ID}/apis/{id}?checkAutomatedPolicies=true
{
  "technology": "flexGateway",
  "endpointUri": "https://<gw-public-host>/<label>/",
  "endpoint": { "deploymentType":"HY", "type":"mcp", "isCloudHub":null,
                "proxyUri":"http://0.0.0.0:8081/<label>/", "tlsContexts":{"inbound":null} },
  "upstreams": [ { "id":"<upid>", "uri":"<mock-mcp-base>/", "label":null, "tlsContext":null } ],
  "routing":   [ { "upstreams":[ { "id":"<upid>", "weight":100 } ] } ],
  "deployment": { "environmentId":"<ENV_ID>", "type":"HY", "expectedStatus":"deployed",
                  "overwrite":false, "targetId":"<GW_TARGET_ID>", "targetName":"<GW_NAME>",
                  "gatewayVersion":"1.0.0" }
}
```

Confirm the PATCH returned a **non-null `deployment`** (`expectedStatus: deployed`,
a real `applicationId`, and `gatewayVersion` resolved to the live version e.g.
`1.13.2`) — that's how you know the deploy took (vs the `api/v1` silent-null trap).
To **redeploy** later (e.g. after applying a policy), re-send the same PATCH with
`overwrite: true`.

**Live-probe constraints (managed Flex GW):** `scheme=http` (TLS terminates at the
gateway), listener `port=8081`, both proxy path and upstream URI must end with
`/`, `gatewayVersion 1.0.0` works even on a newer live gateway (the platform
resolves it to the real running version).

### Optional — register the raw mock as an Exchange *external* instance

Separate from the managed gateway instance above, you can attach the **raw mock
URL** directly to the Exchange asset as an **external** instance. This is an
**Exchange** object (not an API Manager instance): it registers the endpoint for
catalog discovery only — it does **not** front the mock with a gateway and does
**not** enforce any policy. Use it so the asset's catalog page shows the direct
mock endpoint alongside the governed gateway endpoint.

```
POST /exchange/api/v2/assets/{ORG}/{assetId}/versionGroups/{versionGroup}/instances/external
  Content-Type: application/json
  {"name":"A2D Mock","endpointUri":"<raw-mock-url>/mcp","isPublic":false}
```

`{versionGroup}` is the asset's version group (e.g. `v1.0`), **not** the version
(`1.0.0`). `endpointUri` is the raw A2D mock
(`https://a2d-ai.com/api/platform/<mock-id>/mcp`).

## Step 4 — Apply the policy under test

**MCP Support MUST be the first policy in the chain.** It preserves the SSE
framing and the `Mcp-Session-Id` header for every downstream policy — if any other
policy runs before it, the streamable-HTTP handshake breaks and the tools policy
never sees a well-formed session. Order: `mcp-support` (order 1) → your policy
under test → any logging/auth policies.

Apply MCP Support first (empty config), then the policy under test. Resolve the
policy GAV (`anypoint-cli-v4 exchange asset list <assetId>` → the `type=policy`
row, not `policy-implementation`):

```bash
# 1) MCP Support — order 1, so it runs before everything (config: {})
anypoint-cli-v4 api-mgr:policy:apply <instanceId> mcp-support \
  --environment "<ENV>" --groupId <ORG> --policyVersion <ver> -c '{}'

# 2) the policy under test
anypoint-cli-v4 api-mgr:policy:apply <instanceId> <policyAssetId> \
  --environment "<ENV>" --groupId <ORG> --policyVersion <ver> \
  -c "$(cat config.json)"
anypoint-cli-v4 api-mgr:api:redeploy <instanceId> --environment "<ENV>"
```

> **MCP Support GAV is in a MuleSoft group, not your org.** `mcp-support`'s
> `groupId` is a MuleSoft-owned group (e.g. `68ef9520-24e9-4cf2-b2f5-620025690913`,
> `v1.0.1`), **not** your `<ORG>` — resolve it with
> `search_policies`/`exchange asset list`, don't assume it shares your org's group.
> Your policy-under-test uses your `<ORG>` group.

**Raw-REST alternative (no CLI auth).** If the CLI isn't logged in, apply policies
with the same bearer token used everywhere else — `POST …/apis/{id}/policies` with
`order` controlling chain position (MCP Support `order:1`, policy-under-test
`order:2`). `configurationData` is the policy config object inline:

```bash
POST /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis/{id}/policies
  { "groupId":"68ef9520-24e9-4cf2-b2f5-620025690913", "assetId":"mcp-support",
    "assetVersion":"1.0.1", "configurationData":{}, "order":1 }              # → 201

POST /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis/{id}/policies
  { "groupId":"<ORG>", "assetId":"<policyAssetId>", "assetVersion":"<ver>",
    "configurationData": { …policy config… }, "order":2 }                    # → 201
```

Then redeploy via the **`xapi/v1` PATCH** (Step 3, `overwrite:true`) — the
`api-mgr:api:redeploy` CLI equivalent — so the new chain propagates.

**Inbound vs outbound — this bites.** A policy that transforms the *response*
(e.g. rewriting the `tools/list` a server returns) is **outbound**
(`injectionPoint: outbound` in its `gcl.yaml`). Applying it without an upstream
fails: `Error: This policy can not be applied as inbound`. Fix — bind it to the
upstream:

```bash
# fetch the upstream id
GET /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis/{id}/upstreams
# re-apply with --upstreamId (accepted only on :apply, not :edit)
anypoint-cli-v4 api-mgr:policy:apply <instanceId> <policyAssetId> \
  --environment "<ENV>" --groupId <ORG> --policyVersion <ver> \
  --upstreamId <upstreamId> -c "$(cat config.json)"
```

Read the policy's `gcl.yaml` `metadata/capabilities/injectionPoint` to infer the
binding, **then confirm it with the human before applying** — `injectionPoint` can
be absent, ambiguous, or list both, and applying against a wrong guess burns a
redeploy. `outbound` ⇒ you need `--upstreamId`. (If two outbound policies share the
same characteristic, a second one needs `--allowDuplicated`.)

## Step 5 — Verify the governed diff

Redeploy, wait ~30–45s for the config to propagate, then run the JSON-RPC/SSE
handshake **through the gateway** and diff `tools/list` against the Step 1
baseline. **MCP is not a static GET** — you must `initialize` (capture the session
id), send `notifications/initialized`, then `tools/list`:

```bash
GW="https://<gw-public-host>/<label>/mcp"
HDR='-H Content-Type:application/json -H Accept:application/json,text/event-stream -H Accept-Encoding:identity'

# 1) initialize — capture mcp-session-id from the RESPONSE HEADERS
SID=$(curl -sS --max-time 10 -D - $HDR -X POST "$GW" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}' \
  | grep -i '^mcp-session-id:' | awk '{print $2}' | tr -d '\r')

# 2) notifications/initialized — then tools/list, reusing the session id
curl -sS --max-time 10 $HDR -H "mcp-session-id: $SID" -X POST "$GW" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'
curl -sS --max-time 10 $HDR -H "mcp-session-id: $SID" -X POST "$GW" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'   # SSE: read the data: line
```

Responses arrive as SSE `data:` lines — parse the JSON off the `data:` prefix.
Force `Accept-Encoding: identity` (gzip breaks the SSE read) and use `--max-time`
(the stream stays open). Assert each rule fired. Example matrix for a tool-list
governor:

| Tool | Ungoverned (direct mock) | Governed (via gateway) | Rule |
|---|---|---|---|
| public one | present | present, unchanged | default-allow |
| rewrite target | original desc | new desc | rewrite |
| deny (exact name) | present | **absent** | deny exact |
| deny (glob) | present | **absent** | deny pattern |

If the policy governs `tools/call` instead of `tools/list`, run the call for a
denied vs allowed tool and diff the results the same way.

## Common Mistakes

- **Skipping Step 0.** Building before confirming the transport, the governed
  method (`tools/list` vs `tools/call`), and inbound-vs-outbound wastes the loop.
- **Tripping A2D's "Issues Found" quality gate.** The validator hard-fails on:
  missing provider **organization**/**URL** on the server; a tool `description`
  under **200 chars** (bites trivial tools like `health_ping`); a tool property
  `description` under **25 chars**; a tool property `name` under **5 chars**
  (`text`, `id`). Set all of these when you build the mock (Step 1) and mirror them
  in `mcp-metadata.json` (Step 2) — don't discover them at publish.
- **Reaching for a `protocol` field, or expecting a `transport` attribute.** MCP
  has no v0.3/v1.0 split and no `properties.protocol`. The asset is made governable
  by **`type=mcp`**; the transport is declared inside `mcp-metadata.json`
  (`transport.kind`). **Verified:** a published MCP asset's `attributes[]` carries
  only `platform` — don't query for a `transport` attribute or gate on one.
- **Trusting A2D's Exchange publish.** Hand-author `mcp-metadata.json` + raw POST
  (Step 2), same as the A2A card.
- **Pointing the instance at the app artifact.** The `<name>-mule-application`
  app-typed asset is not governable; publish and target the `type=mcp` asset.
- **Using `endpoint.type=rest` for an mcp asset** → rejected. Use `mcp` +
  `technology=flexGateway` + `isCloudHub:null`.
- **Deploying via `api/v1` on a managed/Omni Flex GW** → returns 200 but leaves
  `deployment: null`; the instance never deploys. Use the `apimanager/xapi/v1`
  PATCH surface, with `routing.upstreams` referencing the upstream **by id**.
- **Putting `/mcp` on the upstream URI.** Upstream is host-root only; the MCP
  Support policy adds the `/mcp` surface. A path on the upstream mis-routes.
- **MCP Support not first in the chain.** It must run before every other policy to
  preserve SSE framing + `Mcp-Session-Id`; otherwise the handshake breaks and the
  tools policy sees no session.
- **Verifying with a static GET.** MCP needs the JSON-RPC handshake
  (`initialize` → capture `mcp-session-id` → `notifications/initialized` →
  `tools/list`) over SSE. A bare GET returns nothing useful.
- **Forgetting `Accept-Encoding: identity` / `--max-time`.** gzip breaks the SSE
  read; the open stream hangs a probe without a timeout.
- **Applying an outbound policy without `--upstreamId`** → "can not be applied as
  inbound". Check `injectionPoint` first.
- **Verifying too soon.** Gateway config propagation takes tens of seconds after
  redeploy; a probe fired immediately reads the pre-policy tool list.
- **Missing trailing `/`** on proxy path or upstream URI — the managed Flex GW
  rejects or mis-routes.
- **Publishing without a `home.md`.** A newly published (or hard-delete +
  republished) asset renders a blank catalog page until a home page exists — add
  one (Step 2.6). **For mcp assets the v2 `pages/home` route 404s** (so does `GET
  .../pages`); use the **v1 `portal/draft` PUT + `portal` PATCH** surface instead
  (verified against live mcp assets in a real org).
- **Skipping the connected-app credentials.** Every raw Exchange/API Manager call
  needs a bearer token minted from the human-supplied `client_id`/`client_secret`
  (Step 0 / Step 2.3) — collect them up front, don't stall mid-loop.

## Source Ref

- <https://docs.mulesoft.com/pdk/latest/policies-pdk-policy-templates> — PDK policy
  templates (public).
- MCP manifest shape: `protocolVersion` + `transport {kind: streamableHttp, path}`
  + `tools[]` (each with `inputSchema`/`outputSchema`). Asset made governable at
  publish time via `type=mcp`; the transport lives in the manifest, not an
  Exchange attribute.
- Streamable-HTTP handshake: `initialize` (capture `mcp-session-id`) →
  `notifications/initialized` → `tools/list` / `tools/call`; SSE `data:` lines;
  `Accept-Encoding: identity`.

_Snapshot: 2026-08-02. **Publish step verified live** against an EU Anypoint org:
raw multipart `-F type=mcp` + `files.mcp-metadata.json` → 202 → `completed`,
published as `type: mcp` (attributes carry only `platform`; transport is in the
manifest, NOT a `properties.transport` attribute — the earlier claim was wrong and
is corrected here); throwaway asset hard-deleted after. **A2D quality-gate
minimums** (provider org+URL required; tool description ≥200 chars; property
description ≥25 chars; property name ≥5 chars) captured from a live "Issues Found"
report and baked into Step 1/Step 2. The gateway/apply/verify
facts (endpoint.type=mcp, xapi/v1 deploy, MCP-Support-first chain, JSON-RPC/SSE
tools/list verification) are sourced from a live streamable-HTTP MCP server fronted
by an Agent Fabric Ingress managed Flex Gateway. **Home-page route now confirmed:**
the v2 `pages/home` PUT (and `GET .../pages`) **404 for `type=mcp` assets** — every
published mcp asset in the org lacks the v2 pages route; the working path is the v1
**`portal/draft` PUT + `portal` PATCH** surface the UI uses (verified live — page
published and rendered on the catalog card). See Step 2.6. **`tools/call`-governing
policy verified end-to-end** (2026-08-02): an inbound MCP Tool Token Rate Limit
policy over an A2D mock (default/override/unmetered tiers) — the gateway returned
`200` then `429` with `X-TokenLimit-Limit/Remaining/Reset` + `Retry-After` and a
plain-JSON (not SSE) JSON-RPC `-32000` error body; override tier showed its own
higher limit, unmetered tier emitted no headers. This confirmed two corrections
baked into Step 3/Step 4: the **upstream URI is the surface-minus-`mcp` prefix**
(the A2D mock needs `…/api/platform/<serverId>/`, not the host root), and the
**raw-REST `…/apis/{id}/policies` apply** path works when the CLI isn't
authenticated (MCP Support's GAV lives in a MuleSoft group, not your org)._
