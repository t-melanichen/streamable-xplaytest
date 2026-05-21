# Design Doc: Instantly Shareable Playtest — Cloud Streaming for Private Playtests

**Status**: Draft v0.2 (Week 1-2 research pass)
**Author**: Melanie Chen (Xbox intern, Juno team, Summer 2026)
**Manager**: Brian Bowman · **Mentor**: Emma Park · **PM**: Bec Lyons
**Engineering experts**: Anthony Keller, Timi Bolaji, Ashton Summer, Chuy Galvan

---

## 1. Problem Statement

Today, xplaytest creators can invite testers to download and install pre-release builds via Microsoft Store. This requires a long download, install, and often a dev kit — high friction for casual testers and a poor fit for short-window playtests.

**Goal**: let creators flip a "cloud streaming" toggle on a playtest, then share a single URL that lets authorized testers stream the playtest build in their browser via xbox.com/play — no install, no console.

## 2. P0 Scope (this design covers)

1. **Private offering creation**: when a playtest is created with cloud streaming on, PlayTest calls GSSV (`services.partnerregistry/OfferingsController`) to create a private `OfferingV2` whose `AuthorizationOptions.AllowedFlights` map 1:1 to the playtest's DNA audience groups.
2. **Private offering deletion**: when the playtest is deleted, the offering is deleted.
3. **Audience updates**: when the playtest audience (DNA groups) changes, the offering's `AllowedFlights` updates within seconds.
4. **Build ingestion**: when a new build is published for a cloud-enabled playtest, PlayTest calls GSSV ContentIngestion to register the build as streamable.
5. **Seller allowlist**: offering creation can be limited to specific sellers via PlayTest config (so this rolls out safely behind a flag).
6. **Tester experience**: clicking the shareable link in Bayside (`xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}`) authenticates, verifies flight membership at GSSV session-create time, and streams the playtest's metadata + build.

## 3. Out of Scope (P1/P2/P3)

- Stream Now buttons in Garrison / current playtest UIs
- Launch arg support
- Streaming region / touch controls toggles in xplaytest portal
- Tester feedback collection pipeline
- xplaytest "View as tester" / preview mode
- Migrating non-Juno teams to use shared private offering infrastructure
- DevApi Reader Portal filtering for playtest offerings

## 4. Architecture

```
                   ┌────────────────────────────┐
                   │  xplaytest portal frontend │  (location TBD — NOT in 3 local repos)
                   │  (Partner Center?)         │
                   └────────┬───────────────────┘
                            │ HTTPS
                            ▼
              ┌────────────────────────────────┐         ┌──────────────────────────┐
              │  PlayTestFD (front door)       │────────▶│  GMS Service             │
              │  /api/.../products/.../playtests│ (existing AddS2SAuthHeader)  │
              │  Xbox.Xbet.Service/src/PlayTestFD  │  PlayTest → GMS → DnaIds  │
              └────────┬───────────────────────┘         └──────────────────────────┘
                       │ FabricClient / S2S
                       ▼
              ┌────────────────────────────────┐
              │  PlayTest (core)               │
              │  Xbox.Xbet.Service/src/PlayTest │
              │  SQL+EF persistence            │
              │                                │
              │  ┌──────────────────────────┐  │
              │  │ NEW: PartnerRegistry    │  │   PUT/DELETE /v1/offerings, PUT /titles
              │  │ ServiceClient            │──┼─────────────────────────────────────────▶ GSSV Offering Registry
              │  └──────────────────────────┘  │                                            (services.partnerregistry,
              │  ┌──────────────────────────┐  │                                             OfferingsController)
              │  │ NEW: ContentIngestion   │  │
              │  │ Client (from NuGet)      │──┼─────────────────────────────────────────▶ GSSV ContentIngestion
              │  └──────────────────────────┘  │                                            (*.gssv-*.xboxlive.com/api/contentingestion)
              │  ┌──────────────────────────┐  │
              │  │ ServiceBus processors    │  │
              │  │ (extend existing)        │  │
              │  └──────────────────────────┘  │
              └────────┬───────────────────────┘
                       │ ServiceBus topic "job-status"
                       ▼ (existing)
              ┌────────────────────────────────┐
              │  XPackagePlaytestPublishWorkflow│   builds packages, sets FlightIds
              │  Xbox.Xbet.Service/src/XPackage │
              └────────────────────────────────┘

  Tester flow:
  Shareable link  →  xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}
                  →  Bayside (Xbox.JS/apps/play-xbox) loader
                  →  GSSV  (session-create checks flight membership)
                  →  Garrison VM  (streams the playtest build)
```

## 5. The integration touch points

Five places PlayTest core calls outward. Three are new for this project.

| # | Trigger | New code in PlayTest | External call |
|---|---|---|---|
| 1 | Playtest created with `CloudStreamingEnabled = true` | `PartnerRegistryServiceClient.PutOfferingAsync` + `PutTitleAsync` | `PUT /v1/offerings/{id}` + `PUT /v1/offerings/{id}/titles/{titleId}` on GSSV registry |
| 2 | Playtest audience updated | `PartnerRegistryServiceClient.GetOffering` + `PutOffering` with refreshed `AllowedFlights` | `GET` → mutate → `PUT /v1/offerings/{id}` |
| 3 | Playtest end date updated | `PartnerRegistryServiceClient.PutOffering` with new `ExpirationTime` | `PUT /v1/offerings/{id}` |
| 4 | New build published (ServiceBus event) | Extend `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` | `IContentIngestionClient.IngestPackage*` per (marketGroup, packageId) |
| 5 | Playtest deleted (ServiceBus event) | Extend `XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor` | `DELETE /v1/offerings/{id}` |

The field mapping for #1, #2, #3 is the central design output — see [field-mapping.md](field-mapping.md).

## 6. Detailed mapping — see separate doc

For every field on `OfferingV2`, `Title`, `Flight`, `PlayerAuthorizationOptions`, and the ContentIngestion request, the exact source field on the PlayTest side (or a default / template / TBD with team) is documented in **[design/field-mapping.md](field-mapping.md)**.

Highlights:
- DNA audience groups → `Flight` objects: **reuses the existing `ResolveUserDnaGroupIds` pipeline** (`PlaytestBusinessLogic.cs:1054`). The same GMS call that today produces `PlaytestPublishJobParameters.FlightIds` produces the offering's `AllowedFlights`.
- Title.ProductId = `playtest.PartnerCenterProductId`. Title has no BuildId — Partner Registry only references ProductIds; specific builds are picked by GSSV at session time.
- OfferingV2 streaming infra fields (`Regions`, `DefaultAllocationPools`, `SystemUpdateGroupWeights`, `ServiceLevel`, ...) come from a PlayTest-side template (config or sister offering) — values need GSSV team validation.

## 7. New fields on PlayTest contracts

| Field | Type | Where |
|---|---|---|
| `CloudStreamingEnabled` | bool | `PlaytestCreateRequest`, `PlaytestUpdateRequest`, `PlaytestResponse`, `PlaytestEntity` |
| `OfferingId` | string? | `PlaytestEntity` (set after successful PUT, used for subsequent ops) |
| `TitleId` | string? | `PlaytestEntity` |
| `OfferingStatus` | enum (NotCreated / Pending / Active / Failed / Deleting / Deleted) | `PlaytestEntity` |
| `OfferingFailureReason` | string? | `PlaytestEntity` |

These need:
- A new EF migration for the SQL schema (PlayTest uses SQL+EF cutover, not Cosmos)
- Protobuf field additions on `PlaytestResponse` etc. (incrementing the `[ProtoMember]` tags)
- Validation rules in `PlayTest/Validations/Rules/` for the new state machine

## 8. The PartnerRegistryServiceClient (new code in PlayTest)

Follows the existing `AgeRatingServiceClient` template (`src/PlayTest/PlayTest/ServiceClients/AgeRatingService/AgeRatingServiceClient.cs:27`):

```csharp
namespace PlayTest.ServiceClients.PartnerRegistryService;

public interface IPartnerRegistryServiceClient
{
    Task<OfferingV2> GetOfferingAsync(string offeringId, CancellationToken ct);
    Task PutOfferingAsync(string offeringId, OfferingV2 offering, CancellationToken ct);
    Task DeleteOfferingAsync(string offeringId, CancellationToken ct);
    Task PutTitleAsync(string offeringId, string titleId, string partnerId, Title title, CancellationToken ct);
    Task DeleteTitleAsync(string offeringId, string titleId, string partnerId, CancellationToken ct);
}

public class PartnerRegistryServiceClient : ServiceClient, IPartnerRegistryServiceClient
{
    [UrlFormat("v1/offerings/{offeringId}")]
    public Task<OfferingV2> GetOfferingAsync(...) { ... }

    [UrlFormat("v1/offerings/{offeringId}")]
    public Task PutOfferingAsync(...) { ... }   // PUT — upsert, not POST

    [UrlFormat("v1/offerings/{offeringId}/titles/{titleId}?partnerid={partnerId}")]
    public Task PutTitleAsync(...) { ... }
}
```

S2S auth uses `IS2SAuthHelper.AddS2SAuthHeaderAsync(request, _s2SScope, ct)` exactly like `AgeRatingServiceClient`. New config section in `appsettings.json`: `PartnerRegistryServiceClient` with `BaseUri` + `ResourceId`. The PlayTest AAD app must be added to `AllowedS2SAppIds` on the GSSV registry side.

## 9. The PartnerRegistry call (offering create)

```csharp
public async Task<string> CreatePlaytestOfferingAsync(PublishedPlaytestEntity playtest, CancellationToken ct)
{
    if (!playtest.CloudStreamingEnabled)
        return null;

    // 1. Resolve DNA flights (reuses existing logic — see flights-and-dna-groups.md)
    var flightIds = await _audienceFlightResolver.ResolveAsync(
        playtest.SellerId, playtest.PublishedPlaytestAudiences, ct);

    if (flightIds.Count == 0)
        throw ServiceErrorHelper.CreateValidationError(
            $"Cloud streaming playtest {playtest.PublishedPlaytestId} has no resolvable DNA flights");

    // 2. Build OfferingV2 (see field-mapping.md sections 1, 3, 5)
    var offeringId = $"xpt-{playtest.PublishedPlaytestId:N}";
    var offering = _offeringTemplateProvider.CloneTemplate();
    offering.Id = offeringId;
    offering.PartnerId = playtest.SellerId;
    offering.Name = playtest.PlaytestName;
    offering.Notes = $"playtestId={playtest.PublishedPlaytestId}; seller={playtest.SellerId}";
    offering.ExpirationTime = playtest.PlaytestEndDate;
    offering.AuthorizationOptions = new PlayerAuthorizationOptions
    {
        AllowAllAuthenticatedUsers = false,
        AllowedSandboxId = PackageConstants.RetailSandbox,
        AllowedFlights = flightIds.Select(id => new Flight { FlightId = id }).ToList(),
        CheckStoreEntitlements = false,
    };
    offering.OwnedBy = [playtest.SellerId, "xpt-service"];

    // 3. PUT offering (upsert)
    await _partnerRegistryClient.PutOfferingAsync(offeringId, offering, ct);

    // 4. Build + PUT title (see field-mapping.md section 2)
    var titleId = $"t-{playtest.PublishedPlaytestId:N}";
    var title = new Title
    {
        Id = titleId,
        PartnerId = playtest.SellerId,
        ProductId = playtest.PartnerCenterProductId,
        Platform = ResolvePlatform(playtest),
        AvailableTime = playtest.PlaytestStartDate,
        ExpirationTime = playtest.PlaytestEndDate,
        IsEnabled = true,
        AllowedFlights = flightIds.Select(id => new Flight { FlightId = id }).ToList(),
        FriendlyName = playtest.PlaytestName,
    };
    await _partnerRegistryClient.PutTitleAsync(offeringId, titleId, playtest.SellerId, title, ct);

    return offeringId;
}
```

## 10. The ContentIngestion call (build publish)

The exact request shape is gated on inspecting the NuGet — see [architecture/package-ingestion.md](../architecture/package-ingestion.md). Pseudocode:

```csharp
public async Task IngestPlaytestBuildsAsync(PublishedPlaytestEntity playtest, IList<PlaytestServicingContentIdEntity> scids, CancellationToken ct)
{
    if (!playtest.CloudStreamingEnabled) return;

    var marketGroups = BuildMarketGroupPackages(playtest, scids);            // existing helper
    var flightIds = await _audienceFlightResolver.ResolveAsync(playtest.SellerId, playtest.PublishedPlaytestAudiences, ct);

    foreach (var marketGroup in marketGroups)
    {
        foreach (var packageId in marketGroup.PackageIds)
        {
            var request = BuildIngestPackageRequest(packageId, marketGroup, playtest, flightIds);
            await _contentIngestionClient.IngestPackageAsync(request, ct);   // ← NuGet contract TBD
        }
    }
}
```

## 11. Failure modes & retries

- **GSSV registry PUT fails**: persist `OfferingStatus = Failed`, surface in PlayTestFD. Retry on next playtest update; do not auto-retry to avoid loops.
- **ContentIngestion call fails**: log + dead-letter the Service Bus message (existing PlayTest pattern). Operator can replay.
- **GMS group resolve fails partially**: today PlayTest throws on any single GMS failure (`ResolveUserDnaGroupIds:1102`). Keep that behavior — better to fail loud than ship an over-permissive offering.
- **Build is ingested into GSSV but offering creation fails later**: stale package; harmless (no offering references it). Cleanup not required for P0.
- **Offering created but build ingestion fails**: tester gets "not available" — surface in playtest UI as `Status = Awaiting Build Ingestion`.

## 12. Security considerations

- **Never** call PUT offering without a successful DNA→Flight resolve. Guard with the same validation rule as the existing publish workflow (line 897 of `PlaytestBusinessLogic.cs`).
- The shareable link **must not** leak title metadata pre-auth. Bayside's loader already redirects on `isPrivate` (`loader.server.ts:28-83`) — verify that the playtest's product info doesn't get served in the HTML <head/> before redirect.
- The PlayTest AAD app needs the minimum S2S permission on the GSSV registry (probably a custom role to be requested via the GSSV team).
- Seller allowlist (`PlaytestConfig.CloudStreamingAllowlistedSellers : string[]`) checked at offering-create time.

## 13. Rollout

| Milestone | Date | Gate |
|---|---|---|
| Week 1-2 | Sept 2026 | Onboarding + research (this doc) |
| Week 3-4 | mid-Oct | Design review with manager + GSSV team |
| Week 5-6 | end Oct | Build offering create + delete; feature-flagged off |
| Week 7-8 | mid-Nov | Build ContentIngestion call; e2e with one test seller |
| Week 9-10 | end Nov | Audience updates; widen allowlist to 2-3 sellers |
| Week 11-12 | mid-Dec | Telemetry, docs, polish; final review |

## 14. Open questions (for team — pulled into [open-questions-for-team.md](open-questions-for-team.md))

Top blockers:
1. **GSSV ContentIngestion exact request shape** — must come from NuGet inspection + GSSV team
2. **Streaming infra defaults** — what `Regions` / `DefaultAllocationPools` / `ServiceLevel` should a playtest offering have? Is there a template offering to clone?
3. **Where does the xplaytest portal frontend live?** — not in any of the 3 local repos
4. **`PackageFlightingConfig` semantics** — is `PrincipalGroupId : Guid` the same as a DNA group id?
5. **Seller-PartnerId resolution** — is `playtest.SellerId` directly usable as `OfferingV2.PartnerId`, or is there a lookup?
