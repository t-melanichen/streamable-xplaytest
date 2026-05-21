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

### 5. The rubber-stamp PR problem — confirmed P0 blocker (per Jack Heuberger, 2026-05-21)
Every change to an offering in Partner Registry (adding a title, updating audience) creates an ADO PR. In test, an MSI created the PR and Jack just approved it (~1 min later it reflected in services). In **prod**, SFI now forwards the requester's user identity (`Melanie Chen made the pull request`) so you **cannot self-approve**. There's no automatic rubber-stamp today. This blocks every playtest-with-streaming, not just initial setup.

**Status**: Jack + Timi are working on a workaround. Loose proposal:
- Create a separate `playtest` branch in Partner Registry alongside `main`
- That branch has relaxed protections (no PR required to merge)
- Special client methods that PlayTest's S2S identity calls to write to the `playtest` branch
- Services reference the `playtest` branch in playtest-specific cases

**Status quo for the intern demo**: manually rubber-stamp the PR to move the playtest flow along.

**Do nothing on the PlayTest side until Jack + Timi land the workaround**. We're net consumers of whatever they ship.

### 5a. TitleIngestion `assets` parameter for unpublished playtest builds — likely answered (2026-05-21)
Per Jack: *"we just need a Big ID"* (= `PartnerCenterProductId`). GSSV pulls product metadata from the Microsoft Store using just the Big ID and stores it. Implies **`assets: null` is correct for playtests** — same pattern as the bulk-ingest UI — assuming the playtest's PartnerCenterProductId resolves through the store flow.

**Open sub-question (still ask Anthony or Jack)**:
- Does this hold for **unpublished** playtest builds? The Big ID exists in Partner Center before any public store publication, but does GSSV's store-pull path accept Big IDs that aren't yet store-listed? If not, what's the alternative for the not-yet-published case?

### 5b. TitleIngestion `sandboxId` for playtests — partially answered
The bulk-ingest UI takes a sandbox id. Jack didn't explicitly say what to pass for a playtest.

**Ask Jack**:
- For a playtest, do we pass the playtest's allowlisted sandbox id (e.g. `Playtest-{sellerId}`), or `RETAIL`?

### 5c. TitleIngestion `flightId` — singular vs offering-side gating
`ProductIngestion.JobParameters.flightId` is **singular** (and optional). The bulk-store path passes `null` and lets the offering's `AuthorizationOptions.AllowedFlights` gate access.

**Ask Jack / Anthony**:
- Confirm: for playtests with multi-flight audiences (multiple DNA groups), pass `flightId: null` at ingestion and put the flights on the offering — correct?
- If not, do we need to submit multiple TitleIngestion jobs (one per flight)?

### 5d. After TitleIngestion completes — do we have to PUT anything back?
`JobState.StreamingPackageIds : IList<Guid>` are the ingested package IDs. `Title.cs` in Partner Registry has **no slot for them** — strongly suggests GSSV joins them server-side via ProductId. Jack's framing of *"step one is ingest the game, step two is add it to an offering (which depending on the ingestion step you could potentially get for free)"* reinforces this.

**Ask Anthony**:
- Confirm: once TitleIngestion completes, GSSV server-side links the StreamingPackageIds to the offering through ProductId — PlayTest doesn't need to do a follow-up PUT to the offering / title. Right?
- Is there a status notification we should subscribe to, or do we poll `GetJobStateAsync(jobId)`?

### 5e. Authentication for hitting GSSV from S2S — in progress (Jack, 2026-05-21)
Verified 2026-05-21 that hitting `https://gssv-dev-test.xboxlive.com/api/partnerregistry/v1/offerings` with no token, an Azure DevOps PAT, or a personal-AAD token all return `HTTP 401` from the Xbox Live edge.

**Jack's plan** (committed for ~tomorrow):
- Add Melanie to the **GSAM Services group** (for Dev API access as a dev)
- Register PlayTest's service client app id in **Sage** (Service API Gateway) for the production S2S path
- Add a route entry in Sage for `PUT /v1/offerings/{id}` (and any other endpoints PlayTest needs)
- Brian (manager) to help track down the service client app id for the PlayTest service identity
- Provide a **LinkPad** starter scaffold so Melanie can prototype GSSV calls locally as C# scripts against Dev API

**Reframes the question**: there's no need to mint our own XSTS/AAD token by hand. Going through Dev API (for dev) or Sage (for S2S prod) handles the auth.

### 5f. New question: One offering per playtest, or shared offering with per-playtest titles?
From Jack: *"Our PMs come in here, type in the game [to add to an existing testing offering]"* — implies a shared "testing offering" model with titles added per game.

But earlier: *"[Offerings] let us create special instances of xCloud for different groups, different flights, different individual users with different games and whatever associated with them"* — implies per-flight or per-playtest offerings are possible.

**Ask Jack / Anthony**:
- Should each playtest have its own dedicated `OfferingV2` (one playtest = one offering, with that playtest's audience flights on it), OR
- Should there be a shared "playtest" offering per seller / per audience-type, and we add a new Title per playtest into it?
- The shared model means fewer offerings to manage but trickier DNA-group gating (all titles share `AuthorizationOptions.AllowedFlights`); the per-playtest model is simpler but creates many offerings.

### 5g. New question: GSSV-side work to extend TitleIngestion to write to Offering
Per Jack: *"TitleIngestion adds that title to a title collection. So we would also need to expand this to work for offerings, but that's pretty easy to do."*

**Ask Jack**:
- Who owns this extension and what's the timeline? Is it a prerequisite for any PlayTest implementation we do?
- New `TitleIngestion.JobParameters` field (e.g. `OfferingId : Id?` instead of/in addition to `TitleCollection`)?
- Or a new job type entirely (e.g. `OfferingIngestion`)?

### 5h. New question: Do GSSV services support a way to add a playtest sandbox?
Sandboxes are an Xbox concept used heavily by GSSV (e.g. `RETAIL`, `CERT`, `Playtest-*`). For playtest TitleIngestion to work, the playtest's sandbox needs to be recognized by GSSV.

**Ask Jack**:
- Is there a one-time setup to register a new playtest sandbox with GSSV?
- Or does GSSV's store-pull path infer sandbox from the Big ID?

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
