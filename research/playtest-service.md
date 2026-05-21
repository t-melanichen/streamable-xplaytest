# Research: PlayTest Service (Xbox.Xbet.Service)

## Location
- **Core service**: `src/PlayTest/PlayTest/`
- **Front-door (Partner Center proxy)**: `src/PlayTestFD/PlayTestFD/`
- **ContentAccess** (related): `src/ContentAccess/ContentAccess/`

## Architecture

### PlayTest Core
- **Route**: `/products/{partnerCenterProductId}/playtests`
- **Controllers**:
  - `PlaytestsController` — full CRUD: Get, GetAll, Create, Update, Delete, Publish
  - `PlaytestGroupsController` — audience/group management
  - `ContentAccessController` — content access integration
- **Business Logic**: `PlaytestBusinessLogic` with dependency injection
- **Database**: Cosmos DB (via `Database/` folder)
- **ServiceBus processors** (key for build ingestion):
  - `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` — fires when a build is published
  - `XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor` — fires when a playtest is deleted
- **Validations**: Audience, Package, AgeRatings rules
- **Service Clients**: Uses typed `ServiceClients/` for S2S calls

### PlayTestFD (Front Door)
- **Route**: `/api/{locale}/dashboard/products/{productId}/playtests`
- **Controllers**:
  - `PlaytestsController` — proxies GetAll, GetById to PlayTest core
  - `GroupsController` — proxies audience group operations
- **Headers required**: `MS-CV`, `SellerAccountId`
- Thin proxy layer — no business logic of its own

### Key Data Models
- `PlaytestResponse` — full playtest details
- `PlaytestModel` — summary model
- `PlaytestsResponseModel` — list response
- `PlaytestStatus` enum — lifecycle states

## What Needs to Change (P0)

### 1. Add Partner Registry ServiceClient
- Create new `IPartnerRegistryClient` / `PartnerRegistryClient` in PlayTest's `ServiceClients/`
- Follow existing pattern: extend `Shared.Common.ServiceClient`
- Methods needed:
  - `CreateOfferingAsync(offeringConfig)` → POST `/v1/offerings`
  - `DeleteOfferingAsync(offeringId)` → DELETE `/v1/offerings/{id}`
  - `UpdateOfferingAsync(offeringId, config)` → PATCH/PUT `/v1/offerings/{id}`
  - `UpdateOfferingTitleAsync(offeringId, titleConfig)` → PUT `/v1/offerings/{id}/titles`

### 2. Modify PlaytestBusinessLogic
- On `CreatePlaytest`: if cloud streaming enabled, call Partner Registry to create offering
- On `DeletePlaytest`: if offering exists, call Partner Registry to delete it
- On audience/group update: call Partner Registry to update offering DNA group config
- Store `offeringId` in playtest Cosmos document for later reference

### 3. Extend ServiceBus Publish Processor
- `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor`: after build is published, trigger:
  1. GSSV build ingestion (new S2S call)
  2. Partner Registry title update (configure offering to use new build)

### 4. Add Flights/Allowlist Check
- Gate private offering creation behind a flight flag
- Only allow specific sellers (allowlist) during development
- Leverage existing `Configurations/` patterns

### 5. Return Offering Details via PlayTestFD
- PlayTestFD response should include `streamingLink` constructed from offering details
- Format likely: `https://xbox.com/play/launch/[offeringId]` or similar Bayside deep link
