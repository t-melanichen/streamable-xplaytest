# Research: PlayTest in Xbox.Xbet.Service

## Locations
- Repo root: `C:\Users\t-melanichen\source\repos\Xbox.Xbet.Service`
- PlayTest core service: `src/PlayTest/PlayTest`
- PlayTest front door: `src/PlayTest/PlayTestFD`
- PlayTest tests: `src/PlayTest/PlayTest.Tests`, `src/PlayTest/PlayTestFD.Tests`

## What It Does Today
**PlayTest core** is a backend service for CRUD on playtests + per-playtest data: audiences, packages, statuses, deletion lifecycle. It exposes operations like `CreatePlaytestAsync`, `UpdatePlaytestAsync`, `UpdatePlaytestStatusAsync`, `FinalizePlaytestDeletionAsync`, `RegisterPlaytestPackageAsync`.

**PlayTestFD** is the Partner Center-facing proxy for playtest operations. Route prefix is `/api/{locale}/dashboard/products/{productId}/playtests`.

**ContentAccess** (also in Xbox.Xbet.Service) handles content-access decisions for playtests at runtime — i.e., "is this user allowed to play this playtest's product?" It calls GSSV upstream for streaming-related decisions.

## Persistence — Important Correction

Earlier notes incorrectly said PlayTest uses Cosmos. **It does not.**

- PlayTest uses **SQL Server + Entity Framework** with a cutover pattern.
- See `src/PlayTest/PlayTest/Database/DatabaseAccess.cs:20-144` — orchestrates read/write through SQL.
- `PlaytestSqlClient.cs` exists but is a stub today; the real path is through `DatabaseAccess`.

## Existing ServiceClient Pattern (template for new PartnerRegistry client)

The only outbound HTTP S2S service client in PlayTest today is `AgeRatingServiceClient`:

- File: `src/PlayTest/PlayTest/ServiceClients/AgeRatingService/AgeRatingServiceClient.cs:27`
- Pattern:
  - Extends `Shared.Common.ServiceClient`
  - Constructor takes `HttpClient`, `IOptionsMonitor<TConfig>`, `IS2SAuthHelper`, `ILogger`
  - Resolves S2S scope from `configuration.CurrentValue.ResourceId`
  - Method-level routes via `[UrlFormat("...")]` attribute, with `ReplaceTokens(...)` + `UrlFormatParameters` for path interpolation
  - Calls `_s2sAuthHelper.AddS2SAuthHeaderAsync(request, _s2SScope, ct)` before sending
  - Uses `SendAsync(request, ct)` and `ThrowIfHttpFailedAsync(response, ct)` from the base class
- Config in `appsettings.json` under section `AgeRatingServiceClient` (lines 69-87)

A second strong template lives in **XCloudCore**: `src/XCloudCore/XCloudCore/ServiceClients/XCloudServiceClient.cs` — same pattern.

**Action for the project**: build `PartnerRegistryServiceClient` in `src/PlayTest/PlayTest/ServiceClients/PartnerRegistryService/` using these as templates. Methods:
- `Task<OfferingV2> GetOfferingAsync(string offeringId, CancellationToken ct)`
- `Task PutOfferingAsync(string offeringId, OfferingV2 offering, CancellationToken ct)`
- `Task DeleteOfferingAsync(string offeringId, CancellationToken ct)`
- `Task PutTitleAsync(string offeringId, string titleId, string partnerId, Title title, CancellationToken ct)`

Add the new section to `appsettings.json`. Wire up in `Startup.cs` next to AgeRatingServiceClient.

## ServiceBus Topic Processors (existing — extension points)

### `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor`
- File: `src/PlayTest/PlayTest/ServiceBus/MessageProcessors/XPackagePlaytestPublishWorkflowJobStatusTopicProcessor.cs:18-73`
- `JobType => CoreXPackageJobType.XPackagePlaytestPublishWorkflow`
- Subscribes to topic `job-status` / subscription `playtest`
- **Today**: only calls `UpdatePlaytestStatusAsync`
- **For our project**: this is the extension point. When status indicates a successful publish, branch on "is this playtest cloud-streaming enabled?" and call GSSV ContentIngestion.

### `XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor`
- File: `src/PlayTest/PlayTest/ServiceBus/MessageProcessors/XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor.cs:22-83`
- **Today**: calls `UpdatePlaytestStatusAsync` + `FinalizePlaytestDeletionAsync` on Deleted status
- **For our project**: also branch on cloud-streaming enabled — call `DELETE /v1/offerings/{offeringId}` on Partner Registry.

## XPackagePlaytestPublishWorkflow — what runs upstream

File: `src/XPackage/XPackageWorkflow/XPackageWorkflow/Workflows/Playtest/XPackagePlaytestPublishWorkflow.cs`

State machine: Starting → Validation → FetchingContentIds → StartingContentSubmissionJob → PollingContentSubmissionJob → PlaytestProductCreation → SuccessCompletion (or FailedCompletion).

Already takes `FlightIds` and `MarketGroupPackages` as input parameters — see `XPackageWorkflow.Shared/Workflows/PlaytestPublish/PlaytestPublishJobParameters.cs`. **This is the "xplaytest flights" the project doc refers to.** Today the workflow ends without calling GSSV ContentIngestion or Partner Registry — our project adds those.

Two reasonable places to add the GSSV ingestion call:
1. **A new step inside the workflow** between `PlaytestProductCreation` and `SuccessCompletion`. Pros: atomic; can retry; cleaner failure mode. Cons: requires changes to the workflow in `Xbox.Xbet.Service/src/XPackage/...`.
2. **Inside `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor`** in PlayTest core (downstream of the existing completion event). Pros: lighter change, isolated to PlayTest. Cons: outside the workflow's retry semantics; if PlayTest is down the call is lost (although ServiceBus retries would help).

Recommendation: start with option 2 (smaller blast radius, easier to ship as P0), and migrate to option 1 if reliability requires it.

## PlaytestResponse fields (data already in the contract)

```
PlaytestId, PartnerCenterProductId, SellerId, Name, StartDate, EndDate,
CreatedAt, Status, Type, Description, GameArtUrl,
PlaytestXPackagePublishJobId, PlaytestXProductInfo,
Audiences, Packages, ServicingContentIds, StatusDetail,
PlaytestXPackageDeleteJobId
```

For streaming:
- `PartnerCenterProductId` → maps to `Title.ProductId` in Partner Registry
- `EndDate` → can set `OfferingV2.ExpirationTime`
- `Audiences` → maps to `PlayerAuthorizationOptions.AllowedFlights` (mechanism TBD)
- `Name`, `Description`, `GameArtUrl` → for the shareable link preview (consumed by Bayside)

A new field — `CloudStreamingEnabled : bool` (or equivalent) — almost certainly needs to be added to `PlaytestCreateRequest`/`PlaytestUpdateRequest` so creators can opt in.

## Other Xbox.Xbet.Service services to know about

| Directory | What it likely does |
|---|---|
| `src/XPackage/XPackageFD` + `XPackageIngestionCore` | MSIXVC2 package ingestion (boxes/segments) — already runs today for all playtests |
| `src/XPackage/XPackageWorkflow` | Holds the `XPackagePlaytestPublishWorkflow` state machine |
| `src/XCloudCore` | Read client for xCloud's `/v1/offerings/public` and `/v2/catalog/{offeringId}`. Useful as a ServiceClient template. |
| `src/StreamingPartnerFD` | A partner-facing streaming FD — appears unrelated to the playtest publish path |
| `src/ExternalStreamingCatalog` | A streaming catalog service — worth a brief scan, may be the "/offerings" surface for clients |
| `src/ContentAccess` | Already aware of playtests via `IsPlaytestUserAuthorizedAsync` — and already calls GSSV upstream |

## No existing Partner Registry client

A grep for `PartnerRegistryClient`, `PartnerRegistry`, `partnerregistry` in `src/PlayTest/` returns zero results. **This integration is greenfield work.**
