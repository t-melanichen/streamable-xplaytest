# Research: Partner Registry (services.partnerregistry)

## Location
- Repo: `services.partnerregistry`
- Source: `src/`

## What It Does
Manages the xCloud partner-facing registry: **offerings**, **titles**, **streaming configs**, **regions**, **pools**. Changes are stored in Cosmos DB and can be synced through Azure DevOps PRs for production approval.

## Key Controllers

| Controller | Route | Purpose |
|-----------|-------|---------|
| `OfferingsController` | `/v1/offerings` | Full CRUD for offerings + titles within offerings |
| `RegistryManagementController` | Registry management | Read/submit/PR lookup for registry changes |
| `StreamingConfigController` | `/v1/streamingconfig` | Get/update default streaming builder config |
| `StreamingStackConfigController` | `/v1/streamingstack/configs` | Streaming stack configs |
| `GamePassProductsController` | Game Pass products | Product cache + misconfigurations |

## OfferingV2 Model (key fields for playtest integration)

From `src/Product/PartnerRegistryClient/Contracts/OfferingV2.cs`:

- **Auth/AuthZ config** — who can access the offering (this is where DNA group goes)
- **Allocation pools** — VM/resource allocation
- **Regions** — where streaming is available
- **SUGs** (Server Update Groups)
- **DedicatedFQDN** — pattern: `[offeringid].gssv-play-prod.xboxlive.com`
- **Session limits** — concurrent session caps
- **Service level** — priority/tier
- **TAB access** — testing access?
- **Expiration** — when offering expires

## Title Model (key fields)

From `src/Product/PartnerRegistryClient/Contracts/Title.cs`:

- **Platform** — Xbox, PC, etc.
- **Product IDs** — maps to Partner Center product
- **Entitlements** — access rules
- **Flights** — feature flags for gradual rollout
- **Countries** — geo availability
- **Availability/Expiration** — time-bounded access
- **Server types/SUGs** — VM configuration
- **Input types** — controller, touch, keyboard
- **Program configs** — additional program-specific configuration

## What Needs to Change (P0)

### 1. Support "Playtest" Offering Type
- May need a new offering category/flag to distinguish playtest offerings from production offerings
- Enables filtering in DevApi Reader Portal (P2 stretch goal)
- Consider: should playtest offerings bypass the ADO PR approval flow? (likely yes for speed)

### 2. DNA Group Auth Support
- OfferingV2 already has auth config — need to ensure DNA groups work as the access mechanism
- Currently "flight" / "DNA Group" auth doesn't show up in a user's `/offerings` call (known issue from project doc)
- **This is the key blocker**: GSSV's `/offerings` endpoint needs to return offerings where the user is in the DNA group

### 3. Naming Convention for Playtest Offerings
- Need a unique, deterministic naming scheme: e.g., `playtest-{playtestId}` or `xpt-{sellerId}-{productId}-{playtestId}`
- Must be easily filterable for the DevApi portal

### 4. Fast-Path Creation (Skip PR Approval)
- Normal offering changes go through ADO PR sync for production safety
- Playtest private offerings need instant creation (within seconds per success metrics)
- May need a direct-write path or auto-approved PR flow

## Existing Patterns to Leverage
- The OfferingsController already supports full CRUD — PlayTest just needs to call it
- Auth patterns already exist — just need to configure correctly for DNA groups
- DedicatedFQDN pattern already in place — gives us the streaming endpoint URL
