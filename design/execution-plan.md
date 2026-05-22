# Execution Plan: How This Intern Project Actually Plays Out

> Synthesized from: project spec (Brian's PDF), Jack Heuberger call (xCloud, 2026-05-21 AM), Kushmerick + Retterath + Park onboarding sync (xplaytest + xCloud, 2026-05-21 PM), and 1-2 weeks of repo research.

## Bottom line

- **Ship date**: end of June or early July 2026 public preview (per Retterath, ~6-7 weeks from start)
- **What I'm actually building**: the PlayTest → GSSV integration that lets a creator flip "cloud streaming" on a playtest and have a streamable link work within seconds
- **What I'm NOT building**: GSSV-side changes (Jack + Retterath's team), the rubber-stamp PR workaround (Jack + Timi), the partner-center UI changes (none needed per Jack), the player-facing Bayside UI (already exists in `xbox.com/play`)
- **The critical path is dependency-bound, not implementation-bound**: I can ship the PlayTest side in 2-3 weeks of actual coding. The 6-7 week timeline exists because we're waiting on Jack's TitleIngestion → Offering extension, Jack+Timi's rubber-stamp workaround, and auth onboarding.

---

## The integration in one paragraph

When a creator marks a playtest with `CloudStreaming = true`, PlayTest synchronously calls `PUT /v1/offerings/{id}` on Partner Registry (via Sage in prod, Dev API in dev) to create a private `OfferingV2` whose `AuthorizationOptions.AllowedFlights` are the playtest's resolved DNA group IDs. When a build is later published for that playtest, the existing `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` ServiceBus handler additionally submits a `TitleIngestion` workflow job to GSSV Content Ingestion with the playtest's `PartnerCenterProductId` (= "Big ID"); GSSV ingests the build and joins it server-side to the offering via product ID. The tester clicks a `xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}` link, authenticates with their MSA, GSSV checks flight membership, and the playtest build streams in their browser.

---

## Week-by-week plan

### Week 1 — `wk of 2026-05-19` — **Onboarding + research** ✅ in flight

**What I'm doing**:
- ✅ Research the 3 local repos (`Xbox.Xbet.Service`, `services.partnerregistry`, `Xbox.JS`)
- ✅ Write the field-mapping spec (PlayTest → OfferingV2/Title/Flight)
- ✅ Set up Bruno collection for local Partner Registry probing
- ✅ Capture call notes (Jack, Kushmerick/Retterath/Park)
- 🔄 Get into GSAM Services group (Jack)
- 🔄 Get LinkPad scaffold (Jack)
- 🔄 Get added to xplaytest's shared test account (Emma)
- 🔄 Confirm PlayTest service identity app ID with Brian (for Sage registration)
- 🔄 Try the existing playtest as a tester with Gmail-MSA (David K's suggestion)

**End-of-week success criteria**:
- I can sign into Xbox.com with my Gmail and access the existing playtest as a tester
- LinkPad scaffold exists and at least one GET succeeds against Partner Registry
- I've sent Jack my project doc so he can review and bring back specific feedback

**Risks**: GSAM membership takes longer than a day; LinkPad scaffold not ready by Friday → continue with local-Partner-Registry + Bruno path

---

### Week 2 — `wk of 2026-05-26` — **Auth unblocked + design review + scope decisions**

**What I'm doing**:
- Meet Retterath's incoming PM + Dev → formal design review
- **Lock in 3 scope decisions** that gate Week 3 implementation:
  1. **Single-version vs multi-version playtests for v1**? Retterath's team identified multi-version as a gap. If they can't land cloud-side changes by Week 5, v1 ships single-version-per-playtest.
  2. **One-offering-per-playtest vs shared-offering-with-per-playtest-titles**? Jack's framing hinted both models exist in GSSV. Pick one.
  3. **`sandboxId` for playtest TitleIngestion** — what value does GSSV expect?
- Get Sage route added for `PUT /v1/offerings/{id}` + `DELETE /v1/offerings/{id}` + the TitleIngestion submission route
- First end-to-end auth test: PlayTest service identity → Sage → Partner Registry GET succeeds in test env
- Confirm timeline for Jack+Timi's rubber-stamp PR workaround (best case: ready Week 4-5)

**End-of-week success criteria**:
- Design doc updated with 3 scope decisions locked
- I can hit Partner Registry from a PlayTest-context script through Sage in test env
- Implementation plan for Week 3 is concrete (file paths, classes, methods)

**Risks**: Rubber-stamp workaround timeline pushes right → fallback is manual PR approval for the demo

---

### Week 3 — `wk of 2026-06-02` — **PlayTest → Partner Registry offering CRUD**

**What I'm building** (in `Xbox.Xbet.Service/src/PlayTest`):
- New `IPartnerRegistryClient` / `PartnerRegistryServiceClient` (`ServiceClient` pattern, extends `Shared.Common.ServiceClient`)
- `OfferingV2Builder` that takes a `PlaytestEntity` + resolved DNA group IDs → constructs an `OfferingV2`
- Wire up on `CreatePlaytestAsync` (and `UpdatePlaytestAsync` when streaming is toggled on): synchronously call `PUT /v1/offerings/{xpt-{playtestId}}` if `CloudStreamingEnabled && SellerId ∈ CloudStreamingAllowlistedSellers`
- Wire up on `UpdatePlaytestAsync` audience changes: re-resolve DNA groups → `GET /v1/offerings/{id}` → mutate `AllowedFlights` → `PUT`
- Wire up on `DeletePlaytestAsync`: `DELETE /v1/offerings/{id}`
- Unit tests on the builder; integration test against local-run Partner Registry (Bruno-style)

**End-of-week success criteria**:
- Create a playtest with streaming on → corresponding offering appears in test Partner Registry
- Update audience → AllowedFlights updates
- Delete playtest → offering deleted
- All gated behind allowlist config

**Risks**: Manual PR approval required for every test write — slow but workable

---

### Week 4 — `wk of 2026-06-09` — **TitleIngestion + E2E with ATG sample**

**Prep** (early in week, before any code):
- Clone an ATG sample (VideoTexture per David's recommendation) from `Microsoft/Xbox-GDK-Samples`
- Build once, duplicate the build folder ~3 times
- Edit each copy's `MicrosoftGame.config` to point at distinct test product IDs from our shared test account
- Upload all 3 as playtest products via Partner Center

**What I'm building** (in `Xbox.Xbet.Service/src/PlayTest`):
- New branch in `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor`: if the playtest has `CloudStreamingEnabled`, submit a `TitleIngestion` workflow job to GSSV Content Ingestion
- Use `ProductIngestion.JobParameters` constructor (verified shape from `BulkIngest.razor`): pass `partnerId = SellerId`, `productId = PartnerCenterProductId`, `titleId = ContentEntityIdentifiers.GenerateTitleId(name, ServerPlatform.Xbox)`, `assets = null` (per Jack), `flightId = null` (offering gates), `sandboxId = ?` (Week 2 decision)
- Wrap in `TitleIngestion.JobParameters` with `TitleCollection = null` and `Expiry = playtest.EndDate`
- Submit via `IContentIngestionClient` (NuGet `Microsoft.GameStreaming.ContentIngestion.Client`)
- Log the returned job ID; poll `JobState` periodically OR subscribe to status topic if one exists

**End-of-week success criteria**:
- **The demo**: publish one of my ATG playtests → TitleIngestion completes → my Gmail-MSA tester account can stream it via `xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}` in browser
- This is the first "real proof the integration works"

**Risks**:
- **Jack's TitleIngestion → Offering extension not landed** → fallback: after TitleIngestion completes, do a manual `PUT /v1/offerings/{id}/titles/{titleId}` to attach the title to the offering (two-step)
- **Sandbox question unresolved** → fallback: try `RETAIL` first, then escalate to Jack/Timi if rejected

---

### Week 5 — `wk of 2026-06-16` — **Audience updates + rubber-stamp integration + telemetry**

**What I'm doing**:
- Integrate Jack+Timi's `playtest`-branch workaround when it lands (PlayTest switches from PR-creating client methods to direct-write-to-`playtest`-branch methods)
- Telemetry on: offering create/update/delete latency, TitleIngestion job duration, audience-change-to-stream-eligibility time
- Test the audience-change path with actual DNA group changes
- Bug fixing from Week 4 E2E

**End-of-week success criteria**:
- "Within seconds of creating a playtest, streamable link is live" — verified with timing data
- Rubber-stamp workaround integrated (or fallback documented for demo)

**Risks**: Workaround still not ready → demo with manual PR approval, treat as a known issue for v1.1

---

### Week 6 — `wk of 2026-06-23` — **Pre-PP hardening + bug bash**

**What I'm doing**:
- Widen `CloudStreamingAllowlistedSellers` to 2-3 trusted sellers
- Polish: error states for non-allowlisted users, link generation in PlayTestFD response, Bayside UX checks
- Verify: group enumeration delays don't blow the "within seconds" success metric (Kushmerick called this out as a known performance pain point)
- Documentation: runbook, on-call playbook, post-mortem template
- Bug bash with the team

**End-of-week success criteria**: ready for public preview

---

### Week 7 — `wk of 2026-06-30` — **🎯 PUBLIC PREVIEW**

Ship.

---

### Week 8+ — `Jul-Aug` — **Hardening + stretch goals**

- Multi-version support (if scoped out of v1)
- Garrison / Bastion stretch goals (PlayTest streaming from console clients, not just web)
- Migrate from `CloudStreamingAllowlistedSellers` to broader rollout flighting
- Possible: tester feedback collection pipeline
- Intern-exit doc, knowledge transfer to xplaytest team

---

## Dependency graph

```
┌──────────────────────────────────────────────────────────────────────┐
│ GSSV-side dependencies (I don't own — I wait on)                      │
│                                                                       │
│   Jack: GSAM membership  ─┐                                           │
│   Jack: Sage route + app  ├─→ unblocks my auth → unblocks Week 3      │
│   Jack: LinkPad scaffold  ─┘                                          │
│                                                                       │
│   Jack: TitleIngestion → Offering extension  ─→ unblocks Week 4 E2E   │
│         (fallback: 2-step manual PUT)                                 │
│                                                                       │
│   Jack+Timi: `playtest`-branch rubber-stamp workaround  ─→ Week 5     │
│         (fallback: manual PR approval for demo)                       │
│                                                                       │
│   Retterath's team: multi-playtest-version support  ─→ Week 8+        │
│         (fallback: v1 is single-version-per-playtest)                 │
│                                                                       │
│   Retterath's team: assign ongoing PM + Dev contact  ─→ Week 2         │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ PlayTest-side dependencies (I own)                                    │
│                                                                       │
│   Week 1: Onboarding + research + spec                                │
│   Week 2: Auth working + design decisions locked                      │
│   Week 3: Offering CRUD (create / update / delete)                    │
│   Week 4: TitleIngestion submission + E2E with ATG sample             │
│   Week 5: Audience updates + rubber-stamp integration + telemetry     │
│   Week 6: Hardening, widen allowlist, bug bash                        │
│   Week 7: 🎯 Public Preview                                            │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Risk register

| Risk | Likelihood | Impact | Mitigation / fallback |
|---|---|---|---|
| Auth onboarding (GSAM + Sage + app ID) takes > 1 week | Medium | High — blocks all hosted-env testing | Run Partner Registry locally with `ASPNETCORE_ENVIRONMENT=Development` (Bruno collection ready). Code/test against localhost; deploy-test in Week 2-3 once auth lands. |
| Rubber-stamp PR workaround slips past Week 5 | Medium | High — blocks prod launch but not demo | Manual PR approval for demo; ship public preview with known-issue. Customer-impact is "first audience-change has a manual approval delay" which is tolerable for preview. |
| Jack's TitleIngestion → Offering extension slips past Week 4 | Medium | Medium — blocks clean E2E | Two-step fallback: submit TitleIngestion, then on completion do a manual `PUT /v1/offerings/{id}/titles/{titleId}`. Slightly more code but works. |
| Multi-version GSSV work doesn't ship by Week 6 | High | Low — for v1 | Scope-down v1 to single-version-per-playtest. Document the limitation. Multi-version becomes v1.1. |
| ATG sample build fails / GameConfig editing breaks | Low | Medium — blocks E2E | Pair with David Kushmerick (he's been doing exactly this); fall back to using one of the existing test products from the shared test account that already has a real build. |
| `assets: null` doesn't work for unpublished playtest BigIds (open question §5a) | Medium | High — blocks ingestion entirely | Escalate to Jack + Timi (Timi owns the Microsoft Store integration). If GSSV can't accept unpublished BigIds, this is a deeper GSSV gap that pushes the deadline. |
| `sandboxId` value rejected | Low | Medium | Iterate via LinkPad — Jack to help. |
| Audience propagation (group enumeration delay) breaks "within seconds" SLA | Medium | Medium | Measure in Week 5; if it's the bottleneck, ship Week 7 with a relaxed SLA ("within a minute") and pursue group-side optimization in v1.1. |

---

## Recurring weekly cadence (recommended)

- **Mondays AM**: check in with Jack on GSSV-side progress (rubber-stamp + extension + multi-version). 15 min.
- **Tuesdays + Wednesdays**: implementation focus blocks.
- **Wednesdays PM**: cross-team sync with Retterath's PM + Dev (starting Week 2).
- **Thursdays**: implementation + testing.
- **Fridays AM**: weekly status to Brian + post to the project channel. Update `plan.md` and `design/execution-plan.md` (this doc) with what shifted.

---

## People to know — and who to talk to about what

| Person | Team | Talk to them about |
|---|---|---|
| **Brian Bowman** | xplaytest (manager) | Scope decisions, project blockers escalation, PlayTest service identity app ID |
| **Emma Park** | xplaytest (onboarding lead) | Test account access, Partner Center onboarding, Garrison setup |
| **David Kushmerick** ("Kush") | xplaytest engineer | ATG sample workflow, GameConfig editing, shadow publishing pipeline, group enumeration delays, the playtest test pipeline |
| **David Retterath** | xCloud | Offerings ↔ DNA groups ↔ titles model, version sourcing from X product, multi-version support, ongoing GSSV contact assignment |
| **Jack Heuberger** | xCloud (built TitleIngestion as his intern project) | TitleIngestion contract, TitleIngestion → Offering extension, rubber-stamp workaround, Dev API / Sage / GSAM / LinkPad onboarding, `assets`/`sandboxId`/`flightId` questions |
| **Timi Bolaji** | xCloud | Microsoft Store integration, rubber-stamp workaround, sandbox / BigId edge cases |
| **Anthony Keller** | xCloud | S2S allowlists (`RegistryManagementConfig.AllowedS2SAppIds`), original onboarding contact |
| **Retterath's PM + Dev** (TBD) | xCloud | Joining Week 2 for ongoing collaboration — design review, GSSV-side prioritization |

---

## Definition of "done" for public preview (end June / early July)

A PM, sitting in Partner Center:
1. Creates a playtest for one of the shared-test-account products
2. Sets cloud streaming = true
3. Adds testers via DNA group
4. Publishes a build (using an ATG VideoTexture variant as the test package)
5. Gets a shareable link

A tester, on a laptop with no Xbox installed:
1. Clicks the link
2. Signs in with their Gmail-MSA
3. Stream starts in their browser within seconds
4. They play the build

That's the demo. Everything else is gravy.

---

## What could go right (best case)

If everything lands:
- Week 4 E2E works with ATG sample → recorded demo for stakeholders
- Week 5 rubber-stamp workaround lands → no manual PR drudgery
- Week 6 widened allowlist exercises 2-3 real seller flows
- Week 7 public preview ships clean with multi-second SLA met
- Week 8+ multi-version support lands and we extend to v1.1
- Stretch: Garrison streaming entry point lands by end of internship

## What could go wrong (worst plausible case)

- Auth onboarding takes 2 weeks (not 1) → Week 3 slides to Week 4
- Rubber-stamp workaround doesn't ship → demo uses manual approval, public preview pushed to Week 8
- Single-version-per-playtest documented as known limitation
- Result: still ships, just one week late and with a known-issues list

Either way, the project gets to "real users can stream private playtests" — which is the success metric.
