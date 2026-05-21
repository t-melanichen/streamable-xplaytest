# Bruno Collection: GSSV Offering Registry

A Bruno collection for exercising GSSV's `OfferingsController` (`/v1/offerings`) and title endpoints. Use this to test the create/update/delete offering flow before wiring it up in PlayTest code.

## Install Bruno

```powershell
winget install Bruno.Bruno
```

Or download from https://www.usebruno.com/

## Open the collection

1. Bruno → **Open Collection** → point at this folder (`streamable-xplaytest\bruno\offering-registry`).
2. Bruno → top-right environment dropdown → pick **dev** (start here — auth is bypassed in dev).

## Environments

| Env | Base URL | Auth | Notes |
|---|---|---|---|
| `dev` | `http://localhost:9005` | **None — bypassed** | Requires you to run `services.partnerregistry` locally. **Start here.** |
| `dev-hosted` | `https://gssv-dev-test.xboxlive.com/api/partnerregistry` | Real Xbox Live S2S bearer | Hosted "test" environment. ADO PATs and personal AAD tokens are rejected at the edge — see "Hosted endpoints" below. |
| `int` | `https://gssv-dev-int.xboxlive.com/api/partnerregistry` | Real Xbox Live S2S bearer | Same auth model as `dev-hosted`. |
| `prod` | `https://gssv-dev-prod.xboxlive.com/api/partnerregistry` | Real Xbox Live S2S bearer | DO NOT use for testing. Real production. |

URLs verified from `services.partnerregistry/src/Product/PartnerRegistryService/appsettings.en-development.json:29` and `appsettings.en-int.json` / `appsettings.en-prod.json`. Local port 9005 from `appsettings.en-development.json:9`.

## Quick start (recommended: run the service locally)

This is the fastest path to a working `200 OK` and is what you should do for solo dev.

```powershell
cd "C:\Users\t-melanichen\OneDrive - Microsoft\Desktop\services.partnerregistry\src\Product\PartnerRegistryService"
$env:ASPNETCORE_ENVIRONMENT = "Development"
dotnet run
```

The service listens on `http://localhost:9005` (per `appsettings.en-development.json`). The S2S auth policy is bypassed in `Development` (see `Startup.cs:386` — `policy.RequireAssertion(context => true)`), so Bruno requests don't need a real token.

Then:
1. In Bruno pick the **dev** environment
2. Fill in `partnerId`, `productId` in `environments/dev.bru`
3. Run **`Offerings/put-offering.bru`** → expect `200`/`204`

> ⚠️ Local-run dependencies you may also need: a CosmosDB emulator (`appsettings.en-development.json:20-23` points at `https://localhost:8081`). Install from https://learn.microsoft.com/azure/cosmos-db/local-emulator if you don't already have it.

## Hosted endpoints (`dev-hosted` / `int` / `prod`) — why your token probably won't work

Verified empirically 2026-05-21: hitting `GET https://gssv-dev-test.xboxlive.com/api/partnerregistry/v1/offerings` with each of the following returns `HTTP 401`, empty body, no `WWW-Authenticate` header:

- No `Authorization` header at all
- An Azure DevOps PAT as `Bearer`
- An `az account get-access-token` against `https://xboxlive.com` (fails to mint — `xboxlive.com` isn't an AAD resource in the Microsoft tenant: `AADSTS500011`)

The 401 is coming from the **Xbox Live edge gateway**, not the partner registry service. The edge expects an Xbox Live S2S bearer (likely XSTS, not raw AAD JWT). After the edge, the service still enforces `ApplicationAuthorizationRequirement` against `RegistryManagementConfig.AllowedS2SAppIds` (see `appsettings.en-int.json:27-29`, e.g., `57280b9e-304b-43cf-87b5-dc8644677f0c` for ostg). So you need **both**:
1. A token the edge accepts (Xbox Live S2S, format TBD — onboarding ask for GSSV team)
2. Your app id in the allowlist (onboarding ask for GSSV team)

Until those are sorted, **stick to the `dev` local-run env**. Tracked in `design/open-questions-for-team.md`.

## Variables to set per environment

Open `environments/<env>.bru` and edit:

| Var | What to put |
|---|---|
| `partnerId` | Your playtest's `SellerId`. Ask Brian Bowman for a test seller id, or find one in PlayTest test fixtures. |
| `productId` | A real `PartnerCenterProductId` from a published or in-flight playtest |
| `offeringId` | Anything unique. Convention from `field-mapping.md`: `xpt-{playtestId-no-hyphens}` |
| `titleId` | Convention: `t-{playtestId-no-hyphens}` |
| `dnaGroupId1` | A DNA group GUID. Get one from GMS by calling `IGMSServiceClient.GetUserGroupAsync` and looking at the returned `UserDnAGroupIds[]`. For dev testing, any GUID works. |
| `bearerToken` (secret) | Only needed for int/prod. See "Getting a bearer token" below. |

## Getting a bearer token (only for `dev-hosted`/`int`/`prod`)

> ⚠️ This is currently a blocked path. Verified empirically that ADO PATs and personal-AAD tokens are rejected by the Xbox Live edge. Until GSSV onboards us, **use the `dev` (local-run) env**. Onboarding questions tracked in `design/open-questions-for-team.md`.

The auth policy on `OfferingsController` is `PartnerRegistryAuthPolicy.S2S` (see `services.partnerregistry/.../Configuration/PartnerRegistryAuthPolicy.cs`). In non-Development environments it requires:
- A valid Xbox Live S2S bearer (token format TBD — XSTS vs AAD-via-GSSV-helper)
- The token's app id must be in `RegistryManagementConfig.AllowedS2SAppIds` (see `appsettings.en-int.json:27-29`)

When unblocked, paste the token into the `bearerToken` secret var in the active environment.

## Requests in this collection

### `Offerings/`
1. **put-offering.bru** — `PUT /v1/offerings/{id}` — upsert the offering. **Start here.**
2. **get-offering.bru** — `GET /v1/offerings/{id}` — read it back
3. **get-all-offerings.bru** — `GET /v1/offerings` — useful for finding existing offerings to use as templates
4. **delete-offering.bru** — `DELETE /v1/offerings/{id}`

### `Titles/`
1. **get-titles.bru** — `GET /v1/offerings/{id}/titles`
2. **put-title.bru** — `PUT /v1/offerings/{id}/titles/{titleId}?partnerid=...` — attach a title to the offering
3. **delete-title.bru** — `DELETE /v1/offerings/{id}/titles/{titleId}?partnerid=...`

## Workflow: create a private playtest offering end-to-end

1. Pick the `dev` environment
2. Fill in `partnerId`, `productId`, `offeringId`, `titleId`, `dnaGroupId1` in `environments/dev.bru`
3. Run **`Offerings/put-offering.bru`** → expect `200 OK` (or `204`)
4. Run **`Offerings/get-offering.bru`** → confirm the offering came back with the fields you sent
5. Run **`Titles/put-title.bru`** → expect `200 OK`
6. Run **`Titles/get-titles.bru`** → confirm the title is attached
7. Run **`Offerings/delete-offering.bru`** to clean up

## Caveats

- The `OfferingV2` body in `put-offering.bru` has **placeholder values** for streaming-infra fields (`Regions`, `ServiceLevel`, `DefaultAllocationPools`, `SystemUpdateGroupWeights`, etc.). These need to be confirmed with the GSSV team. The minimum required-for-validation set may be smaller — try a barebones PUT first to see what fields the server actually requires vs defaults.
- `OfferingV2` is also subject to server-side validation rules (see `services.partnerregistry/src/Product/PartnerRegistryService/Processors/Validation/`). Read the 4xx response body if your PUT is rejected — it'll list the validation errors.
- `partnerid` on the title routes is a **query string** parameter, not a path param. Easy to get wrong.
- The verb for offering creation is **PUT** (upsert), not POST. There is no `POST /v1/offerings`.

## See also

- `design/field-mapping.md` — every field on `OfferingV2`/`Title`/`Flight` mapped to its PlayTest source
- `research/partner-registry.md` — full API surface and contract notes
- `architecture/flights-and-dna-groups.md` — how `dnaGroupId1` resolves from a playtest audience
