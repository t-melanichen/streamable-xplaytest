# Design Doc: Instantly Shareable Playtest — Cloud Streaming for Private Playtests

**Status**: Draft v0.2 (Week 1-2 research pass)
**Author**: Melanie Chen (Xbox intern, Juno team, Summer 2026)
**Manager**: Brian Bowman · **Onboarding Buddy**: Aditya Toney · **Mentor**: Emma Park · **Admin**: Jessie Masih
**Feature Team Leaders**: David Kushmerick + Bec Lyons (jointly)
**Engineering Experts**: Anthony Keller, Timi Bolaji, Ashton Summer, Chuy Galvan
**UR Champ**: Ellery Charlson
**xCloud collaborators** (discovered via calls, not in PDF): Jack Heuberger (built TitleIngestion), David Retterath (committed PM + Dev)

---

## 1. Problem Statement

Today, xplaytest creators can invite testers to download and install pre-release builds via Microsoft Store. This requires a long download, install, and often a dev kit — high friction for casual testers and a poor fit for short-window playtests. **A secondary pain point** (raised by David Kushmerick on 2026-05-21): if a tester already has the retail copy of the same game installed, installing a playtest build of the same product can collide / require uninstall.

**Goal**: let creators flip a "cloud streaming" toggle on a playtest, then share a single URL that lets authorized testers stream the playtest build in their browser via xbox.com/play — no install, no console.

**Why streaming wins for playtests**:
1. **Instant access** — no multi-GB download wait; clicking the link starts the game in seconds
2. **No retail-install conflict** — testers keep their retail copy installed and stream a different build of the same product side-by-side
3. **No dev kit required** — opens playtesting to a much wider, more casual audience

**Target ship date**: end of June / early July 2026 public preview (per Retterath, 2026-05-21).

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

## 10. The TitleIngestion call (build publish)

When a new build is published for a streaming-enabled playtest, PlayTest submits a **TitleIngestion** workflow job to GSSV. Verified contract from `services.contentingestion/.../Workflows/TitleIngestion.cs` and the consumption pattern in `services.contentingestion`'s `BulkIngest.razor` (Anthony, 2026-05-21). See [architecture/package-ingestion.md](../architecture/package-ingestion.md) and [design/field-mapping.md §6](field-mapping.md) for the full mapping.

```csharp
public async Task IngestPlaytestBuildAsync(PublishedPlaytestEntity playtest, IList<PlaytestServicingContentIdEntity> scids, CancellationToken ct)
{
    if (!playtest.CloudStreamingEnabled) return;

    var name        = ContentEntityIdentifiers.GenerateNeutralStreamingPackageName(playtest.PlaytestName);
    var titleId     = ContentEntityIdentifiers.GenerateTitleId(name, ServerPlatform.Xbox);

    var productParams = new ProductIngestion.JobParameters(
        partnerId:    playtest.SellerId,
        titleId:      titleId,
        productId:    playtest.PartnerCenterProductId,
        platform:     ServerPlatform.Xbox,
        name:         name,
        description:  playtest.Description ?? string.Empty,
        assets:       BuildPlaytestAssets(playtest, scids),  // ← shape TBD: see open-questions
        flightId:     null,                                  // gating lives on the offering's AllowedFlights
        sandboxId:    playtest.SandboxId);                   // TBD: playtest sandbox id

    var titleParams = new TitleIngestion.JobParameters(
        ProductIngestionJobParameters: productParams,
        TitleCollection: null,                               // playtests don't belong to a TitleCollection
        Expiry:          playtest.EndDate);                  // mirrors offering ExpirationTime

    var jobId = await _contentIngestionClient.SubmitTitleIngestionAsync(titleParams, ct);   // method name TBD
    // optional: persist jobId on the playtest for ops visibility / status polling
}
```

`BuildPlaytestAssets(...)` is the only remaining unknown. The bulk-ingest path in the Content Portal passes `assets: null` and lets GSSV resolve from the public store — that path won't work for playtests because playtest builds aren't store-published. **P0 blocker** to resolve with Anthony.

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

> **Official 12-week intern timeline** (per Brian's project plan PDF — see [`research/project-spec.md`](../research/project-spec.md)). **Public Preview** (per Retterath, 2026-05-21) aligns with the **Midpoint Connect at end of Core Service Work (Week 6-7, ~6/30)** — PP is the project **midpoint**, not the end. ~5 more weeks of frontend + polish follow.

| Weeks | Date range | Phase | Key gates |
|---|---|---|---|
| **1-2** | 5/19 – 5/29 | Onboarding + Welcome (Agency / Copilot CLI ramp-up) | First Connect — **6/2** |
| **3-4** | 6/2 – 6/12 | Design Doc + Prototyping / Manual Testing | Reviewed design doc; engineering estimates locked |
| **5-7** | 6/15 – 7/3 | **Core Service Work** — GSSV + xplaytest + CAS | 🎯 **Public Preview** + Midpoint Connect — **6/30** |
| **8-10** | 7/6 – 7/24 | **Frontend Work** — Bayside player + xplaytest portal (React, Tailwind) | Final Connect — **7/28** |
| **11-12** | 7/27 – 8/7 | Documentation, Metrics, Polish — dashboards, monitors, TSGs | Final Project Presentation (date TBD by UR Champ) |

See [`design/execution-plan.md`](execution-plan.md) for the full week-by-week breakdown.

**Critical-path risks for the deadline**:
- Jack+Timi rubber-stamp PR workaround — if it slips, demo-only fallback (manual approval) blocks public preview
- Jack's TitleIngestion → Offering extension — without it, no E2E happens
- Retterath's multi-version GSSV work — scope-down to single-version-per-playtest if it can't land by Week 6

## 14. Open questions (for team — pulled into [open-questions-for-team.md](open-questions-for-team.md))

**Status**: Updated after Jack Heuberger call + Kushmerick/Retterath/Park onboarding sync (both 2026-05-21). See `open-questions-for-team.md` for the full list (questions 5f-5i).

Top blockers (ordered by deadline impact):
1. **Multi-version support for playtests** (NEW, Retterath) — today GSSV resolves "what build to stream" live from X product as latest-ingested-per-productId. Playtests may need multiple concurrent versions of a product. **Cloud-side work owned by Retterath's team.** v1 fallback: single-version-per-playtest scope-down. See §5i.
2. **Rubber-stamp PR problem in prod** (Kushmerick + Jack) — every Partner Registry write creates an ADO PR; SFI forbids self-approval. Jack + Timi designing a `playtest`-branch workaround. **PlayTest waits.** Demo fallback: manual approval. See §5.
3. **TitleIngestion → Offering linking is GSSV-side work** (Jack) — current TitleIngestion writes to a `TitleCollection`. Extension to write to an Offering is "pretty easy" but on Jack's todo. May block E2E. See §5g.
4. **S2S auth onboarding** (Jack) — In progress: GSAM Services group + Sage route + caller-app-id registration + LinkPad scaffold. Brian helping with the PlayTest service identity app id. See §5e.
5. **`ProductIngestion.JobParameters.assets`** (Jack) — likely `null` ("just need a Big ID"). Open sub-question: does it work for **unpublished** playtest BigIds? See §5a.
6. **`sandboxId` for playtest TitleIngestion** — still TBD. See §5b.
7. **One offering per playtest, or shared offering with per-playtest titles?** — design decision for review. See §5f.
8. **"xCloud distribution Services"** (Kushmerick's term) vs **Content Ingestion** — same thing or downstream? Confirm with Retterath/Jack.
9. **Streaming infra defaults** — what `Regions` / `DefaultAllocationPools` / `ServiceLevel` should a playtest offering have?
10. **Where does the xplaytest portal frontend live?** — not in any of the 3 local repos
11. **`PackageFlightingConfig` semantics** — is `PrincipalGroupId : Guid` the same as a DNA group id?
12. **Seller-PartnerId resolution** — is `playtest.SellerId` directly usable as `OfferingV2.PartnerId`?
