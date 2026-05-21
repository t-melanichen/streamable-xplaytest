# Architecture: Flights, DNA Groups, and Playtest Audiences

> **Status**: Investigation summary — contains a critical open question for the team

This is one of the trickiest mapping problems in the project. The project doc says:

> *"Only users authorized for the playtest are configured for access to the Private Offering, likely through the integration of DNA group support in xCloud."*

But the existing **OfferingV2** model in `services.partnerregistry` does **not** have a "DNA group" concept. It has **flights**. This document explains what we found and the open question.

---

## What the Code Actually Has

### `Flight` model
**File**: `services.partnerregistry\src\Product\PartnerRegistryClient\Contracts\Flight.cs:10-23`

```csharp
public class Flight
{
    /// If not null or empty, users must be members of the provided flightId to access this offering
    public string FlightId { get; set; }

    /// If access requires flight membership, and this value is not null or empty, users must be
    /// not only members of the flight, but also one of the specific variations listed
    public ICollection<string> FlightVariations { get; set; }
}
```

### `PlayerAuthorizationOptions` (on `OfferingV2.AuthorizationOptions`)
**File**: `services.partnerregistry\src\Product\PartnerRegistryClient\Contracts\PlayerAuthorizationOptions.cs:11-79`

```csharp
public class PlayerAuthorizationOptions
{
    public bool AllowAllAuthenticatedUsers { get; set; }
    public ICollection<string> AllowedPlayerUserIds { get; set; }
    public ICollection<string> AllowedPlayerXuids { get; set; }
    public ICollection<string> AllowedPuids { get; set; }
    public ICollection<Flight> AllowedFlights { get; set; }          // ← USE THIS
    public ICollection<Flight> AdditionalClaimsFlights { get; set; }
    public ICollection<string> AllowedEntitlements { get; set; }
    public ICollection<string> AllowedSubscriptions { get; set; }
    public ICollection<string> AllowedCountries { get; set; }
    public Id? AllowedSandboxId { get; set; }
    public bool CheckStoreEntitlements { get; set; }
}
```

### `PackageFlightingConfig` (on `Title.PackageFlightingConfig`)
**File**: `services.partnerregistry\src\Product\PartnerRegistryClient\Contracts\PackageFlightingConfig.cs:11-30`

```csharp
public class PackageFlightingConfig
{
    public Id FlightId { get; set; }
    public Guid PrincipalGroupId { get; set; }   // ← suspicious: is this the DNA group?
    public Id XipFlightId { get; set; }
    public Id XipVariationId { get; set; }

    public bool IsEnrollable() =>
        !string.IsNullOrWhiteSpace(this.XipFlightId) &&
        !string.IsNullOrWhiteSpace(this.XipVariationId);
}
```

`PrincipalGroupId` is a `Guid`. **DNA groups are typically expressed as GUIDs.** This strongly suggests `PrincipalGroupId` IS the DNA group reference at the package-flighting level — but the team must confirm.

### Validation rules
**File**: `services.partnerregistry\src\Product\PartnerRegistryService\Processors\Validation\ValidationProcessorUtilities.cs:1920-1985`

- `PackageFlightingConfig` is rejected for title collections (i.e., it's per-title)
- `FlightId` must be present
- `FlightId` must parse as `Guid`
- `PrincipalGroupId` must be non-empty
- `XipFlightId` and `XipVariationId` must both exist for "enrollable" flights

### No `DnaGroup` class exists in the repo
A grep for `DnaGroup`, `Dna`, `Audience`, `AccessGroup`, `UserGroup` in `services.partnerregistry/src/` returned **no contract classes**. The closest things are the `Flight` model and `PrincipalGroupId` on `PackageFlightingConfig`.

---

## The Playtest Side

In PlayTest (`Xbox.Xbet.Service/src/PlayTest`), playtest audiences are represented as `Audiences` on the `PlaytestResponse`/`PlaytestCreateRequest`/`PlaytestUpdateRequest` contracts. The PlaytestGroups controller (`/products/{productId}/playtestgroups`) manages this list. The audience model in PlayTest is the source of truth for "who can test."

What we don't yet know:
- Does the PlayTest `Audience` model already hold DNA group IDs?
- Or does it hold user XUIDs / emails that get *resolved* into a DNA group at runtime?

**Action**: read `src/PlayTest/PlayTest/Database/Entities/` and the `Audience` contract to find out (queued).

---

## ⚠️ Open Question for the Team

> **How should a playtest audience's DNA group(s) be encoded into an `OfferingV2.AuthorizationOptions`?**

Three possibilities — please confirm with the team (likely Anthony Keller or Timi Bolaji):

### Option A — Use `AllowedFlights` with the DNA group ID as `FlightId`
```csharp
offering.AuthorizationOptions.AllowedFlights = new[]
{
    new Flight { FlightId = "<dna-group-guid>", FlightVariations = null }
};
```
- **Pro**: Leverages existing model; no schema changes.
- **Con**: Relies on convention — `FlightId` is a `string`, so it could hold anything. The auth pipeline downstream must know to interpret this as a DNA group lookup.

### Option B — Set `PackageFlightingConfig.PrincipalGroupId` on each `Title`
```csharp
title.PackageFlightingConfig = new PackageFlightingConfig
{
    FlightId = "<some-flight-guid>",
    PrincipalGroupId = Guid.Parse("<dna-group-guid>"),
    XipFlightId = ...,
    XipVariationId = ...,
};
```
- **Pro**: `PrincipalGroupId` looks semantically correct (it's a `Guid`, not a string).
- **Con**: Required validation says `XipFlightId`/`XipVariationId` must also be filled in if enrollable — adds dependencies. Also per-title rather than per-offering.

### Option C — A new property must be added to `OfferingV2`
The P0 goal *"Private offerings support DNA group access compatible with playtest"* may explicitly require a new model field. The project doc notes:

> *"Currently 'flight' / 'DNA Group' auth doesn't show up in a user's /offerings call. This is a problem for the UX."*

This suggests there's a real gap — flights/DNA groups don't propagate to the user-facing `/offerings` response yet. The fix might require a Partner Registry contract change *and* a downstream consumer (GSSV) change.

---

## Questions to ask the team

1. Is `PackageFlightingConfig.PrincipalGroupId` the existing DNA group hook? If yes, what generates/registers DNA group GUIDs today?
2. Where is the "GA flight" defined (`XusAudience.IsGAFlight(flightId)` is used in validation)? Is there an `XusAudience` lookup that resolves a flight ID into a sandbox + DNA group?
3. When a user calls GSSV's `/offerings` endpoint, what code path determines whether a flight-authed offering is included in the response? What change is needed there?
4. Should playtest offerings get a brand new `Audience`/`DnaGroup` property on `OfferingV2`, or can they reuse the existing `AllowedFlights`?
5. Who manages the DNA group lifecycle — does xplaytest create the DNA group from the playtest audience, or is the DNA group already registered elsewhere and xplaytest just gets the ID?
