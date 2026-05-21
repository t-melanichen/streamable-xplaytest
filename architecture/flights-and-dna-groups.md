# Architecture: Flights, DNA Groups, and Playtest Audiences

> The DNA group → Flight mapping is **already implemented in PlayTest** and used today by the XPackage publish workflow. The same pipeline gets reused for the GSSV offering creation.

---

## The data flow (verified — `PlaytestBusinessLogic.cs:1054-1123`)

```
Playtest creator picks audiences in xplaytest portal
            │
            ▼
PlaytestAudience  { AudienceId : Guid, GmsGroupId : string }
            │
            ▼  IGMSServiceClient.GetUserGroupAsync(sellerId, gmsGroupId)
            │
GMS Group response  { Id, Name, Members[], UserDnAGroupIds[] }
            │
            ▼  flatten + distinct across all audiences
List<string> flightIds   ←──── these ARE the DNA group GUIDs
            │
            ├─► Used today as:
            │     PlaytestPublishJobParameters.FlightIds
            │     PlaytestDeleteJobParameters.FlightIds
            │     (consumed by XPackagePlaytestPublishWorkflow)
            │
            └─► For our project, also used as:
                  OfferingV2.AuthorizationOptions.AllowedFlights = flightIds.Select(id => new Flight { FlightId = id })
                  Title.AllowedFlights = same
                  (optionally) Title.PackageFlightingConfig.FlightId / PrincipalGroupId
```

## The contract classes

### `PlaytestAudienceRequest` / `PlaytestAudienceResponse` (PlayTest)
```csharp
[ProtoContract]
public record PlaytestAudienceRequest
{
    [ProtoMember(1)] public Guid AudienceId { get; set; }
    [ProtoMember(2)] public string GmsGroupId { get; set; }   // GMS = Group Management Service
}
```
Validated by `AudienceMustBeValidRule` — every audience must have a non-empty `GmsGroupId`.

### GMS Group (resolved via `IGMSServiceClient`)
The relevant property after the GMS call is `group.UserDnAGroupIds : string[]` — a flat list of DNA group GUIDs. PlayTest just dedupes across audiences.

### `Flight` (GSSV, in `services.partnerregistry/.../Contracts/Flight.cs`)
```csharp
public class Flight
{
    public string FlightId { get; set; }                       // DNA group GUID goes here
    public ICollection<string> FlightVariations { get; set; }  // null for playtest
}
```

### `PlayerAuthorizationOptions.AllowedFlights : ICollection<Flight>`
On `OfferingV2.AuthorizationOptions`. Each DNA group becomes one `Flight` entry. Membership in any one allowed flight grants access.

## Reuse strategy

The intern should **not** reinvent this. Two options:

### Option A — extract a shared helper (recommended)
Refactor `PlaytestBusinessLogic.ResolveUserDnaGroupIds` (private, line 1060) into an `IAudienceFlightResolver` interface in `PlayTest.BusinessLogic`, used by:
- `QueuePublishJobAsync` (existing call site)
- `QueueDeleteJobAsync` (existing call site)
- the new offering-creation path in `PartnerRegistryServiceClient`

### Option B — duplicate the call in the new client
If the refactor is risky, just call `IGMSServiceClient.GetUserGroupAsync` directly from the new `PartnerRegistryServiceClient` and project to `Flight[]`. Less elegant but lower-risk and ships faster.

## Validation rule parity

PlayTest already enforces: "if `PlaytestType.Private` and FlightIds is empty → reject" (`PlaytestBusinessLogic.cs:897`).

Mirror this for the offering call: if cloud streaming is enabled and no DNA groups resolve to flight ids, **fail loudly before calling GSSV** — never create an offering with `AllowAllAuthenticatedUsers = true` for a playtest.

## Open questions for the team (confirm before shipping)

1. **`PackageFlightingConfig.PrincipalGroupId : Guid`** — is this the same DNA group id as the flight, or a separate concept? Validation rule says it must be non-empty and `FlightId` must parse as Guid (so the convention is "DNA group GUID = FlightId = PrincipalGroupId" — sanity check with Anthony Keller).
2. **`XusAudience.IsGAFlight(flightId)`** is used to distinguish GA flights from non-GA. Where does the `XusAudience` definition live? If it's needed at runtime to decide pool selection, PlayTest's offering needs to follow the same convention so the validation passes.
3. **Are FlightVariations ever needed for playtest**? Today's PlayTest pipeline only resolves bare flight ids — no variations. Confirm the GSSV side accepts bare-flight offerings without variations.
