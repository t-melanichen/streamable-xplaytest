# Architecture: Package / Build Ingestion for Streaming

> **The 3-system distinction**: GSSV, ContentIngestion, and XPackage Ingestion are three different things. xPlayTest and GSSV do not talk today — this is net-new integration.

---

## The three systems

| # | System | Lives in | Owned by | What it does |
|---|---|---|---|---|
| 1 | **XPackage Ingestion** (existing) | `Xbox.Xbet.Service/src/XPackage/XPackageFD` + `XPackageIngestionCore` + `XPackageWorkflow` | Xbox build/package team | Takes raw MSIXVC2 build → segments into boxes → tracks status. **Already runs today for every playtest publish via `XPackagePlaytestPublishWorkflow`.** |
| 2 | **GSSV ContentIngestion** (NEW integration) | External GSSV service. Source: `Xbox.Streaming/_git/services.contentingestion`. Client via NuGet `Microsoft.GameStreaming.ContentIngestion.Client 1.0.2604.2902`. Host: `https://*.gssv-*.xboxlive.com/api/contentingestion/` (proxied through **Dev API** for dev, **Sage** for prod S2S). | GSSV team (Jack Heuberger built TitleIngestion as his own intern project) | Workflow-based ingestion. Multiple job types (`ProductIngestion`, `TitleIngestion`, …). **TitleIngestion is what we kick off** — it wraps a child `ProductIngestion` job and emits `StreamingPackageIds`. **GSSV does not kick this off itself — PlayTest has to.** Per Jack: *"we just need a Big ID"*. |
| 3 | **GSSV Offering Registry** = `services.partnerregistry` (NEW integration) | `services.partnerregistry/src/Product/PartnerRegistryService` (namespace `Microsoft.GameStreaming.Partners.Contracts`). Same proxy story — Dev API for dev, Sage for prod S2S. | GSSV team | The offering CRUD API. PlayTest calls this to create / update / delete the private offering and its title. **All writes today flow through ADO PRs — see "Rubber-stamp PR blocker" below.** |

System (1) is the existing pipeline. Systems (2) and (3) are the two new integrations the intern owns. Both (2) and (3) are GSSV-owned services — they just live in different repos and have different APIs.

> **Access layer**: PlayTest will never hit `gssv-*.xboxlive.com` directly. Dev/test traffic goes through **Dev API** (developer reverse proxy) — see `architecture/service-map.md` for the access-layer diagram. Prod S2S traffic goes through **Sage** (Service API Gateway), which has per-(caller-app-id, method, route) allowlists. Jack is registering PlayTest in Sage and adding Melanie to the GSAM Services group for Dev API.

---

## ⚠️ Rubber-stamp PR blocker (per Jack, 2026-05-21) — P0

Every write to Partner Registry creates an Azure DevOps PR. In **prod**, SFI forbids self-approval and PlayTest's S2S identity gets forwarded as the requester user, so PlayTest cannot auto-merge its own PRs. **This blocks every playtest-with-streaming, not just initial setup.** Without a workaround the new-playtest → streamable-link latency is "however long until a human approves the PR."

**Working solution** (Jack + Timi are designing): a separate `playtest` branch in Partner Registry alongside `main` with relaxed protections; special client methods that write directly to that branch without a PR; GSSV services reference the `playtest` branch in playtest-specific cases. PlayTest is a net consumer here — we don't implement the workaround, we just call whatever client methods Jack ships.

**Demo fallback** (until the workaround lands): a human manually approves the PR to move the playtest along.

---

## ⚠️ TitleIngestion → Offering linking is GSSV-side work (per Jack, 2026-05-21)

Current `TitleIngestion` writes a title into a **TitleCollection** (e.g. `TESTXTESTING`, `ContentValidation`). For our project, we need the ingested title to land in an **Offering**. Per Jack: *"we would also need to expand this to work for offerings, but that's pretty easy to do."*

**This is GSSV-side work** — not PlayTest's lift. The shape of the extension (new `OfferingId` field on `JobParameters`? new job type?) is open question §5g.

> **Implication for plan**: PlayTest implementation may be **blocked on Jack landing this extension first**, or we have to do a two-step (TitleIngestion → manual PUT to add the StreamingPackageId to the offering's Title).

---

## Lifecycle — when each new call fires

The two new GSSV integrations happen at **different points** in the playtest lifecycle, not at the same time:

```
[Playtest CREATE  streaming=true]                      [Build PUBLISH #N]
   │                                                       │
   ├── PUT  /v1/offerings/{id}              ⟵ system (3)   ├── existing: XPackagePlaytestPublishWorkflow
   └── PUT  /v1/offerings/{id}/titles/{tid} ⟵ system (3)   └── ServiceBus → XPackagePublishWorkflowJobStatusTopicProcessor
        ↑ ordering matters: title is a                         └── NEW: submit TitleIngestion workflow ⟵ system (2)
        sub-resource, offering must exist first                       └── GSSV TitleIngestion runs → JobState.StreamingPackageIds
```

**Why the offering can be created with no build yet:** `Title` has no `BuildId` field — only `ProductId`. GSSV resolves "what build to stream" at session-create time by looking up the latest ingested build for that ProductId. So the offering is the long-lived access-policy shell; ingestions hydrate the build pool over time.

**Why TitleIngestion has to wait for a build:** `TitleIngestion.JobState.StreamingPackageIds` and `EstimatedInstallSizeInBytes` only exist if a real build went through. There's nothing to ingest until XPackage has produced one.

---

## What's already in code that we may leverage

`PlaytestBusinessLogic.cs:868` builds `MarketGroupPackages` for the existing XPackage publish workflow:

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

Whether this same shape can feed `ProductIngestion.JobParameters.assets` depends on the (still-unknown) `assets` type — see "What we still don't have" below. If the assets type is per-package, we may iterate over `PublishedPlaytestPackages` directly without the market-group grouping.

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
        └── YES → submit TitleIngestion workflow job to GSSV:
                      var jobId = await _contentIngestionClient.SubmitTitleIngestionAsync(
                          new TitleIngestion.JobParameters(
                              new ProductIngestion.JobParameters(...),   // ← shape TBD from repo
                              TitleCollection: null,
                              Expiry: playtest.EndDate),
                          ct);
                  → store jobId on the playtest for status tracking
                  → poll (or subscribe to a status topic) until JobState.StreamingPackageIds populated
                  → update playtest status
```

The **offering itself is not touched on each build** — it was already created at playtest-create time and points at `ProductId`. GSSV picks the most recent ingested build for that ProductId at session time.

It's an open question whether we have to PUT `StreamingPackageIds` back onto the `Title` after each TitleIngestion completes, or whether GSSV joins them server-side via `ProductId`. `Title.cs` has no slot for them, which suggests the latter — but confirm with Anthony.

## TitleIngestion workflow contract (verified shape)

Source: `Xbox.Streaming/_git/services.contentingestion/src/Product/ContentCatalog.Common.Contracts/Workflows/TitleIngestion.cs` (shared by Anthony 2026-05-21).

```csharp
namespace Microsoft.GameStreaming.Services.ContentCatalog.Common.Contracts.Workflows;

public static class TitleIngestion
{
    public const string JobType = "TitleIngestion";
    public static readonly string QueueName = JobType.ToLowerInvariant();

    // What we send
    public record JobParameters(
        ProductIngestion.JobParameters ProductIngestionJobParameters,
        Id? TitleCollection,
        DateTime? Expiry) : IIngestionParameters
    {
        public Id  GetLockId()       => $"{JobType}_{this.ProductIngestionJobParameters.PartnerId}_{this.ProductIngestionJobParameters.TitleId}";
        public string GetPartitionKey() => this.ProductIngestionJobParameters.GetPartitionKey();
        public string GetJobType()      => JobType;
        public string GetQueueName()    => QueueName;

        public void Validate()
        {
            this.ProductIngestionJobParameters.Validate();
            if (!string.IsNullOrWhiteSpace(this.TitleCollection))
            {
                if (this.Expiry.HasValue && this.Expiry.Value < DateTime.Now)
                    throw new ArgumentException($"The expiry date {this.Expiry.Value} is in the past.");

                CommonValidate.IsNotEmptyOrWhitespace(
                    this.ProductIngestionJobParameters.PackageNameOverride,
                    nameof(this.ProductIngestionJobParameters.PackageNameOverride));
            }
        }
    }

    // What we get back
    public class JobState
    {
        public ChildJobState?  ChildProductIngestionJob       { get; set; }
        public IList<Guid>?    StreamingPackageIds            { get; set; }   // ← the IDs we care about
        public uint?           XboxTitleId                    { get; set; }
        public string          StatusDetails                  { get; set; } = string.Empty;
        public long            EstimatedInstallSizeInBytes    { get; set; }
    }
}
```

**Things we infer from this:**
- The lock id format `TitleIngestion_{PartnerId}_{TitleId}` implies **a TitleIngestion is keyed on `(PartnerId, TitleId)`** — repeated submissions for the same `(PartnerId, TitleId)` serialize via lock. So for "creator publishes build #2", the same lock id fires and the system handles it.
- `TitleCollection` is optional. Verified `null` for playtests — `TitleCollection` is GSSV's internal catalog grouping (`"TESTXTESTING"`, `"ContentValidation"`, `"OLTEST"`, `"CERTTITLES"`).
- `Expiry` aligns nicely with `playtest.EndDate`.
- `Id` is the `Microsoft.GameStreaming.Common.Id` implicit-string type we've seen on `OfferingV2`, `Title`, etc.

## Consumption pattern from `services.contentingestion` (verified)

From the GSSV Content Portal's `BulkIngest.razor` (shared by Anthony 2026-05-21):

```csharp
using Microsoft.GameStreaming.ContentIngestion.Contracts.External.Workflow;

// build one TitleIngestion.JobParameters per title
var titleParameters = this.ProductList.Select(p =>
    ContentPortalProcessor.GetTitleIngestionParameters(
        ContentPortalProcessor.GetProductIngestionParameters(
            partnerId:    this.PartnerId,
            titleId:      p.TitleId,
            productId:    p.ProductId,           // BigId
            platform:     ServerPlatform.Xbox,
            name:         p.Name,
            description:  p.Description,
            assets:       null,                  // null = let GSSV resolve from store
            flightId:     null,                  // singular! optional
            sandboxId:    this.SearchedSandboxId),
        p.TitleCollection,                       // null for playtests
        p.ExpirationTime))
    .ToList();

await this.ContentPortalProcessor.BulkIngestTitlesAsync(titleParameters, createdBy);
```

And `TitleId` is **generated**, not made up:

```csharp
var name    = ContentEntityIdentifiers.GenerateNeutralStreamingPackageName(productTitle);
var titleId = ContentEntityIdentifiers.GenerateTitleId(name, ServerPlatform.Xbox);
```

These helpers live in `Microsoft.GameStreaming.Services.Common.Content.Ids` / `Microsoft.GameStreaming.Services.Common.Content.StoreClient.Contracts.Schema` — both shipped in the GSSV common NuGet(s). **Use these helpers — do not hand-roll TitleIds.**

### So `ProductIngestion.JobParameters` has (at least) these constructor params:

| Param | Type | PlayTest source |
|---|---|---|
| `partnerId` | `Id` | `playtest.SellerId` |
| `titleId` | `Id` | `ContentEntityIdentifiers.GenerateTitleId(name, ServerPlatform.Xbox)` |
| `productId` | `Id` | `playtest.PartnerCenterProductId` (the BigId) |
| `platform` | `ServerPlatform` | `ServerPlatform.Xbox` (only Xbox supported today) |
| `name` | string | `ContentEntityIdentifiers.GenerateNeutralStreamingPackageName(playtest.Name or product title)` |
| `description` | string | `playtest.Description` (or product short description) |
| `assets` | `…?` | **null for store products — likely NOT null for playtests** (see open question below) |
| `flightId` | `Id?` | **singular** — likely `null` for playtests, with offering's `AllowedFlights` doing the gating |
| `sandboxId` | `Id` | TBD — playtest sandbox, not "RETAIL" |

## What we still don't have

- **`assets` parameter shape and whether playtests need to pass it explicitly.** The bulk-ingest path passes `assets: null` because the source is a store-published product (GSSV calls `GetBulkTitlesFromStoreAsync` to resolve). **Playtest builds aren't store-published** — they live in XPackage. So we likely need to pass `assets` with the XPackage / CMX info. **This is the sharpest remaining unknown.**
- **`sandboxId` for playtests.** The UI takes a sandbox ID per ingestion. Playtests have their own sandbox-allowlist concept; need to know what GSSV expects here.
- **`flightId: null` vs per-flight ingestion.** Singular `flightId` means either "one ingestion per flight" or "leave null and let the offering do the gating". The bulk-store path passes null. **Likely answer: null is correct for playtests** because `OfferingV2.AuthorizationOptions.AllowedFlights` is where gating happens — but confirm.
- **The submission entry point for an S2S caller.** `BulkIngestTitlesAsync(IReadOnlyCollection<TitleIngestion.JobParameters>, createdBy)` is on a Blazor-side `IContentPortalProcessor` that wraps the actual client. From an S2S service (PlayTest) we likely call the underlying `IContentIngestionClient` method directly — name TBD.
- **Job-status delivery model.** Does the client surface a `WaitAsync` / completion future? A polled `GetJobStateAsync(jobId)`? A ServiceBus topic we subscribe to? Affects fire-and-forget vs wait-on-completion.

## How the intern gets the rest of the contract

1. **Ask Anthony for read access** to `Xbox.Streaming/_git/services.contentingestion`, or have him paste the contents of:
   - `src/Product/ContentCatalog.Common.Contracts/Workflows/ProductIngestion.cs` (to see the `assets` parameter type)
   - `src/Product/ContentCatalog.Common.Contracts/Workflows/Abstractions/*.cs` (for `IIngestionParameters`, `ChildJobState`)
   - The submission-side interface (`IContentIngestionClient`)
   - The `IContentPortalProcessor` implementation showing how `GetProductIngestionParameters` builds the actual JobParameters constructor call
2. **Pull the NuGet locally** as a fallback — restore `services.partnerregistry` once, then look in `%USERPROFILE%\.nuget\packages\microsoft.gamestreaming.contentingestion.client\1.0.2604.2902\` and open the assemblies in **ILSpy** (or dotPeek) to confirm the client method names + request types.
3. **Find another S2S caller** — search the Microsoft codebase org-wide for `IContentIngestionClient` and `TitleIngestion.JobParameters` usages (outside Blazor UIs) to find a service that already submits these jobs and copy its pattern.

## Trigger point in PlayTest (the new code)

In `src/PlayTest/PlayTest/ServiceBus/MessageProcessors/XPackagePlaytestPublishWorkflowJobStatusTopicProcessor.cs:18-73`, after the existing `UpdatePlaytestStatusAsync` call:

```csharp
if (statusMessage.Status == JobStatus.Success)
{
    var playtest = await _playtestBusinessLogic.GetPlaytestAsync(...);
    if (playtest.CloudStreamingEnabled)
    {
        var name        = ContentEntityIdentifiers.GenerateNeutralStreamingPackageName(playtest.Name);
        var titleId     = ContentEntityIdentifiers.GenerateTitleId(name, ServerPlatform.Xbox);
        var description = playtest.Description ?? string.Empty;

        var productParams = new ProductIngestion.JobParameters(
            partnerId:    playtest.SellerId,
            titleId:      titleId,
            productId:    playtest.PartnerCenterProductId,
            platform:     ServerPlatform.Xbox,
            name:         name,
            description:  description,
            assets:       BuildPlaytestAssets(playtest, servicingContentIds),  // ← shape TBD (P0 blocker)
            flightId:     null,                                                // gating lives on the offering
            sandboxId:    playtest.SandboxId);                                 // TBD: which sandbox?

        var titleParams = new TitleIngestion.JobParameters(
            ProductIngestionJobParameters: productParams,
            TitleCollection: null,                                             // playtests don't belong to a collection
            Expiry:          playtest.EndDate);                                // mirrors offering ExpirationTime

        var jobId = await _contentIngestionClient.SubmitTitleIngestionAsync(titleParams, ct);  // method name TBD
        // optional: persist jobId on the playtest for ops visibility
    }
}

await playtestBusinessLogic.UpdatePlaytestStatusAsync(...);
```

`BuildPlaytestAssets(...)` is the function that **needs the `assets` parameter type** from `ProductIngestion.cs` to be fully written. Today's bulk-ingest UI passes `assets: null` and lets GSSV resolve from the public store — that path won't work for playtests because playtest builds aren't store-published. See "What we still don't have" above.
