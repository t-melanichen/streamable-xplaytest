# Architecture: Package Ingestion into xCloud

> **Status**: Investigation summary — contains a significant gap

The P0 goal *"When a new build is uploaded for a playtest that is configured for cloud, that build is ingested into Game Streaming Services"* requires kicking off package ingestion on the xCloud side. This document captures what we know and what is missing.

---

## What We Know

### 1. The ingestion client lives in a NuGet package
**File**: `services.partnerregistry\src\Product\PartnerRegistryService\PartnerRegistryService.csproj:24`

```xml
<PackageReference Include="Microsoft.GameStreaming.ContentIngestion.Client" Version="1.0.2604.2902" />
```

The actual `IContentIngestionClient` interface and the request/response contracts are **NOT in any of the 3 local repos**. They are in this NuGet package, which is owned by the **GameStreaming / GSSV team**.

### 2. Partner Registry already wires it up — but only consumes data
**File**: `services.partnerregistry\src\Product\PartnerRegistryService\Startup.cs:241`

```csharp
services.AddGSHttpClient<IContentIngestionClient, ContentIngestionClient>();
```

**File**: `services.partnerregistry\src\Product\PartnerRegistryService\appsettings.en-development.json:41-49`
```json
"ContentIngestionClientSettings": {
  "BaseUri": "https://americas.gssv-dev-test.xboxlive.com/api/contentingestion/"
}
```

So we know:
- The base URI pattern: `https://{region}.gssv-{env}.xboxlive.com/api/contentingestion/`
- DI registration uses `AddGSHttpClient<>` (a GameStreaming-specific extension method — also from the NuGet)

But Partner Registry **does not call any ingestion-trigger methods**. A grep for `IngestPackage`, `IngestBuild`, `IngestContent` returned **no usages in source code** — only validation that reads metadata about already-ingested packages.

### 3. Validation reads ingested package metadata
**File**: `services.partnerregistry\src\Product\PartnerRegistryService\Processors\Validation\ValidationProcessorUtilities.cs:1514-1545`

```csharp
var package = packageInfos.FirstOrDefault(
    p => p.MainGameProperties.ContainsKey(MetadataPropertyNames.MsCatalogProductId) &&
         productId.Equals(p?.MainGameProperties[MetadataPropertyNames.MsCatalogProductId]?.ToString(),
            StringComparison.OrdinalIgnoreCase));
```

This shows that ingested packages carry:
- `MainGameProperties` (a dictionary including `MsCatalogProductId`)
- `Markets` (a collection of market codes for multi-market ingestion)

The full shape lives in the NuGet types `WireStreamingPackage` and `PackageInfoForOfferValidation`.

### 4. PlayTest's existing publish processor
**File**: `Xbox.Xbet.Service\src\PlayTest\PlayTest\ServiceBus\MessageProcessors\XPackagePlaytestPublishWorkflowJobStatusTopicProcessor.cs`

```csharp
JobType => CoreXPackageJobType.XPackagePlaytestPublishWorkflow;
// ...
await playtestBusinessLogic.UpdatePlaytestStatusAsync(...)
```

The processor fires when an `XPackage` publish workflow completes. **It currently only updates the playtest status — no GSSV/Partner Registry calls.** This is the extension point for our work.

---

## The Open Questions

We **do not yet know**:

### A. The exact ingestion contract
What HTTP route do you POST to? What goes in the body? Possibilities to investigate:
- `POST /api/contentingestion/v1/packages` with a package manifest?
- `POST /api/contentingestion/v1/ingest` with a package URI?
- Something else entirely?

**Where to find this**: 
1. Pull the `Microsoft.GameStreaming.ContentIngestion.Client` NuGet package and inspect its public interface
2. Find another service in Microsoft that calls it (search internal codebase for `IContentIngestionClient` usages outside Partner Registry)
3. Ask the GSSV team (Anthony Keller / Timi Bolaji)

### B. The "package URI" question
xCloud needs a way to fetch the package. We don't yet know:
- Does PlayTest need to upload the package to a specific storage location and pass the URL?
- Or does xCloud know how to fetch the package from XPackage Service given just an ID?
- What package ID format is expected (XPackage ID? Partner Center Product ID? Both?)

**Where to find this**: same as above. Also examine the `WireStreamingPackage` and `PackageInfoForOfferValidation` types from the NuGet to see what properties an ingested package has.

### C. Auth requirements
`AddGSHttpClient<>` likely handles S2S auth automatically, but PlayTest will need:
- An AAD app registration with the right permissions on the ContentIngestion API
- The correct resource ID / scope to request a token for

### D. Sync vs async ingestion
The success metric says "A few seconds / minutes after uploading a new build". This implies async — PlayTest probably needs to:
1. Trigger ingestion (POST returns a job ID or 202 Accepted)
2. Either poll for completion, or subscribe to a completion event
3. Only then update the Partner Registry offering's title to point at the new build

### E. Is there an existing ingestion code path?
**Important hint**: the `XPackagePlaytestPublishWorkflow` is named like there's already a workflow that "publishes" the package somewhere. The publish destination today might already be xCloud-adjacent, just gated for cloud-streaming use. Worth checking with the team whether existing publish already does ingestion partially.

---

## Recommended Path Forward

1. **Pull the NuGet package** locally. The intern can run:
   ```
   nuget install Microsoft.GameStreaming.ContentIngestion.Client -Version 1.0.2604.2902
   ```
   Then inspect the assemblies (e.g., with ILSpy or a similar decompiler — `XBET.Services.sln` likely already references this via NuGet restore so it might already be cached locally).

2. **Search the org-wide code** for `IContentIngestionClient` usage. Other Xbox teams must call it.

3. **Have a 30-min sync with the GSSV team** to get the contract walked through. Bring concrete questions:
   - What is the HTTP route(s)?
   - What identifies a package (URI vs ID)?
   - Sync or async completion?
   - What S2S permissions does PlayTest need?
   - Are there test/dev sandboxes we can hit?

4. **Document the answer in `design/api-contracts.md`** once known.
