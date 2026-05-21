# Design Doc: Instantly Shareable Playtest (P0)

> **Author**: Melanie Chen  
> **Status**: Draft  
> **Last Updated**: 2026-05-21  
> **Reviewers**: Brian Bowman, Emma Park, Anthony Keller, Timi Bolaji

---

## 1. Problem Statement

Game creators using xplaytest can upload private RETAIL builds for testers, but testers must download hundreds of GBs locally. This creates barriers:
- Long download/install times
- Hardware provisioning burden on creators
- Security risk from local game code on physical devices

**Proposed solution**: Integrate xplaytest with Xbox Cloud Gaming so creators can share a streaming link — testers play instantly via the browser with no install.

---

## 2. Goals (P0 Scope)

| # | Goal | Success Metric |
|---|------|---------------|
| 1 | Creating a playtest creates a private offering | Offering visible in DevApi Portal within seconds |
| 2 | Deleting a playtest deletes the private offering | Offering fully cleaned up |
| 3 | Unique offering naming per playtest | Filterable by title/seller |
| 4 | Audience DNA group updates sync to offering | DNA group membership = streaming access |
| 5 | Creation gated by seller allowlist (flights) | Only approved sellers during development |
| 6 | DNA group auth works for offerings | User's `/offerings` call returns playtest offerings |
| 7 | Shareable streaming link in xplaytest portal | Link opens Bayside streaming session |
| 8 | No info leakage for unauthorized users | Bayside shows nothing pre-auth |
| 9 | New build uploads trigger GSSV ingestion | New build streamable within minutes |
| 10 | New build configured in private offering | Offering title matches uploaded build |

---

## 3. Architecture Overview

```
Creator → xplaytest Portal (PlayTestFD) → PlayTest Core
                                               │
                          ┌────────────────────┼────────────────────┐
                          │                    │                    │
                          ▼                    ▼                    ▼
                   Partner Registry      GSSV (build ingest)   ContentAccess
                   (create offering)     (streaming VMs)       (entitlements)
                          │                    │
                          └────────┬───────────┘
                                   ▼
                          Play Xbox (Bayside)
                          (tester streams game)
```

### Services Modified

| Service | Changes |
|---------|---------|
| **PlayTest Core** | New `PartnerRegistryClient`, offering lifecycle hooks in business logic, extended ServiceBus processor |
| **Partner Registry** | DNA group auth fix for `/offerings`, possible fast-path for playtest offerings |
| **Play Xbox (Bayside)** | Handle playtest link route, enforce auth-before-details, DNA group check |

---

## 4. Detailed Design

### 4.1 Playtest Creation → Private Offering

**Trigger**: Creator creates a playtest with `cloudStreamingEnabled: true`

**Flow**:
1. PlayTestFD receives `POST /api/{locale}/dashboard/products/{productId}/playtests` with new field `cloudStreamingEnabled`
2. PlayTest Core validates and persists playtest to Cosmos
3. If `cloudStreamingEnabled`, PlayTest calls Partner Registry:
   ```
   POST /v1/offerings
   {
     "offeringId": "xpt-{sellerId}-{playtestId}",
     "displayName": "Playtest: {gameName} ({playtestId})",
     "type": "playtest",
     "auth": {
       "dnaGroupId": "{playtestDnaGroupId}"
     },
     "regions": ["default"],  // or creator-configured
     "sessionLimits": { ... },
     "expiration": "{playtestExpiration}"
   }
   ```
4. Store returned `offeringId` in playtest Cosmos document
5. Construct streaming link: `https://xbox.com/play/launch/xpt-{sellerId}-{playtestId}`
6. Return link to creator via PlayTestFD response

**Flight gate**: Check `IsCloudStreamingEnabled` flight flag + seller allowlist before step 3.

### 4.2 Playtest Deletion → Offering Cleanup

**Trigger**: Creator deletes a playtest (or existing `XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor` fires)

**Flow**:
1. PlayTest Core receives delete request
2. Read `offeringId` from playtest Cosmos document
3. Call Partner Registry: `DELETE /v1/offerings/{offeringId}`
4. Delete playtest from Cosmos (existing behavior)

### 4.3 Audience/DNA Group Update → Offering Config Sync

**Trigger**: Creator changes the DNA group for a playtest's audience

**Flow**:
1. `PlaytestGroupsController` receives group update
2. PlayTest business logic updates audience in Cosmos
3. If playtest has `offeringId`, call Partner Registry:
   ```
   PATCH /v1/offerings/{offeringId}
   {
     "auth": {
       "dnaGroupId": "{newDnaGroupId}"
     }
   }
   ```

### 4.4 Build Upload → GSSV Ingestion + Offering Title Update

**Trigger**: `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` fires after a build is published

**Flow**:
1. ServiceBus processor receives publish completion event
2. Check if playtest has `cloudStreamingEnabled` and `offeringId`
3. Call GSSV to ingest the new build:
   ```
   POST /v1/builds/ingest
   {
     "packageId": "{xpackageId}",
     "productId": "{partnerCenterProductId}",
     "offeringId": "{offeringId}"
   }
   ```
4. Once ingested, update the offering's title configuration:
   ```
   PUT /v1/offerings/{offeringId}/titles
   {
     "titles": [{
       "productId": "{partnerCenterProductId}",
       "buildId": "{ingestedBuildId}",
       "platform": "XboxOneAndSeriesX"
     }]
   }
   ```
5. Game is now streamable on the new build

### 4.5 Tester Streaming Experience (Bayside)

**Trigger**: Tester clicks shareable link `https://xbox.com/play/launch/xpt-{sellerId}-{playtestId}`

**Flow**:
1. Bayside route handler receives request
2. **Force login** — redirect to auth if not signed in (no offering details revealed)
3. After auth, call GSSV: `GET /offerings/{offeringId}` with user token
4. GSSV checks DNA group membership for the offering
5. **If authorized**: 
   - Fetch and display playtest metadata (title, art, "This is a non-public playtest build")
   - Show "Stream Now" button
   - On click: initiate streaming session
6. **If not authorized**:
   - Show generic "You don't have access" message
   - No game title, art, or details leaked

### 4.6 Flights / Seller Allowlist

- New PlayTest configuration: `AllowedCloudStreamingSellers` (list of seller IDs)
- Feature flag: `IsCloudStreamingEnabled` (global kill switch)
- On offering creation, check both before proceeding
- Follows existing `Configurations/` patterns in PlayTest Core

---

## 5. Security Considerations

| Risk | Mitigation |
|------|-----------|
| Playtest details leaked to non-members | Bayside enforces auth before revealing ANY offering info |
| Unauthorized streaming access | DNA group checked by GSSV at session start |
| Offering created for unauthorized seller | Flight gate + seller allowlist in PlayTest |
| Stale offering after playtest deletion | Deletion is synchronous; consider async cleanup job as backup |
| Build available before offering is ready | Offering title update only happens AFTER successful ingestion |

---

## 6. Data Model Changes

### PlayTest Cosmos Document (new fields)
```json
{
  "playtestId": "guid",
  "cloudStreamingEnabled": true,
  "offeringId": "xpt-{sellerId}-{playtestId}",
  "streamingLink": "https://xbox.com/play/launch/xpt-...",
  "lastIngestedBuildId": "guid",
  "offeringCreatedAt": "2026-06-15T00:00:00Z"
}
```

### Partner Registry OfferingV2 (leverage existing fields)
- `auth.dnaGroupId` — maps to playtest audience group
- `type` or tag — "playtest" to distinguish from production offerings
- `expiration` — matches playtest end date

---

## 7. Open Questions

| # | Question | Owner | Notes |
|---|----------|-------|-------|
| 1 | Does Partner Registry need a fast-path that skips ADO PR approval for playtest offerings? | Partner Registry team | Success metric requires "within seconds" |
| 2 | What's the exact GSSV build ingestion API? | GSSV team (Anthony Keller?) | Need API spec |
| 3 | How does DNA group auth currently work in GSSV's `/offerings`? | GSSV team | Known issue: DNA group offerings don't show in user's `/offerings` |
| 4 | Should the streaming link use a new Bayside route or the existing offering enrollment flow? | Xbox.JS team | Existing dev mode supports private offerings |
| 5 | What metadata is available for playtest titles in the offering? | xplaytest / Partner Center | Title, art, genre — what's already in Partner Center? |
| 6 | What are the session limits/regions for playtest offerings? | PM (Brian Bowman) | Cost/capacity implications |
| 7 | Can playtest offerings auto-expire with the playtest? | Partner Registry team | Avoid orphaned offerings |

---

## 8. Rollout Plan

1. **Phase 1 (Weeks 5-6)**: PlayTest → Partner Registry integration (create/delete/update offering)
2. **Phase 2 (Week 7)**: ServiceBus build ingestion + offering title update
3. **Phase 3 (Weeks 8-9)**: Bayside playtest link handling + streaming UX
4. **Phase 4 (Week 10)**: DNA group auth fix in GSSV/offerings
5. **Phase 5 (Weeks 11-12)**: Documentation, dashboards, monitoring, TSGs

All phases gated behind seller allowlist. Gradually expand to more sellers as confidence grows.

---

## 9. Dependencies & Risks

| Dependency | Risk | Mitigation |
|-----------|------|-----------|
| GSSV build ingestion API | May not exist yet; need API designed | Prototype with manual offering creation first |
| DNA group in /offerings | Known issue — needs GSSV fix | Work with GSSV team early; may need their help |
| Partner Registry fast-path | PR approval flow could block speed | Discuss with Partner Registry team in Week 3 |
| Bayside routing | New route may need review/approval | Use existing dev mode flow as starting point |
