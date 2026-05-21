# Open Questions for the Team

Use this as a working list. Cross things off as you get answers in your 1:1s with manager (Brian Bowman), mentor (Emma Park), or engineering experts (Anthony Keller, Timi Bolaji, Ashton Summer, Chuy Galvan).

---

## 🔴 P0 Blockers — answer these in the first 2 weeks

### 1. Where does the xplaytest portal frontend live?
The PlayTestFD route prefix is `/api/{locale}/dashboard/products/{productId}/playtests` — the "dashboard" suggests it's called from a dashboard (likely Partner Center). It is **not** in any of the 3 local repos:
- `Xbox.JS/apps/play-xbox` is Bayside (player-facing)
- `Xbox.JS/apps/xboxcom-edgewater` is the xbox.com Edgewater frontend
- `Xbox.Xbet.Service` is the backend monorepo
- `services.partnerregistry` is the offering management backend

**Ask**: Which repo holds the xplaytest portal frontend? Do you have access? Will UI changes (e.g., "Enable cloud streaming" toggle) be in scope, or will the PM coordinate that work with another team?

### 2. How does a DNA group become a flight ID?
The `Flight.FlightId` on `OfferingV2.AuthorizationOptions.AllowedFlights` is a `string`. `PackageFlightingConfig.PrincipalGroupId` on `Title` is a `Guid`. DNA groups are typically GUIDs. See `architecture/flights-and-dna-groups.md` for the full breakdown.

**Ask**: 
- Is `PackageFlightingConfig.PrincipalGroupId` the existing DNA group hook?
- Or should playtest set `AllowedFlights` with the DNA group GUID as `FlightId`?
- Or does this require adding a new property on `OfferingV2`?

### 3. What is the content ingestion API contract?
The `Microsoft.GameStreaming.ContentIngestion.Client v1.0.2604.2902` NuGet defines it but Partner Registry doesn't call any ingestion-trigger methods. See `architecture/package-ingestion.md`.

**Ask**:
- What is the HTTP route to kick off package ingestion?
- What package identifier does GSSV expect (XPackage ID, Product ID, URI)?
- Sync or async completion?
- What S2S permissions does PlayTest need on the ContentIngestion API?
- Is there a test/dev environment to hit?

### 4. Why don't flight-authed offerings show up in `/offerings`?
The project doc explicitly notes: *"Currently 'flight' / 'DNA Group' auth doesn't show up in a user's /offerings call. This is a problem for the UX."*

**Ask**: 
- Whose code owns the `/offerings` filtering logic? (Likely GSSV, not Partner Registry.)
- What change is needed to include flight-authed offerings in the user's response?
- Is this a P0 blocker for the project or a parallel workstream?

### 5. Should playtest offerings bypass the ADO PR approval flow?
Partner Registry has a `RegistryManagementController` that creates Azure DevOps PRs for offering changes. Success metric requires "within a few seconds" of playtest creation — incompatible with PR review.

**Ask**:
- Can PlayTest call `OfferingsController` directly (`PUT /v1/offerings/{id}`) and skip the PR flow?
- Or does playtest offering creation need to go through a fast-path that auto-approves PRs?

---

## 🟡 P1 — needed before design review (Week 3-4)

### 6. What is the shareable streaming link format?
Existing Bayside route is `/stream/:productId/:productName?`. Dev mode supports `?configs=streaming.auth.offering:<offeringId>`.

**Ask**:
- Should the shareable link use this format: `xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}`?
- Or do we want a new dedicated playtest route like `xbox.com/play/playtest/{token}`?
- How does the link survive auth redirects (must not leak details pre-login)?

### 7. What metadata does the playtest offering need to display in Bayside?
The P0 goal: *"testers know this is a non public version of the game and as much metadata as possible (title, genre, art, etc.) that the store / playtest has is available to be seen."*

The PlayTest `PlaytestResponse` has `Name`, `Description`, `GameArtUrl`, `PlaytestXProductInfo`. Bayside fetches `catalog.queries.productInfo.detailed({ productId })`.

**Ask**: For unreleased games, does the catalog already have product metadata? If not, where does playtest art/metadata come from?

### 8. How does PlayTest currently model audiences?
PlayTest has `Audiences` on the contract but we haven't verified whether audience IDs are XUIDs, emails, DNA group IDs, or some abstraction.

**Action for intern**: read `src/PlayTest/PlayTest/Database/Entities/` and the `Audience` contract — then verify with Anthony Keller.

### 9. What's the seller allowlist mechanism in PlayTest?
The P0 goal: *"Creation of a private offering can be limited to a specific title / seller for internal testing"*. The project doc mentions "xplaytest 'flights' functionality".

**Ask**: Is there an existing PlayTest config-driven seller allowlist we can reuse? Or do we need to add a new appsettings section?

### 10. Should the offering inherit playtest expiration?
PlayTest has `EndDate`. Partner Registry `OfferingV2.ExpirationTime` is `DateTime?`. 

**Ask**: Should we set `OfferingV2.ExpirationTime = playtest.EndDate` so offerings auto-expire? Any cleanup-job risk?

---

## 🟢 P2/P3 — stretch goals, defer until P0 working

### 11. DevApi Reader Portal filtering
**Ask**: Where is the DevApi Reader Portal code? Can we add a "Playtest Offerings" tab/filter?

### 12. Bayside/Garrison playtest details "Stream Now" button
**Ask**: P3 stretch — would adding a "Stream Now" button to the existing Garrison playtest details page conflict with any planned Garrison changes?

### 13. Launch args support
**Ask**: Does GSSV already support launch arguments? If so, where can we pass them through the URL?

### 14. Streaming region / touch controls config in xplaytest portal
**Ask**: Are there existing partner-facing UI components for editing offering streaming config (regions, input types) we can drop into the xplaytest portal?

### 15. Tester feedback collection
**Ask**: Is there an existing feedback collection pipeline (e.g., OCV, Office Customer Voice) that xplaytest can integrate with?

---

## 🛠️ Practical onboarding asks

### 16. Repo access
**Ask**: 
- Permissions to push branches on `Xbox.Xbet.Service`, `Xbox.JS`, `services.partnerregistry`?
- Permissions to view/run the ADO Team board: `https://dev.azure.com/microsoft/Xbox/_sprints/taskboard/Juno/...`?
- Access to internal NuGet feeds (to download `Microsoft.GameStreaming.ContentIngestion.Client`)?

### 17. Dev / test environments
**Ask**: 
- How do I run PlayTest locally with a real or mocked Partner Registry?
- Is there a shared dev sandbox where I can create test playtests and offerings?
- Is there a Garrison build I can use to test private offerings end-to-end?

### 18. Existing related work
**Ask**: 
- Has anyone prototyped any of this before? Any branches/spikes/PRs to look at?
- Are there design docs for adjacent features (e.g., GSSV onboarding, Partner Center playtest flow) I should read first?

### 19. Identify the GSSV/xCloud team contact
The project doc lists Anthony Keller / Timi Bolaji / Ashton Summer / Chuy Galvan as engineering experts but doesn't say who owns what. 
**Ask**: Who's the right person for GSSV/ContentIngestion vs PlayTest vs Partner Registry vs Bayside?

### 20. Telemetry / monitoring expectations
For the Week 11-12 "Documentation, Metrics, Polish" milestone.
**Ask**: What dashboards exist today for PlayTest, Partner Registry, and GSSV? Where do I add new metrics?
