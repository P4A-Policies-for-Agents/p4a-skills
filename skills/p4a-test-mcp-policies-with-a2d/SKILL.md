---
name: p4a-test-mcp-policies-with-a2d
description: Use when testing an MCP gateway policy end-to-end against a realistic server — building a mock MCP server in A2D (Agent-to-Discovery), publishing its manifest to Anypoint Exchange as a type=mcp asset, fronting it with a Flex/Omni Gateway API instance, applying the policy under test, and diffing the governed tools/list against the ungoverned mock to prove the policy fired. Covers the confirm-transport-first step and the fact that MCP has no v0.3/v1.0 split — the asset is pinned by the publish-time properties.transport value (streamablehttp), NOT a protocol field; the working publish method (A2D's publish is unreliable — hand-author mcp-metadata.json + raw multipart POST, as for A2A); the gateway instance (endpoint.type=mcp, rest rejected; technology=flexGateway; isCloudHub null; deploy via apimanager/xapi/v1 because api/v1 silently leaves deployment null on managed/Omni Flex GW); the MCP-Support-must-be-first-in-chain rule (it preserves SSE framing + Mcp-Session-Id downstream); and verifying over a JSON-RPC/SSE handshake (initialize → notifications/initialized → tools/list), NOT a static GET. Do NOT use for writing the policy's PDK/Rust code (see omni-gateway-pdk-skills), the PDK-local cargo test harness, testing an A2A Agent Card policy (see p4a-test-a2a-policies-with-a2d), or driving the P4A MCP server to deploy a catalog policy (see p4a-mcp-usage).
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
dance. The lever that pins the asset is `properties.transport=streamablehttp`.

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

- **Transport.** This skill covers **streamable-HTTP** (`properties.transport=streamablehttp`;
  gateway `endpoint.type=mcp`). SSE-only servers behave differently at the wire;
  confirm the mock and the policy target streamable-HTTP.
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
design a mock server whose tool list gives the policy something to act on. For a
tool-list/visibility policy, that means a deliberate mix:

- a **public** tool that should stay (control),
- a tool targeted for **description/annotation rewrite** (verify the text changes),
- a tool targeted for **exact-name deny** (verify it disappears),
- a tool matching a **glob/pattern deny** (verify pattern matching).

Give each tool a real `inputSchema`/`outputSchema` and `mockScenarios` so
`tools/call` returns deterministic data (`===` / `exists` operators; append the
`exists` catch-all **last** so specific matches win).

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
   MCP analogue of the A2A card — it is the payload the policy will govern.

2. **Author the Exchange descriptor.** `assetId`, `groupId`, `version`, `name`,
   `description`, `tags`, and the properties block — the key field is
   **`properties.transport=streamablehttp`** (plus `properties.platform`). There
   is **no `properties.protocol`** for MCP; `transport` is what pins the asset as
   MCP and unlocks the `endpoint.type=mcp` gateway instance (Step 3).

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
     -F type=mcp
     -F name=...
     -F status=published
     -F description=...                        # keep concise
     -F properties.transport=streamablehttp    # ← the MCP lever (NOT properties.protocol)
     -F properties.platform=a2d
     -F "files.mcp-metadata.json=@mcp-metadata.json;type=application/json"
   ```

   The manifest rides as a file field named **`files.mcp-metadata.json`**. The
   bare `/assets` and org-only routes 404 or hit the wrong facade.

5. **Poll** — response is **202** + a `publicationStatusLink`. Poll it (~2s) until
   `status: completed` (statusCode 201). `409` = already published (bump version to
   re-publish). Confirm the tag stuck:
   `GET /exchange/api/v2/assets/{ORG}/{assetId}/{version}` → the asset should show
   `type: mcp` and the `transport` property.

6. **Always add a `home.md` portal home page.** A freshly published asset has a
   blank catalog page until a home page exists — `PUT` one every time you publish
   (and again after any hard-delete + republish, which drops portal pages). Author
   a `home.md` describing the mock server (what it is, its tools + sensitivity, the
   transport/path, the raw + governed endpoints) and upload it:

   ```
   PUT /exchange/api/v2/assets/{ORG}/{assetId}/{version}/pages/home
     Authorization: Bearer $TOKEN
     Content-Type: text/markdown
     --data-binary @home.md            # → 200/201; page now renders on the catalog
   ```

7. **Hard-delete** a mistaken asset via `DELETE /exchange/api/v2/assets/{group}/{asset}/{version}`
   + header `X-Delete-Type: hard-delete` (→ 204). The CLI has no hard-delete flag.
   Hard-delete drops the asset's portal pages too — re-`PUT` the `home.md` (step 6)
   after republishing.

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
    "uri": "<mock-base-url>/"                  # host ROOT only — see note below
  },
  "spec": {"groupId":"<ORG>","assetId":"<assetId>","version":"<version>"},
  "instanceLabel": "<label>"
}
```

> **Upstream URI is the host root, with NO `/mcp` path.** The MCP Support policy
> maps the gateway route onto the server's `/mcp` streamable-HTTP surface, so the
> upstream carries no path — just the host, trailing `/`. Putting `/mcp` on the
> upstream double-paths the route and mis-routes the handshake.

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
  "upstreams": [ { "id":"<upid>", "uri":"<mock-base-url>/", "label":null, "tlsContext":null } ],
  "routing":   [ { "upstreams":[ { "id":"<upid>", "weight":100 } ] } ],
  "deployment": { "environmentId":"<ENV_ID>", "type":"HY", "expectedStatus":"deployed",
                  "overwrite":false, "targetId":"<GW_TARGET_ID>", "targetName":"<GW_NAME>",
                  "gatewayVersion":"1.0.0" }
}
```

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
- **Reaching for a `protocol` field.** MCP has no v0.3/v1.0 split and no
  `properties.protocol` — the asset is pinned by **`properties.transport=streamablehttp`**
  and `type=mcp`. Setting a protocol field does nothing.
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
  republished) asset renders a blank catalog page until you `PUT` a home page —
  always upload one (Step 2.6).
- **Skipping the connected-app credentials.** Every raw Exchange/API Manager call
  needs a bearer token minted from the human-supplied `client_id`/`client_secret`
  (Step 0 / Step 2.3) — collect them up front, don't stall mid-loop.

## Source Ref

- <https://docs.mulesoft.com/pdk/latest/policies-pdk-policy-templates> — PDK policy
  templates (public).
- MCP manifest shape: `protocolVersion` + `transport {kind: streamableHttp, path}`
  + `tools[]` (each with `inputSchema`/`outputSchema`). Asset pinned at publish
  time via `type=mcp` + `properties.transport=streamablehttp`.
- Streamable-HTTP handshake: `initialize` (capture `mcp-session-id`) →
  `notifications/initialized` → `tools/list` / `tools/call`; SSE `data:` lines;
  `Accept-Encoding: identity`.

_Snapshot: 2026-08-02 (mirrors [[p4a-test-a2a-policies-with-a2d]]; MCP-specific
facts — type=mcp + properties.transport, endpoint.type=mcp, xapi/v1 deploy,
MCP-Support-first chain, JSON-RPC/SSE tools/list verification — verified against a
live streamable-HTTP MCP server fronted by an Agent Fabric Ingress managed Flex
Gateway instance)._
