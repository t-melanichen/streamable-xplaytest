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

| Env | Base URL | Notes |
|---|---|---|
| `dev` | `https://gssv-dev-test.xboxlive.com/api/partnerregistry` | Auth bypassed (`policy.RequireAssertion(context => true)`). Start here. |
| `int` | `https://gssv-dev-int.xboxlive.com/api/partnerregistry` | Bearer token required. AAD app must be in `RegistryManagementConfig.AllowedS2SAppIds`. |
| `prod` | `https://gssv-dev-prod.xboxlive.com/api/partnerregistry` | DO NOT use for testing. Real production. |

Source for the URLs: `services.partnerregistry/src/Product/PartnerRegistryService/appsettings.en-development.json:29` (and `appsettings.en-int.json`, `appsettings.en-prod.json`).

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

## Getting a bearer token (for int/prod only)

The auth policy on `OfferingsController` is `PartnerRegistryAuthPolicy.S2S` (see `services.partnerregistry/src/Product/PartnerRegistryService/Configuration/PartnerRegistryAuthPolicy.cs`). In non-dev environments it requires:
- A valid AAD JWT (`AuthScheme.DefaultBearer`)
- The token's app id must be in `RegistryManagementConfig.AllowedS2SAppIds` (see `appsettings.en-int.json:27-29`)

Quickest way to get a token for testing:

```powershell
# Sign in with your work account
az login

# Get a token. The resource id is GSSV's AAD app id — get the exact value from the GSSV team.
# Example using a placeholder resource id:
az account get-access-token --resource "<GSSV-RESOURCE-ID>" --query accessToken -o tsv
```

Then paste the token into the `bearerToken` secret var in the active environment.

> ⚠️ Your personal AAD account most likely does NOT have the right S2S role for int/prod.
> You'll either need (a) the PlayTest service's AAD app id added to `AllowedS2SAppIds`,
> or (b) to use the GSSV team's test app id (ask Anthony Keller).
> Until then, **stick to dev** — it bypasses auth entirely.

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
