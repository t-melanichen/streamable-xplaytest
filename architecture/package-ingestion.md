# Architecture: Package / Build Ingestion for Streaming

> **The 3-system distinction**: GSSV, ContentIngestion, and XPackage Ingestion are three different things. xPlayTest and GSSV do not talk today — this is net-new integration.

---

## The three systems

| # | System | Lives in | Owned by | What it does |
|---|---|---|---|---|
| 1 | **XPackage Ingestion** (existing) | `Xbox.Xbet.Service/src/XPackage/XPackageFD` + `XPackageIngestionCore` + `XPackageWorkflow` | Xbox build/package team | Takes raw MSIXVC2 build → segments into boxes → tracks status. **Already runs today for every playtest publish via `XPackagePlaytestPublishWorkflow`.** |
| 2 | **GSSV ContentIngestion** (NEW integration) | External — `https://*.gssv-*.xboxlive.com/api/contentingestion/`. Client via NuGet `Microsoft.GameStreaming.ContentIngestion.Client 1.0.2604.2902`. | GSSV team | The intern calls this to register an XPackage-ingested build with the xCloud streaming infrastructure. **GSSV does not kick this off itself — PlayTest has to.** |
| 3 | **GSSV Offering Registry** = `services.partnerregistry` (NEW integration) | `services.partnerregistry/src/Product/PartnerRegistryService` (namespace `Microsoft.GameStreaming.Partners.Contracts`) | GSSV team | The offering CRUD API. PlayTest calls this to create / update / delete the private offering and its title. |

System (1) is the existing pipeline. Systems (2) and (3) are the two new integrations the intern owns. Both (2) and (3) are GSSV-owned services — they just live in different repos and have different APIs.

---

## What's already in code that we leverage

`PlaytestBusinessLogic.cs:868` already builds `MarketGroupPackages` for the publish workflow:

```csharp
var marketGroupPackages = publishedPlaytestEntity.PublishedPlaytestPackages
    .Where(p => !string.IsNullOrWhiteSpace(p.MarketGroupId))
    .GroupBy(p => p.MarketGroupId, StringComparer.OrdinalIgnoreCase)
    .Select(g => new MarketGroupPackages
    {
        MarketGroupId = g.Key,
        PackageIds    = g.Select(x => x.PackageId).Distinct().ToList(),
        ServicingContentId = servicingContentIds
            .FirstOrDefault(s => s.MarketGroupId == g.Key)?.ServicingContentId.ToString(),
    })
    .ToList();
```

This same helper builds the data we pass to GSSV ContentIngestion. Reuse via the existing `BuildMarketGroupPackages` helper.

## Per-build flow (what we add)

```
Creator uploads new build
        │
        ▼
[XPackage] XPackagePlaytestPublishWorkflow
   (existing — Validate → FetchContentIds → ContentSubmission → PlaytestProductCreation → Success)
        │
        ▼ ServiceBus topic "job-status"
        │
[PlayTest core] XPackagePlaytestPublishWorkflowJobStatusTopicProcessor
        │
        ▼ Today: only UpdatePlaytestStatusAsync
        │
        ▼ NEW: branch on playtest.CloudStreamingEnabled
        │
        ├── NO  → existing behavior (status update only)
        │
        └── YES → for each (marketGroup, packageId) in BuildMarketGroupPackages(...):
                      await contentIngestionClient.IngestPackageAsync(BuildIngestRequest(
                          packageId, marketGroup, playtest, flightIds), ct);
                  → update playtest status
```

The offering itself is **not** touched on each build — it was already created at playtest-create time and points at `ProductId`. GSSV picks the most recent ingested build for that ProductId at session time.

## Data PlayTest has available to pass to GSSV ContentIngestion

| Field | Type | Source |
|---|---|---|
| `PackageId` | string (XPackage ID, typically a Guid) | `PlaytestPackageEntity.PackageId` |
| `MarketGroupId` | string | `PlaytestPackageEntity.MarketGroupId` |
| `CMXPackageSetId` | string | `PlaytestPackageEntity.CMXPackageSetId` (CMX = Content Management Experience) |
| `ServicingContentId` | Guid (as string) | `PlaytestServicingContentIdEntity` joined by `MarketGroupId` |
| `ProductBigId` | string | `playtest.PartnerCenterProductId` |
| `PublisherId` | string | `playtest.SellerId` |
| `Sandbox` | string | `PackageConstants.RetailSandbox` |
| `FlightIds` | List&lt;string&gt; | `ResolveUserDnaGroupIds(sellerId, audiences)` → DNA group ids |
| `PackageFamilyName` | string | from XProduct lookup (existing path in `XPackagePlaytestPublishWorkflow`) |

## GSSV ContentIngestion contract (must verify against NuGet)

What we've verified about the NuGet from grep of services.partnerregistry usages:

- Namespace `Microsoft.GameStreaming.ContentIngestion.Client` exposes `IContentIngestionClient` + `ContentIngestionClient`
- Namespace `Microsoft.GameStreaming.ContentIngestion.Contracts.External` exposes `WireStreamingPackage`, `PackageInfoForOfferValidation`, `StreamingPackageInfoV2`
- Namespace `Microsoft.GameStreaming.ContentIngestion.Contracts.External.Assets` exposes `AssetMetadata`, `AssetProperties`, `AssetInfoV2`
- Namespace `Microsoft.GameStreaming.ContentIngestion.Contracts.External.Query` exposes `PackageSearchQuery`, `IngestionEntityFilter`
- `WireStreamingPackage` shape (from mock at `MockContentResolutionClient.cs:60-69`):
  ```csharp
  new WireStreamingPackage
  {
      Id = Guid.NewGuid(),
      GameMetadata = new AssetMetadata(
          assetId: Guid.NewGuid(),
          sourceId: <Id>,             // probably the XPackage PackageId or a derived value
          markets: ["US"],
          platforms: ["GEN9"],
          new AssetProperties
          {
              ProductId = <PartnerCenterProductId>,
              NominalMarket = "US",
          }),
  };
  ```
- `PackageInfoForOfferValidation` has `PackageId : Guid`, `SourceId : Id`, `MainGameProperties : Dictionary<string,object>` (with constant key `MetadataPropertyNames.MsCatalogProductId`), `Markets`.
- Dev base URI: `https://americas.gssv-dev-test.xboxlive.com/api/contentingestion/`
- DI registration uses `services.AddGSHttpClient<IContentIngestionClient, ContentIngestionClient>()` (extension method from the same NuGet that wires up S2S auth)

What we **don't** have from local code: the exact `IngestPackage*` method name(s), request type, and properties. That's in the NuGet assembly.

## How the intern gets the exact contract

1. **Pull the NuGet locally** — restore `services.partnerregistry` once, then look in `%USERPROFILE%\.nuget\packages\microsoft.gamestreaming.contentingestion.client\1.0.2604.2902\` for the assemblies. Open `Microsoft.GameStreaming.ContentIngestion.Client.dll` in **ILSpy** (or `dotnet-symbol` + dotPeek) and read the public interface for `IContentIngestionClient`.
2. **Find another caller** — search the Microsoft codebase org-wide for `IContentIngestionClient.Ingest` to find a service that already calls it. Copy that pattern.
3. **Sync with GSSV team** — bring concrete questions:
   - Exact route(s) under `/api/contentingestion/`?
   - Required vs optional request properties?
   - Sync (single response) vs async (202 + job ID + poll)?
   - What S2S permission does PlayTest's AAD app need?
   - Is there a dev sandbox we can hit (e.g. `americas.gssv-dev-test`)?
   - When a Product has multiple ingested builds, how does GSSV pick which to stream? (Latest? Latest matching the flight?)
   - Does ContentIngestion need to be told about flights, or is that purely on the offering's `AllowedFlights`?

## Trigger point in PlayTest (the new code)

In `src/PlayTest/PlayTest/ServiceBus/MessageProcessors/XPackagePlaytestPublishWorkflowJobStatusTopicProcessor.cs:18-73`, after the existing `UpdatePlaytestStatusAsync` call:

```csharp
if (statusMessage.Status == JobStatus.Success)
{
    var playtest = await _playtestBusinessLogic.GetPlaytestAsync(...);
    if (playtest.CloudStreamingEnabled)
    {
        var packageGroups = BuildMarketGroupPackages(playtest, servicingContentIds);
        var flightIds = await _audienceFlightResolver.ResolveAsync(playtest.SellerId, playtest.PublishedPlaytestAudiences, ct);

        foreach (var marketGroup in packageGroups)
        {
            foreach (var packageId in marketGroup.PackageIds)
            {
                await _contentIngestionClient.IngestPackageAsync(
                    BuildIngestRequest(packageId, marketGroup, playtest, flightIds),
                    ct);
            }
        }
    }
}

await playtestBusinessLogic.UpdatePlaytestStatusAsync(...);
```

`BuildIngestRequest(...)` is the function that **needs the NuGet contract** to be fully written.
