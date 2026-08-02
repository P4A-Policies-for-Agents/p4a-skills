---
name: p4a-test-a2a-policies-with-a2d
description: Use when testing an A2A gateway policy end-to-end against a realistic agent — building a mock A2A agent in A2D (Agent-to-Discovery), publishing its Agent Card to Anypoint Exchange as a type=agent asset, fronting it with a Flex/Omni Gateway API instance, applying the policy under test, and diffing the governed Agent Card against the ungoverned mock to prove the policy fired. Covers the confirm-protocol-version-first step and the fact that protocol version is set by the publish-time properties.protocol value (a2a vs a2a_v1), NOT the card body shape; the working publish method (A2D's publish_to_exchange is broken — use a hand-authored card + raw multipart POST); creating the gateway instance per version (v0.3 via api-mgr CLI --type a2a, v1.0 via raw REST endpoint.type=a2a_v1 + flexGateway + isCloudHub null); and the inbound-vs-outbound policy apply distinction (outbound needs --upstreamId). Do NOT use for writing the policy's PDK/Rust code (see the omni-gateway-pdk-skills repo), for the PDK-local cargo test harness (that tests the policy in isolation, not against a live agent), or for driving the P4A MCP server to deploy a catalog policy (see p4a-mcp-usage).
---

# P4A: Test A2A Policies with A2D

## Overview

An A2A gateway policy (e.g. an Agent Card governor, an A2A auth policy, a
skill-visibility filter) can only be *fully* proven against a real Agent Card
served over the wire. The PDK-local `cargo test` harness exercises the policy in
isolation; this skill covers the **integration loop** — a live mock agent behind
a real Flex/Omni Gateway instance:

```
A2D mock agent  ──►  Exchange (type=agent asset)  ──►  Flex GW API instance  ──►  policy applied
     │                                                        │
     └─ ungoverned Agent Card (baseline)          governed Agent Card (verify the diff)
```

The **proof** is the diff: fetch the Agent Card directly from the A2D mock
(ungoverned baseline), fetch it again through the gateway (policy applied), and
confirm the policy transformed it as configured.

## When to Use

- Testing any A2A policy against a live Agent Card over HTTP, not just unit tests.
- Demonstrating a policy's effect (skill hiding, description rewrite, auth gating)
  with a real before/after Agent Card diff.
- Standing up a disposable mock agent when you don't have a real one to point at.

**Do NOT use** for:

- Writing the policy's PDK/Rust code — see the `omni-gateway-pdk-skills` repo.
- The PDK-local `cargo test` / `make test` harness — that tests the policy in
  isolation with a stubbed request, not against a live agent. Run it first; this
  skill is the layer above it.
- Deploying a catalog policy through the P4A MCP server — see [[p4a-mcp-usage]].
- Publishing a *non-agent* asset (REST/MCP/LLM) — the publish shape differs.

## Step 0 — Confirm the protocol version FIRST

**Before building anything, ask the human which A2A protocol version(s) to test.**
The version drives how you **publish** and how you create the **gateway
instance** — NOT the Agent Card body shape (see the boxed warning below).

- **v0.3:** publish with `properties.protocol=a2a`; gateway instance
  `endpoint.type=a2a`. The `anypoint-cli-v4` api-mgr commands support this
  directly (`--type a2a`).
- **v1.0:** publish with `properties.protocol=**a2a_v1**`; gateway instance
  `endpoint.type=a2a_v1`. The CLI has **no** `a2a_v1` type — the v1.0 instance
  must be created via **raw API Manager REST** (Step 3).

> **The protocol version is set by the `properties.protocol` value at publish
> time — it is NOT derived from the Agent Card body.** Exchange also emits a
> separate, cosmetic `protocol-version` attribute derived from card shape
> (top-level `url` present ⇒ `v0.3`), but that attribute is *not* what drives the
> catalog display or the API Manager endpoint-type gate — `properties.protocol`
> is. Publishing a genuine v1.0 card with `properties.protocol=a2a` still tags the
> asset (and every downstream instance) "A2A v0.3". Verified against the Anypoint
> UI publish flow (HAR): the portal sends `properties.protocol=a2a_v1` for a v1.0
> agent.

A policy may behave differently across versions (different field to rewrite,
different place skills live). Do not assume — confirm which the human wants, and
whether they want **both** (two Exchange assets + two gateway instances) or one.
Also confirm the **surface**: public well-known card only, or extended card too.

If the mock has no extended card, extended-surface features are out of scope —
say so and test the public well-known card (`GET /.well-known/agent-card.json`).

**Also ask the human for a connected-app `client_id` and `client_secret`.** Every
raw Exchange and API Manager REST call in this skill (publish, poll, delete,
external instance, instance create/PATCH, upstream fetch) authenticates with a
bearer token minted from these credentials via `client_credentials` (Step 2).
The connected app must have the Exchange and API Manager scopes for the target
org/env. Do NOT hardcode or echo the secret — take it from the human, keep it in
an env var (e.g. `$CLIENT_ID` / `$CLIENT_SECRET`), and reuse the token for all
calls below.

## Step 1 — Build the mock agent in A2D

Use the A2D MCP tools to design a mock agent whose Agent Card gives the policy
something to act on. For a skill/visibility policy, that means a deliberate mix:

- a **public** skill that should stay (control),
- a skill targeted for **rewrite** (verify the text changes),
- a skill targeted for **exact-id deny** (verify it disappears),
- a skill matching a **glob deny** pattern (verify pattern matching).

Confirm the live mock serves them all: `GET <mock-base>/.well-known/agent-card.json`
should list every skill. This is your **ungoverned baseline** — save it.

## Step 2 — Publish the Agent Card to Exchange as `type=agent`

> **A2D's `publish_to_exchange_agent_card` is broken for this.** It serializes
> from `get_agent_card_spec`, which emits a HOLLOW card (`skills: []`,
> `supportedInterfaces:[{url:""}]`) — Exchange's agent-domain validator rejects it
> (`400 INVALID_ASSET_METADATA / "no registered domain plugin"`). The live mock
> endpoint serves skills correctly; only the Exchange builder is hollow. **Publish
> by hand instead.**

1. **Hand-author a card JSON.** AMF requires a **top-level `url`** — a card
   without it is rejected (`400 "required key [url] not found"`), regardless of
   protocol version. A v1.0 card may *also* carry `supportedInterfaces[]`, but
   that doesn't change the tag (the `properties.protocol` field does — see Step 0).
   Each skill needs `id`/`name`/`description`/`tags` and (if the mock declares
   them) `inputModes`/`outputModes`.

2. **Get a token** — `POST /accounts/api/v2/oauth2/token`, `grant_type=client_credentials`,
   form-encoded `client_id`/`client_secret`. Use the connected-app credentials the
   human supplied in Step 0 (`$CLIENT_ID` / `$CLIENT_SECRET`); mint once and reuse
   the bearer token for every raw REST call in Steps 2–4.

   ```bash
   TOKEN=$(curl -sS -X POST https://anypoint.mulesoft.com/accounts/api/v2/oauth2/token \
     -d grant_type=client_credentials \
     -d "client_id=$CLIENT_ID" -d "client_secret=$CLIENT_SECRET" | jq -r .access_token)
   ```

3. **Raw multipart POST** (the CLI's `exchange asset upload` has no `agent` type —
   it won't work). Endpoint carries **org AND full GAV** in the path:

   ```
   POST /exchange/api/v2/organizations/{ORG}/assets/{groupId}/{assetId}/{version}
     -F type=agent
     -F name=...
     -F status=published
     -F description=...                   # keep concise
     -F properties.protocol=a2a           # v0.3  ── or ──  a2a_v1  for v1.0
     -F properties.platform=a2d
     -F "files.a2a-card.json=@card.json;type=application/json"
   ```

   **`properties.protocol` is the protocol-version lever** (Step 0): `a2a` → the
   asset displays "A2A v0.3"; `a2a_v1` → "A2A v1.0" and unlocks the `a2a_v1`
   gateway instance type (Step 3). The card rides as a file field named
   **`files.a2a-card.json`**. The bare `/assets` and org-only routes 404 or hit
   the wrong facade.

4. **Poll** — response is **202** + a `publicationStatusLink`. Poll it (~2s) until
   `status: completed` (statusCode 201). `409` = already published (bump version to
   re-publish). Confirm the tag stuck:
   `GET /exchange/api/v2/assets/{ORG}/{assetId}/{version}` → `attributes[]` should
   show `{key:"protocol", value:"a2a_v1"}` (the `protocol-version` attribute may
   still read `v0.3` — that one is cosmetic and card-shape-derived; ignore it).

5. **Always add a `home.md` portal home page.** A freshly published asset has a
   blank catalog page until a home page exists — `PUT` one every time you publish
   (and again after any hard-delete + republish, which drops portal pages). Author
   a `home.md` describing the mock agent (what it is, its skills, the raw + governed
   endpoints) and upload it as the asset's home:

   ```
   PUT /exchange/api/v2/assets/{ORG}/{assetId}/{version}/pages/home
     Authorization: Bearer $TOKEN
     Content-Type: text/markdown
     --data-binary @home.md            # → 200/201; page now renders on the catalog
   ```

6. **Hard-delete** a mistaken asset via `DELETE /exchange/api/v2/assets/{group}/{asset}/{version}`
   + header `X-Delete-Type: hard-delete` (→ 204). The CLI has no hard-delete flag.
   Hard-delete drops the asset's portal pages too — re-`PUT` the `home.md` (step 5)
   after republishing.

See [[publish-a2a-agent-to-exchange]] for the full recipe and field list.

## Step 3 — Front the agent with a Flex Gateway API instance

The `create_and_manage_api_instances` MCP tool only does `mule4`/`basicEndpoint`
— it **cannot** create a Flex a2a instance. The mock's public URL is a valid
**external upstream** — no deployed backing app required. How you create the
instance depends on the protocol version:

### v0.3 — `anypoint-cli-v4` (`--type a2a`)

```bash
# create: --type a2a, external upstream, proxy path + upstream must end with "/"
anypoint-cli-v4 api-mgr:api:manage <assetId> <version> <ORG> \
  --environment "<ENV>" --isFlex --withProxy \
  --scheme http --port 8081 --path /<label>/ \
  --type a2a --uri "<mock-base-url>/" \
  --apiInstanceLabel <label>            # → "Created new API with ID: <id>"
```

### v1.0 — raw REST (`endpoint.type=a2a_v1`)

The CLI's `--type` enum has **no `a2a_v1`** — a v1.0 instance must be created via
raw API Manager REST against an asset published with `properties.protocol=a2a_v1`
(Step 2). The three fields that make it validate (each learned from a distinct
400):

```bash
POST /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis
{
  "technology": "flexGateway",                       # else: "a2a_v1 endpoint requires flexGateway technology"
  "endpoint": {
    "type": "a2a_v1",                                # else APIM tags it v0.3
    "deploymentType": "HY",
    "isCloudHub": null,                              # MUST be null, not false, and MUST be present
    "proxyUri": "http://0.0.0.0:8081/<label>/",
    "uri": "<mock-base-url>/"
  },
  "spec": {"groupId":"<ORG>","assetId":"<assetId>","version":"<version>"},
  "instanceLabel": "<label>"
}
```

If the linked asset is still `properties.protocol=a2a`, this 400s with
`"a2a_v1 endpoint type using agent asset type must have an A2A v1 protocol"` —
fix the asset (Step 2), don't fight the instance.

### Both versions — then PATCH endpointUri + deploy

```bash
# the CLI --endpointUri flag doesn't persist on managed Flex GW — PATCH it:
PATCH /apimanager/api/v1/organizations/{ORG}/environments/{ENV_ID}/apis/{id}
  {"endpointUri": "https://<gw-public-host>/<label>/"}

# deploy onto the gateway target
anypoint-cli-v4 api-mgr:api:deploy <id> --environment "<ENV>" \
  --target <GW_TARGET_ID> --gatewayVersion 1.0.0 \
  --applicationName <label>-ingress
```

**Live-probe constraints (managed Flex GW):** `scheme=http` (TLS terminates at the
gateway), listener `port=8081`, both proxy path and upstream URI must end with
`/`, `--gatewayVersion 1.0.0` works even on a newer live gateway.

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
  {"name":"A2D Mock","endpointUri":"<raw-mock-url>/","isPublic":false}
```

- `{versionGroup}` is the asset's version group (e.g. `v1.0`), **not** the version
  (`1.0.0`) — get it from `GET /exchange/api/v2/assets?...&types=agent` (`versionGroup` field).
- `endpointUri` is the raw A2D mock (`https://www.a2d-ai.com/api/platform/<mock-id>/a2a/`)
  — trailing `/` matches the UI.
- Returns **201** with `"type":"external"` and the new instance `id`. Verified
  against the Anypoint Exchange UI "add endpoint" flow (HAR).

## Step 4 — Apply the policy under test

Resolve the policy GAV (`anypoint-cli-v4 exchange asset list <assetId>` → the
`type=policy` row, not `policy-implementation`). Then:

```bash
anypoint-cli-v4 api-mgr:policy:apply <instanceId> <policyAssetId> \
  --environment "<ENV>" --groupId <ORG> --policyVersion <ver> \
  -c "$(cat config.json)"          # policy config JSON
anypoint-cli-v4 api-mgr:api:redeploy <instanceId> --environment "<ENV>"
```

**Inbound vs outbound — this bites.** A policy that transforms the *response*
(e.g. rewriting the Agent Card the agent returns) is **outbound**
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

Check the policy's `gcl.yaml` `metadata/capabilities/injectionPoint` before
applying — `outbound` ⇒ you need `--upstreamId`.

## Step 5 — Verify the governed diff

Redeploy, wait ~30–45s for the config to propagate, then fetch **through the
gateway** and diff against the Step 1 baseline:

```bash
curl -sS "https://<gw-public-host>/<label>/.well-known/agent-card.json" | jq
```

Assert each rule fired. Example matrix for a skill-governor policy:

| Skill | Ungoverned (direct mock) | Governed (via gateway) | Rule |
|---|---|---|---|
| public one | present | present, unchanged | default-allow |
| rewrite target | original desc | new desc | rewrite |
| deny (exact id) | present | **absent** | deny exact |
| deny (glob) | present | **absent** | deny pattern |

To genuinely exercise both versions, back each instance with its **own A2D mock**
retagged to that protocol (`update_agent_card protocol_version=0.3.0|1.0`) — the
live v0.3 mock emits a top-level `url`, the v1.0 mock emits `supportedInterfaces`.
If both instances instead point at the *same* mock, their governed responses are
**identical** (one card shape), so the version split lives only in the two
Exchange assets, not the wire — note that so it doesn't read as a bug.

## Common Mistakes

- **Skipping Step 0.** Building before confirming the version wastes the whole
  loop when the human wanted the other version — or both.
- **Publishing v1.0 with `properties.protocol=a2a`.** The asset tags "A2A v0.3"
  no matter how the card body looks, and the `a2a_v1` gateway instance then 400s
  (`must have an A2A v1 protocol`). The protocol is set by `properties.protocol`,
  not the card shape — use `a2a_v1` for v1.0.
- **Chasing the tag via card shape.** Adding/removing `supportedInterfaces`,
  `additionalInterfaces`, or `preferredTransport` does **not** change the tag.
  Neither does an explicit `attributes.protocol-version` form field (ignored).
  Only `properties.protocol` moves it.
- **Trusting A2D's Exchange publish.** It emits a hollow card. Hand-author + raw
  POST (Step 2).
- **Omitting the top-level `url` on any card.** Exchange rejects it
  (`required key [url] not found`) — required for v0.3 *and* v1.0.
- **Using the CLI for a v1.0 instance.** `--type` has no `a2a_v1`; use raw REST
  with `endpoint.type=a2a_v1` + `technology=flexGateway` + `isCloudHub:null`.
- **Applying an outbound policy without `--upstreamId`** → "can not be applied as
  inbound". Check `injectionPoint` first.
- **Verifying too soon.** Gateway config propagation takes tens of seconds after
  redeploy; a probe fired immediately reads the pre-policy card.
- **Missing trailing `/`** on proxy path or upstream URI — the managed Flex GW
  rejects or mis-routes.
- **Publishing without a `home.md`.** A newly published (or hard-delete +
  republished) asset renders a blank catalog page until you `PUT` a home page —
  always upload one (Step 2.5).
- **Skipping the connected-app credentials.** Every raw Exchange/API Manager call
  needs a bearer token minted from the human-supplied `client_id`/`client_secret`
  (Step 0 / Step 2.2) — collect them up front, don't stall mid-loop.

## Source Ref

- <https://docs.mulesoft.com/pdk/latest/policies-pdk-policy-templates> — PDK policy
  templates (public).
- A2A Agent Card schema: v0.3 vs v1.0. Protocol version is set at publish time via
  `properties.protocol` (`a2a` / `a2a_v1`), confirmed against the Anypoint UI
  publish flow.

_Snapshot: 2026-08-02 (verified end-to-end for BOTH protocol versions: A2D v0.3 +
v1.0 mocks → Exchange type=agent (a2a / a2a_v1) → Flex GW a2a + a2a_v1 instances →
outbound Agent-Card governor → governed-card diff)._
