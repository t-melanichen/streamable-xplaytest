# Research: services.partnerregistry

## Location
- Repo root: `C:\Users\t-melanichen\OneDrive - Microsoft\Desktop\services.partnerregistry`
- Source: `src/Product/PartnerRegistryService` (the ASP.NET service) + `src/Product/PartnerRegistryClient/Contracts` (the contract models)

## What It Does
Manages the xCloud partner-facing registry: **offerings**, **titles**, **streaming configs**, **regions**, **pools**. Changes are stored backing storage and can be synced through Azure DevOps PRs for production approval.

> ⚠️ **This repo does NOT do GSSV content ingestion**. It has a NuGet reference to `Microsoft.GameStreaming.ContentIngestion.Client 1.0.2604.2902` only for *reading* ingested package metadata (e.g., to validate that an offering's title points at a real ingested ProductId). It does not call any `IngestPackage*` methods.

## Key Controllers

| Controller | Route | Purpose |
|-----------|-------|---------|
| `OfferingsController` | `v1/offerings` | Full CRUD for offerings + titles within offerings |
| `RegistryManagementController` | `v1/registrymanagement` | PR-sync surface (`/submit`, `/s2s/submit`) — **NOT the primary creation path** |
| `StreamingConfigController` | `v1/streamingconfig` | Get/update default streaming builder config |
| `StreamingStackConfigController` | `v1/streamingstack/configs` | Streaming stack configs |
| `GamePassProductsController` | Game Pass products | Product cache + misconfigurations |

## OfferingsController endpoints (verified — file `Controllers/OfferingsController.cs:21-139`)

| Method | Route | Body | Notes |
|---|---|---|---|
| GET | `/v1/offerings` | — | Returns `IEnumerable<OfferingV2>` |
| GET | `/v1/offerings/{id}` | — | Returns `OfferingV2` |
| **PUT** | `/v1/offerings/{id}` | `OfferingV2` | **Upsert** (create or replace). Not POST. |
| DELETE | `/v1/offerings/{id}` | — | |
| GET | `/v1/offerings/{offeringid}/titles` | — | |
| PUT | `/v1/offerings/{offeringid}/titles/{titleid}?partnerid={id}` | `Title` | Single-title upsert |
| PUT | `/v1/offerings/{offeringid}/titles?partnerid={id}&pullRequestTitle={?}` | `Title[]` | Bulk title upsert |
| DELETE | `/v1/offerings/{offeringid}/titles/{titleid}?partnerid={id}` | — | |
| POST | `/v1/offerings/{offeringid}/deletetitles` | `Id[]` | Bulk title delete |

**Auth**: The controller has no `[Authorize]` attribute. S2S auth is enforced by `PartnerRegistryAuthPolicy.S2S` (`ApplicationAuthorizationRequirement` against `AllowedS2SAppIds` config). PlayTest will need its AAD AppId added to that allowlist.

## OfferingV2 — actual properties (verified — file `Contracts/OfferingV2.cs:18-224`)

```csharp
public class OfferingV2
{
    public Id Id { get; set; }
    public IList<Id> TitleCollections { get; set; }
    public Id PartnerId { get; set; }
    public string Name { get; set; }
    public PlayerAuthenticationOptions AuthenticationOptions { get; set; }
    public PlayerAuthorizationOptions AuthorizationOptions { get; set; }   // ← where flight/DNA goes
    public Dictionary<Id, Id> DefaultAllocationPools { get; set; }
    public ICollection<string> Regions { get; set; }
    public ICollection<string> TemporarilyDisabledRegions { get; set; }
    public bool AllowRegionSelection { get; set; }
    public JObject ClientCloudSettings { get; set; }
    public bool FallbackToGA { get; set; }
    public ICollection<Id> SelectableSystemUpdateGroups { get; set; }
    public uint? ServerIdleWarningTimeInSeconds { get; set; }
    public uint? SessionIdleTimeInSeconds { get; set; }
    public uint? ResourceHoldTimeoutInSeconds { get; set; }
    public uint? ResourceAllocationTimeoutInSeconds { get; set; }
    [Obsolete] public uint? MaxGameplayTimeInSeconds { get; set; }
    public IList<TimeLimit> PerAccessLevelLimitOverrides { get; set; }
    public IList<ProgramTimeLimit> TimeLimitOverrides { get; set; }
    public bool EnableXboxRegionFallback { get; set; }
    public IList<Id> DefaultAllowedServerTypes { get; set; }
    public ICollection<Id> SelectableServerTypes { get; set; }
    public IList<string> DefaultSupportedInputTypes { get; set; }
    public bool DedicatedFQDN { get; set; }                                // for [offeringid].gssv-play-*.xboxlive.com
    public bool AllowCrossOfferingResume { get; set; }
    public bool AllowRigAccessoryConnections { get; set; }
    public int? MaxParallelSessionsPerCore { get; set; }
    public IReadOnlyDictionary<Id, int> SystemUpdateGroupWeights { get; set; }
    public Id ServiceLevel { get; set; }
    public Id TabAccessGroup { get; set; }
    public bool CanUploadGameFiles { get; set; }
    public string Notes { get; set; }
    public List<string> OwnedBy { get; set; }
    public DateTime? ExpirationTime { get; set; }                          // can set this from playtest.EndDate
}
```

Notable: **no `Type`, no `DisplayName`, no `Description` field**. Earlier versions of this doc invented those — they don't exist. Use `Name` and `Notes`.

## PlayerAuthorizationOptions — access control (verified — `Contracts/PlayerAuthorizationOptions.cs:11-79`)

```csharp
public class PlayerAuthorizationOptions
{
    public bool AllowAllAuthenticatedUsers { get; set; }
    public ICollection<string> AllowedPlayerUserIds { get; set; }
    public ICollection<string> AllowedPlayerXuids { get; set; }
    public ICollection<string> AllowedPuids { get; set; }
    public ICollection<Flight> AllowedFlights { get; set; }            // ← primary candidate for playtest audience
    public ICollection<Flight> AdditionalClaimsFlights { get; set; }
    public ICollection<string> AllowedEntitlements { get; set; }
    public ICollection<string> AllowedSubscriptions { get; set; }
    public ICollection<string> AllowedCountries { get; set; }
    public Id? AllowedSandboxId { get; set; }
    public bool CheckStoreEntitlements { get; set; }
}
```

## Flight — the audience primitive (verified — `Contracts/Flight.cs:10-23`)

```csharp
public class Flight
{
    // If not null/empty, users must be members of this flightId to access this offering
    public string FlightId { get; set; }

    // If access requires flight membership AND this is non-empty,
    // users must additionally be in one of these variations
    public ICollection<string> FlightVariations { get; set; }
}
```

**No `DnaGroup` contract class exists in the repo**. The relationship between "DNA group" (project doc language) and `Flight.FlightId` is the key open question — see `architecture/flights-and-dna-groups.md`.

## Title — verified (`Contracts/Title.cs:18-247`)

Key fields for our use:
- `Id Id` — title source id
- `Id PartnerId`, `Id? OfferingId`
- `Id Platform`
- `string ProductId` — Universal Store Product ID (**no `BuildId` field exists**)
- `ICollection<Flight>? AllowedFlights`
- `PackageFlightingConfig? PackageFlightingConfig` — per-title package flighting (has `FlightId`, `PrincipalGroupId : Guid`, `XipFlightId`, `XipVariationId`)
- `DateTime? AvailableTime`, `DateTime? ExpirationTime`
- `bool IsEnabled`
- `IList<Id>? Programs`, `IList<ProgramConfig>? ProgramConfigs`
- `UserContentPolicyConfig? UserContentPolicyConfig`

**Important**: `Title` has NO `BuildId` or `PackageId` field. Partner Registry maps offerings → titles by **ProductId only**. The "which build is current" decision lives in GSSV / XPackage, not here.

## Validation rules around flights (verified — `Processors/Validation/ValidationProcessorUtilities.cs:1920-1985`)

- `PackageFlightingConfig` is rejected for title collections (per-title only)
- `FlightId` must be present and parse as `Guid`
- `PrincipalGroupId` must be non-empty
- `XipFlightId`/`XipVariationId` must both exist for "enrollable" flights
- `XusAudience.IsGAFlight(flightId)` distinguishes "GA" flights from flighted/non-GA ones (suggests there's an `XusAudience` lookup somewhere that knows about flight categories — worth asking team)

## What's NOT in this repo (and was wrongly attributed to it in earlier notes)

- No "playtest" type/category on OfferingV2
- No DnaGroup contract or model
- No GSSV ContentIngestion *call sites* — only the DI registration for the NuGet client, used for read-side validation
- No "Private Offering" first-class concept — a private offering is just an OfferingV2 with restrictive `AuthorizationOptions.AllowedFlights`
- No xplaytest portal frontend
