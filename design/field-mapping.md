# P0 Field Mapping: Playtest → GSSV Offering

> **This is the central P0 deliverable.** It's the field-by-field mapping from PlayTest's playtest model to the GSSV `OfferingV2` + `Title` + `Flight` model. When a playtest is created with cloud streaming enabled, PlayTest core uses this mapping to call GSSV's `PUT /v1/offerings/{id}` (and follow-up calls).

**Key facts:**
- `services.partnerregistry` is the **GSSV offering registry** (namespace `Microsoft.GameStreaming.Partners.Contracts`). Calling it = calling GSSV.
- xPlayTest and GSSV do not talk today. This is **net-new integration** that the intern needs to build.
- The DNA group → flight mapping is **already implemented in PlayTest** for the XPackage publish workflow (`PlaytestBusinessLogic.ResolveUserDnaGroupIds` at line 1054-1123). The same pipeline is reused here for the GSSV offering.

---

## 1. Identifier mapping

| GSSV side | PlayTest side | How |
|---|---|---|
| `OfferingV2.Id` (`Id` — string) | derived from `playtest.PlaytestId` (+ prefix) | Suggested: `xpt-{playtestId-without-hyphens}`. Must be unique, deterministic, and contain only chars valid for `Id`. Stored back on the playtest as `OfferingId` for later updates/delete. |
| `OfferingV2.PartnerId` | `playtest.SellerId` (or a lookup) | If PartnerRegistry uses Seller IDs directly as PartnerIds, pass through. Otherwise add a sellerId → partnerId lookup (verify with partner team). |
| `OfferingV2.Name` | `playtest.Name` | Direct copy. Truncate to whatever max length `Id`/`string` constraints apply. |
| `OfferingV2.Notes` | `playtest.Description` (+ extras) | Useful for stuffing `PlaytestId`, `SellerId`, ContentRatings JSON, etc., so an operator can trace an offering back to its playtest. |
| `OfferingV2.OwnedBy` | `[ playtest.SellerId, "xpt-service" ]` | Useful audit field. |
| `OfferingV2.ExpirationTime` | `playtest.EndDate` | Direct copy. When the playtest ends, the offering becomes inaccessible without an explicit DELETE call (the delete still goes via the delete workflow). |

## 2. Title mapping

`PUT /v1/offerings/{offeringId}/titles/{titleId}?partnerid={partnerId}` with body `Title`:

| `Title` field | PlayTest source | Notes |
|---|---|---|
| `Id` (title source id) | derived from `playtest.PlaytestId` + `playtest.PartnerCenterProductId` | e.g. `t-{playtestId}` — must be unique within the offering. Stored on playtest. |
| `PartnerId` | `playtest.SellerId` | As above. |
| `OfferingId` | the offering id from #1 | Set by Partner Registry on PUT. |
| `Platform` | derived from the playtest's packages | If a package is XSX/XOne, use `XboxOneAndSeriesX`; if PC, use `PC`. Worth asking team whether per-platform sub-offerings or a multi-platform title is preferred. |
| `ProductId` | `playtest.PartnerCenterProductId` | Universal Store product id. Direct copy. |
| `XboxTitleId` | (not on the playtest contract) | Need to resolve from XProduct/Catalog. If unavailable, leave null. |
| `FriendlyName` | `playtest.Name` | Optional but useful for ops tooling. |
| `AllowedFlights` | (same flights as Offering, see #3) | Title-level flights act as an additional filter — set them equal to the offering flights for consistency. |
| `AvailableTime` | `playtest.StartDate` | |
| `ExpirationTime` | `playtest.EndDate` | |
| `IsEnabled` | `true` (toggle to false on suspension) | |
| `Programs` / `ProgramConfigs` | (TBD) | Likely a single "Playtest" program id once GSSV defines one. |
| `SupportedInputTypes` | (TBD — default `["GameController","Touch"]` until decided) | |
| `PackageFlightingConfig` | see #4 | Per-title, optional. Only set if the playtest must run a different build than GA. |

## 3. Audience → Flight mapping (DNA groups)

**This pipeline already exists in PlayTest code.** Reused for the offering call.

```
PlaytestAudience.GmsGroupId
    │
    ▼  IGMSServiceClient.GetUserGroupAsync(sellerId, gmsGroupId)
    │
GMS Group  ──►  group.UserDnAGroupIds  (string[] — these are DNA group GUIDs)
    │
    ▼  one Flight per DNA id
OfferingV2.AuthorizationOptions.AllowedFlights = userDnAGroupIds.Select(id => new Flight {
    FlightId = id,
    FlightVariations = null,
})
```

| `PlayerAuthorizationOptions` field | Value |
|---|---|
| `AllowAllAuthenticatedUsers` | `false` (private offering) |
| `AllowedFlights` | one `Flight` per DNA group id resolved from GMS (see above) |
| `AllowedSandboxId` | `RetailSandboxId` (constant `PackageConstants.RetailSandbox`) |
| `CheckStoreEntitlements` | `false` (entitlement is via flight, not store) |
| `AllowedPlayerXuids` / `AllowedPuids` | `[]` (empty — gating is via flight) |
| `AllowedEntitlements` / `AllowedSubscriptions` | `[]` |
| `AllowedCountries` | (TBD — likely null = no restriction) |

`PlayerAuthenticationOptions` — needs a default playtest-specific value (TBD with team; presumably the standard player auth options used by other private offerings).

**Reuse strategy**: lift `ResolveUserDnaGroupIds` from `PlaytestBusinessLogic.cs:1054-1123` (or extract to a shared helper) and call it from the new `PartnerRegistryServiceClient` wrapper so the offering-creation path uses the same code as the publish-workflow path.

**Open question for team**: should the offering keep flights in sync as audiences change, or is it create-once / immutable? The project doc P0.3 says "When the playtest config changes the access (DNA group) for the private offering should update" — so PlayTest must call `PUT /v1/offerings/{id}` with refreshed AuthorizationOptions on every audience change.

## 4. Package flighting (per-title, optional)

If the playtest must run a different build than the GA package, set `Title.PackageFlightingConfig`:

```csharp
title.PackageFlightingConfig = new PackageFlightingConfig
{
    FlightId         = (Id)<one of the DNA flight ids>,  // must parse as Guid
    PrincipalGroupId = Guid.Parse(<DNA group id>),       // GMS validates that this IS the DNA group
    XipFlightId      = (Id)<TBD>,                        // unknown until clarified with GSSV team
    XipVariationId   = (Id)<TBD>,                        // unknown — likely from XPackage publish
};
```

`PrincipalGroupId : Guid` strongly looks like the DNA group reference at the package level — confirm with Anthony Keller. `XipFlightId`/`XipVariationId` come from the XPackage publish side; the existing `XPackagePlaytestPublishWorkflow` already passes `FlightIds` through to `ContentVersionPublishRequest.ContentUpdateRequest.FlightIds`, so XIP IDs are likely returned from that publish step.

If a playtest uses the same package as GA (just gated audience), skip `PackageFlightingConfig`.

## 5. Streaming infra defaults (need a template / decision)

These fields have no PlayTest counterpart. PlayTest will need a config-driven "playtest offering template" in `appsettings.json` to pre-fill them, or a sister offering to copy from. Confirm sane defaults with GSSV team.

| Field | Suggested default | Why |
|---|---|---|
| `Regions` | `["EastUS", "WestUS", "WestEurope"]` (or full GSSV region list) | Project doc says regions/touch controls are stretch in xplaytest portal — start permissive |
| `TemporarilyDisabledRegions` | `[]` | |
| `AllowRegionSelection` | `true` | Lets the tester pick a region in Bayside |
| `DedicatedFQDN` | `true` | Project doc: each private offering gets `[offeringid].gssv-play-prod.xboxlive.com` |
| `DefaultAllocationPools` | `<copy from template offering>` | VM pools — must come from GSSV-defined templates |
| `SelectableSystemUpdateGroups` | `<copy from template>` | |
| `SystemUpdateGroupWeights` | `<copy from template>` | |
| `ServiceLevel` | `<copy from template>` (e.g. "Standard") | |
| `TabAccessGroup` | `<copy from template>` | |
| `DefaultAllowedServerTypes` | `["XboxSeriesX", "XboxSeriesS"]` (or per-platform) | |
| `SelectableServerTypes` | same | |
| `DefaultSupportedInputTypes` | `["GameController", "Touch"]` | |
| `ServerIdleWarningTimeInSeconds` | `300` | |
| `SessionIdleTimeInSeconds` | `600` | |
| `ResourceHoldTimeoutInSeconds` | `60` | |
| `ResourceAllocationTimeoutInSeconds` | `60` | |
| `PerAccessLevelLimitOverrides` | `[]` (defaults from registry) | Time limits per access level |
| `TimeLimitOverrides` | `[]` | Per-program time limits |
| `EnableXboxRegionFallback` | `true` | |
| `AllowCrossOfferingResume` | `false` (playtest is isolated) | |
| `AllowRigAccessoryConnections` | `false` | |
| `MaxParallelSessionsPerCore` | (default) | |
| `CanUploadGameFiles` | `false` | |
| `ClientCloudSettings` | `null` | |
| `FallbackToGA` | `false` — **important**: playtest must NOT silently fall back to GA build | |

**Decision needed**: should there be a `PlaytestOfferingTemplate` offering in Partner Registry that PlayTest reads at startup and clones? Or a static JSON template in PlayTest config? The clone-from-template approach is more resilient to GSSV-side changes.

---

## 6. Package URI / Build ingestion (P0.4)

When a new build is published, PlayTest needs to call **GSSV ContentIngestion** so the build becomes streamable. This is the second integration.

### Data we have on each playtest package (from PlayTest, ready to pass)

From `PlaytestPackageResponse` + `PublishedPlaytestPackageEntity` + `PlaytestServicingContentIdEntity`:

| Field | Type | Source |
|---|---|---|
| `PackageId` | string (XPackage ID — typically a Guid) | PlayTest DB |
| `MarketGroupId` | string | PlayTest DB |
| `CMXPackageSetId` | string | PlayTest DB |
| `ServicingContentId` | Guid (as string) | PlayTest DB, joined from `PlaytestServicingContentIdEntity` |
| `ProductBigId` | string | playtest.PartnerCenterProductId |
| `PublisherId` | string | playtest.SellerId |
| `Sandbox` | string | constant `PackageConstants.RetailSandbox` |
| `FlightIds` | List<string> | from `ResolveUserDnaGroupIds` (same DNA flight ids as offering) |
| `PackageFamilyName` | string | from XProduct lookup (already done in XPackagePlaytestPublishWorkflow) |

These are also already grouped into `MarketGroupPackages` ({ MarketGroupId, PackageIds[], ServicingContentId }) by `BuildMarketGroupPackages` in `PlaytestBusinessLogic.cs:1024-1043`. Reuse that helper.

### GSSV ContentIngestion contract (what we have to discover from NuGet)

The client is in NuGet `Microsoft.GameStreaming.ContentIngestion.Client 1.0.2604.2902`. We've verified the following namespaces are available:

- `Microsoft.GameStreaming.ContentIngestion.Client` — `IContentIngestionClient`, `ContentIngestionClient`
- `Microsoft.GameStreaming.ContentIngestion.Contracts.External` — `WireStreamingPackage`, `PackageInfoForOfferValidation`, `StreamingPackageInfoV2`
- `Microsoft.GameStreaming.ContentIngestion.Contracts.External.Assets` — `AssetMetadata`, `AssetProperties`, `AssetInfoV2`
- `Microsoft.GameStreaming.ContentIngestion.Contracts.External.Query` — `PackageSearchQuery`, `IngestionEntityFilter`

`WireStreamingPackage` (verified shape from mock):
```csharp
new WireStreamingPackage
{
    Id = Guid.NewGuid(),                        // GSSV-internal package id
    GameMetadata = new AssetMetadata(
        assetId: Guid.NewGuid(),
        sourceId: <SourceId — likely PackageId>,
        markets: ["US"],
        platforms: ["GEN9"],
        new AssetProperties
        {
            ProductId = <PartnerCenterProductId>,
            NominalMarket = "US",
        }),
};
```

So a `WireStreamingPackage` has `Id`, `GameMetadata.AssetId`, `GameMetadata.SourceId`, `Markets`, `Platforms`, `Properties.ProductId`, `Properties.NominalMarket`, `Assets[]`.

`PackageInfoForOfferValidation` has `PackageId : Guid`, `SourceId : Id`, `MainGameProperties : Dictionary<string,object>`, `Markets`.

### Building the package URI / request to ContentIngestion

> ⚠️ **The exact `IContentIngestionClient.IngestPackage*` method signature is in the NuGet assembly** — pull it locally to inspect. Don't ship before verifying.

Based on the surrounding contracts (Assets, AssetMetadata, AssetProperties, Markets, Platforms), the ingestion request almost certainly takes a form like:

```csharp
await contentIngestionClient.IngestPackageAsync(new IngestPackageRequest
{
    SourceId = packageId,                  // the XPackage PackageId
    PartnerId = sellerId,                  // PublisherId
    ProductId = partnerCenterProductId,    // BigId
    Markets = marketGroup.Markets,         // resolve from MarketGroupId → markets
    Platforms = ["GEN9"],                  // or per-package
    Assets = [
        new AssetInfoV2 {
            AssetId = <CMXPackageSetId-derived or XPackage asset id>,
            ...
        },
    ],
    Sandbox = "RETAIL",
    FlightIds = flightIds,                 // from ResolveUserDnaGroupIds
}, cancellationToken);
```

But the actual request type name, properties, and required fields **must be confirmed by inspecting the NuGet** before implementation.

### How to get the contract

1. Pull the NuGet package:
   ```
   nuget install Microsoft.GameStreaming.ContentIngestion.Client -Version 1.0.2604.2902 -Source <internal-feed>
   ```
2. Decompile `Microsoft.GameStreaming.ContentIngestion.Client.dll` with ILSpy / dotPeek and read `IContentIngestionClient`.
3. Cross-reference with another internal service that already calls `IngestPackageAsync` (search the Microsoft codebase org-wide for `IContentIngestionClient` usages outside `services.partnerregistry`).
4. Confirm with Anthony Keller / Timi Bolaji (GSSV team).

### Trigger point in PlayTest

After `XPackagePlaytestPublishWorkflow` completes, the `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` in PlayTest core gets the Service Bus event. The new branch:

```csharp
if (status == Success && playtest.CloudStreamingEnabled)
{
    foreach (var marketGroup in BuildMarketGroupPackages(publishedPlaytest, ...))
    {
        foreach (var packageId in marketGroup.PackageIds)
        {
            await _contentIngestionClient.IngestPackageAsync(
                BuildIngestRequest(packageId, marketGroup, publishedPlaytest, flightIds),
                ct);
        }
    }
}
await playtestBusinessLogic.UpdatePlaytestStatusAsync(...);
```

---

## 7. Summary: the integration touch points

| Trigger | PlayTest action | GSSV API call |
|---|---|---|
| Playtest **created** with `CloudStreamingEnabled = true` | Build OfferingV2 + Title (this doc, sections 1-5) | `PUT /v1/offerings/{id}` then `PUT /v1/offerings/{id}/titles/{titleId}?partnerid=...` |
| Playtest **audience changes** | Re-resolve DNA flights, update `AllowedFlights` | `GET /v1/offerings/{id}` → mutate → `PUT /v1/offerings/{id}` |
| Playtest **end date changes** | Update `ExpirationTime` on offering + title | `PUT /v1/offerings/{id}` (+ title PUT if changed) |
| New **build published** | Call GSSV ContentIngestion per package (section 6) | NuGet `IContentIngestionClient.IngestPackage*` — exact route in NuGet |
| Playtest **deleted** | Delete the offering | `DELETE /v1/offerings/{id}` (and DELETE titles first if Partner Registry doesn't cascade — verify) |
| Playtest **suspended** (stretch) | Toggle `Title.IsEnabled = false` | `PUT /v1/offerings/{id}/titles/{titleId}` with updated `IsEnabled` |

## 8. New fields that need to be added to PlayTest

To support all the above, the playtest contract needs at minimum:

| Field | Type | Purpose |
|---|---|---|
| `CloudStreamingEnabled` | `bool` | Toggle for "this playtest is streamable" |
| `OfferingId` | `string?` | Set by PlayTest after the GSSV PUT succeeds. Used for subsequent updates/delete. |
| `TitleId` | `string?` | Set by PlayTest after title PUT. Used for subsequent title operations. |
| `OfferingStatus` | `enum` (NotCreated / Pending / Active / Failed / Deleting / Deleted) | For operator visibility and idempotency of the create flow. |
| `OfferingFailureReason` | `string?` | When status = Failed, why. |

These need migration on `PlaytestEntity` (SQL+EF), plus protobuf changes in the Shared contracts.
