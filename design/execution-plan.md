# Execution Plan: How This Intern Project Actually Plays Out

> **Canonical reference**: [`research/project-spec.md`](../research/project-spec.md) (synthesized from Brian's PDF — the official project plan).
> **Synthesized from**: project spec PDF, Jack Heuberger call (xCloud, 2026-05-21 AM), Kushmerick + Retterath + Park onboarding sync (xplaytest + xCloud, 2026-05-21 PM), and 1-2 weeks of repo research.

## Bottom line

- **Internship duration**: ~12 weeks (per official PDF), running through end-of-August.
- **Public preview target**: end of June / early July 2026 (per Retterath) — this aligns with the **Midpoint Connect (6/30) at end of Core Service Work, Weeks 5-7**. PP is the project **midpoint**, not the end.
- **After PP**, ~5 more weeks: Frontend Work (Weeks 8-10) + Documentation/Metrics/Polish (Weeks 11-12) + Final Presentation.
- **What I'm building (P0)**: PlayTest → GSSV integration (offering CRUD + TitleIngestion submission) + xplaytest portal toggle UI + Bayside link-handling polish.
- **What I'm NOT building**: GSSV-side TitleIngestion → Offering extension (Jack), the rubber-stamp PR workaround (Jack + Timi), multi-version GSSV (Retterath's team).
- **The critical path through PP is dependency-bound, not implementation-bound**: I can ship the PlayTest side in 2-3 weeks of actual coding. The compressed PP timeline exists because we're waiting on Jack's TitleIngestion → Offering extension, Jack+Timi's rubber-stamp workaround, and auth onboarding.

---

## The integration in one paragraph

When a creator marks a playtest with `CloudStreaming = true`, PlayTest synchronously calls `PUT /v1/offerings/{id}` on Partner Registry (via Sage in prod, Dev API in dev) to create a private `OfferingV2` whose `AuthorizationOptions.AllowedFlights` are the playtest's resolved DNA group IDs. When a build is later published for that playtest, the existing `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` ServiceBus handler additionally submits a `TitleIngestion` workflow job to GSSV Content Ingestion with the playtest's `PartnerCenterProductId` (= "Big ID"); GSSV ingests the build and joins it server-side to the offering via product ID. The tester clicks a `xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}` link, authenticates with their MSA, GSSV checks flight membership, and the playtest build streams in their browser.

---

## Official 12-week timeline (from PDF)

| Weeks | Phase | Deliverable | Checkpoint |
|---|---|---|---|
| **1-2** | Onboarding + Welcome (Agency / Copilot CLI ramp-up) | Productive on stack; understand project | **First Connect — 6/2** |
| **3-4** | Design Doc + Prototyping / Manual Testing | Reviewed design doc; detailed engineering estimates | — |
| **5-7** | **Core Service Work** (GSSV + xplaytest + CAS) | Services work complete | **Midpoint Connect — 6/30** (← **Public Preview target**) |
| **8-10** | **Front End Work** | Bayside player updates + xplaytest portal updates (React, Tailwind) | **Final Connect — 7/28** |
| **11-12** | Documentation, Metrics, Polish | Dashboards, monitors, TSGs | **Final Project Presentation** |

---

## Week-by-week plan

### Phase 1 — Onboarding (Weeks 1-2)

#### Week 1 — `wk of 2026-05-19` — **Onboarding + research** ✅ in flight

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
- 🔄 Check whether `services.permissions` (listed in PDF resources) is an additional touch point I should clone

**End-of-week success criteria**:
- I can sign into Xbox.com with my Gmail and access the existing playtest as a tester
- LinkPad scaffold exists and at least one GET succeeds against Partner Registry
- I've sent Jack my project doc so he can review and bring back specific feedback

**Risks**: GSAM membership takes longer than a day; LinkPad scaffold not ready by Friday → continue with local-Partner-Registry + Bruno path.

---

#### Week 2 — `wk of 2026-05-26` — **Auth unblocked + First Connect prep**

**What I'm doing**:
- Meet Retterath's incoming PM + Dev → first design-review pass
- First end-to-end auth test: PlayTest service identity → Sage → Partner Registry GET succeeds in test env
- Get Sage routes added for `PUT /v1/offerings/{id}`, `DELETE /v1/offerings/{id}`, and the TitleIngestion submission route
- Confirm timeline for Jack+Timi's rubber-stamp PR workaround (best case: ready Week 4-5)
- Prep for **First Connect (6/2)** with Brian: synthesize what I've learned, articulate the design space, lay out the 3 open scope decisions

**End-of-week success criteria**:
- I can hit Partner Registry from a PlayTest-context script through Sage in test env
- First Connect deck/notes ready
- Have a working LinkPad script that exercises offering CRUD against dev

**🔵 First Connect — 6/2** (Impact Conversation with Brian)

---

### Phase 2 — Design Doc + Prototyping (Weeks 3-4)

**Deliverable per PDF**: *"Reviewed Design Doc of E2E work; detailed engineering estimates established"*

#### Week 3 — `wk of 2026-06-02` — **Lock the design + prototype the integration**

**What I'm doing**:
- **Lock in the 3 scope decisions** that gate Core Service Work:
  1. **Single-version vs multi-version playtests for v1**? Retterath's team identified multi-version as a gap. If they can't land cloud-side changes by Week 5-6, v1 ships single-version-per-playtest.
  2. **One-offering-per-playtest vs shared-offering-with-per-playtest-titles**? Jack's framing hinted both models exist in GSSV. Pick one.
  3. **`sandboxId` for playtest TitleIngestion** — what value does GSSV expect?
- Update `design/design-doc.md` with decisions, locked sequence diagrams, and concrete file/class/method targets in `Xbox.Xbet.Service/src/PlayTest`
- Build a **manual end-to-end prototype** through LinkPad: spin up a fake "playtest" → create offering → submit TitleIngestion → stream from `xbox.com/play` against a sandbox product. Proves the integration is achievable before Core Service Work begins.
- Detailed engineering estimates per P0 item

**End-of-week success criteria**:
- Design doc rev'd and circulated to Brian / Kushmerick / Bec / Retterath's PM
- Prototype works end-to-end (even if held together by tape)
- Engineering estimates for every P0 item

---

#### Week 4 — `wk of 2026-06-09` — **Design review + ATG sample prep**

**What I'm doing**:
- Design review meetings with xplaytest leads (Kushmerick + Bec) and xCloud (Retterath's PM + Dev + Jack)
- Incorporate feedback into design doc
- **ATG sample prep** (David K's recipe):
  - Clone `Microsoft/Xbox-GDK-Samples` and build the VideoTexture sample
  - Duplicate the build folder ~3 times; edit each copy's `MicrosoftGame.config` to point at distinct test product IDs from our shared test account
  - Upload all 3 as playtest products via Partner Center
- Validate the **decision register** is signed off by EOW (no more design changes after this — Week 5 starts Core Service Work)

**End-of-week success criteria**:
- Reviewed design doc signed off
- 3 ATG playtest packages exist and are uploadable
- Implementation plan for Week 5 is concrete (file paths, classes, methods)

**Risks**: Design review uncovers a major architectural gap → may slide implementation to Week 5b/6.

---

### Phase 3 — Core Service Work (Weeks 5-7) — 🎯 **PP target**

**Deliverable per PDF**: *"Services work complete across: GSSV, xplaytest, CAS?, etc."*

#### Week 5 — `wk of 2026-06-16` — **PlayTest → Partner Registry offering CRUD**

**What I'm building** (in `Xbox.Xbet.Service/src/PlayTest`):
- New `IPartnerRegistryClient` / `PartnerRegistryServiceClient` (`ServiceClient` pattern, extends `Shared.Common.ServiceClient`)
- `OfferingV2Builder` that takes a `PlaytestEntity` + resolved DNA group IDs → constructs an `OfferingV2`
- Wire up on `CreatePlaytestAsync` (and `UpdatePlaytestAsync` when streaming is toggled on): synchronously call `PUT /v1/offerings/{xpt-{playtestId}}` if `CloudStreamingEnabled && SellerId ∈ CloudStreamingAllowlistedSellers`
- Wire up on `UpdatePlaytestAsync` audience changes: re-resolve DNA groups → `GET /v1/offerings/{id}` → mutate `AllowedFlights` → `PUT`
- Wire up on `DeletePlaytestAsync`: `DELETE /v1/offerings/{id}`
- Unit tests on the builder; integration test against local-run Partner Registry (Bruno-style)

**End-of-week success criteria** (covers **P0.1**, **P0.2**, **P0.4**, **P0.5**, **P0.6**):
- Create a playtest with streaming on → corresponding offering appears in test Partner Registry
- Update audience → AllowedFlights updates
- Delete playtest → offering deleted
- All gated behind seller allowlist config

---

#### Week 6 — `wk of 2026-06-23` — **TitleIngestion + E2E with ATG sample**

**What I'm building** (in `Xbox.Xbet.Service/src/PlayTest`):
- New branch in `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor`: if the playtest has `CloudStreamingEnabled`, submit a `TitleIngestion` workflow job to GSSV Content Ingestion
- Use `ProductIngestion.JobParameters` constructor (verified shape from `BulkIngest.razor`): `partnerId = SellerId`, `productId = PartnerCenterProductId`, `titleId = ContentEntityIdentifiers.GenerateTitleId(name, ServerPlatform.Xbox)`, `assets = null` (per Jack), `flightId = null` (offering gates), `sandboxId = ?` (Week 3 decision)
- Wrap in `TitleIngestion.JobParameters` with `TitleCollection = null` and `Expiry = playtest.EndDate`
- Submit via `IContentIngestionClient`; log returned job ID; poll `JobState` periodically OR subscribe to status topic if one exists
- **Catalog / CAS / Hydration touch point** (Outcome #2 in PDF): if Content Access changes are needed to surface private offering metadata to Bayside, scope and land them this week

**End-of-week success criteria** (covers **P0.9**, **P0.10**, partial **P1**):
- **The demo**: publish one of my ATG playtests → TitleIngestion completes → my Gmail-MSA tester account can stream it via `xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}` in browser
- This is the first "real proof the integration works"

**🔵 Midpoint Connect — 6/30** (Impact Conversation with Brian — **Public Preview target**)

---

#### Week 7 — `wk of 2026-06-30` — **PP hardening + rubber-stamp integration**

**What I'm doing**:
- Integrate Jack+Timi's `playtest`-branch workaround when it lands (PlayTest switches from PR-creating client methods to direct-write-to-`playtest`-branch methods)
- Telemetry: offering create/update/delete latency, TitleIngestion job duration, audience-change-to-stream-eligibility time
- Verify: group enumeration delays don't blow the "within seconds" success metric (Kushmerick called this out as a known performance pain point)
- Test the audience-change path with actual DNA group changes
- Widen `CloudStreamingAllowlistedSellers` to 2-3 trusted sellers
- Bug bash with the team
- **🎯 Ship Public Preview** by end of week

**End-of-week success criteria**:
- "Within seconds of creating a playtest, streamable link is live" — verified with timing data
- Rubber-stamp workaround integrated (or fallback: manual approval documented as known issue)
- **Public Preview goes live**

**Risks**: Workaround still not ready → ship PP with manual PR approval as a known issue for v1.1.

---

### Phase 4 — Frontend Work (Weeks 8-10)

**Deliverable per PDF**: *"Bayside player updates + xplaytest portal updates (React, Tailwind)"*

> This is the phase I'd previously been missing. The PDF's "Skills" section explicitly calls out *"Full Stack & Front End Development (React, Tailwind)"*. The frontend is half the project — Core Service Work makes streaming **work**; Frontend Work makes it **usable**.

#### Week 8 — `wk of 2026-07-07` — **xplaytest portal updates**

**What I'm building** (xplaytest portal — wherever it lives; confirm with Kushmerick/Bec at start of Week 8):
- **Cloud streaming toggle UI** on the playtest create/edit form
- **Shareable link** display + copy-to-clipboard on the playtest details page (covers **P0.7**)
- Status indicator: "ready to stream" / "ingesting build" / "ingestion failed" — surfaces TitleIngestion job state from the PlayTest core (covers P0.9 stretch goal: *"user feedback about when build is 'ready for testing'"*)
- Error / disabled states for non-allowlisted sellers

**End-of-week success criteria**:
- Creator can flip "cloud streaming" toggle in the portal
- Shareable link appears + copies correctly
- Build-status indicator updates after TitleIngestion completes

---

#### Week 9 — `wk of 2026-07-14` — **Bayside player updates**

**What I'm building** (`Xbox.JS/apps/play-xbox`):
- **Link-handling polish**: when a tester clicks `xbox.com/play/stream/{productId}?configs=streaming.auth.offering:{offeringId}`, ensure the auth → offering-membership-check → stream flow is smooth (covers **P0.8**)
- Verify Bayside's existing `isPrivate` redirect (`loader.server.ts:28-83`) doesn't leak playtest title metadata pre-auth
- **"Non-public version" badging**: testers know they're streaming a non-public build (per North-Star step 6)
- Surface as much metadata as available (title, genre, art) from the playtest/store (per North-Star step 6)
- **"Install locally or stream now"** affordance if applicable (per North-Star step 7)

**End-of-week success criteria**:
- Tester clicks link → lands in Bayside → auths → streams; no metadata leak
- Tester clearly sees "this is a playtest build, not retail"
- All P0.7 + P0.8 + P1 (Bayside login → stream) covered

---

#### Week 10 — `wk of 2026-07-21` — **Frontend polish + Final Connect prep**

**What I'm doing**:
- Cross-browser/device testing on Bayside changes
- Accessibility pass on the portal toggle + status indicator
- Final design polish based on UX feedback from the team
- E2E flow demo recording for **Final Connect**

**🔵 Final Connect — 7/28** (Impact Conversation with Brian)

---

### Phase 5 — Documentation, Metrics, Polish (Weeks 11-12)

**Deliverable per PDF**: *"Dashboards, monitors, TSGs"*

#### Week 11 — `wk of 2026-07-28` — **Dashboards + monitors + TSGs**

**What I'm doing**:
- **Dashboards**: Grafana/Kusto dashboards covering offering CRUD latency, TitleIngestion success rate + duration, stream session start rate, error breakdown
- **Monitors / alerts**: page-out alerts for offering CRUD failures, TitleIngestion failure spikes, rubber-stamp workaround backlog (if relevant)
- **TSGs (Troubleshooting Guides)**: on-call runbooks for "playtest streaming not available", "ingestion stuck", "audience changes not propagating"
- **Engineering Excellence pass**: code review feedback, refactors, increase test coverage where thin

---

#### Week 12 — `wk of 2026-08-04` — **Final Presentation prep + knowledge transfer**

**What I'm doing**:
- **Final Project Presentation** prep (date TBD by UR Champs / Ellery Charlson — likely week of 8/11)
- **Knowledge transfer doc** to xplaytest team — covers everything in this repo, plus operational handoffs
- Migrate documentation from this personal repo into the team wiki / ADO
- Address any "v1.1" backlog items if time permits:
  - Multi-version support (if Retterath's team landed it)
  - Bastion / Garrison "stream now" button (P3)
  - Launch args (P3)
  - Feedback collection (P3)

**End of internship**: ~end of August

---

## Dependency graph

```
┌──────────────────────────────────────────────────────────────────────┐
│ GSSV-side dependencies (I don't own — I wait on)                      │
│                                                                       │
│   Jack: GSAM membership  ─┐                                           │
│   Jack: Sage route + app  ├─→ unblocks my auth → unblocks Week 3 LP   │
│   Jack: LinkPad scaffold  ─┘                                          │
│                                                                       │
│   Jack: TitleIngestion → Offering extension  ─→ unblocks Week 6 E2E   │
│         (fallback: 2-step manual PUT)                                 │
│                                                                       │
│   Jack+Timi: `playtest`-branch rubber-stamp workaround  ─→ Week 7     │
│         (fallback: manual PR approval for PP launch)                  │
│                                                                       │
│   Retterath's team: multi-playtest-version support  ─→ Week 11+       │
│         (fallback: v1 is single-version-per-playtest)                 │
│                                                                       │
│   Retterath's team: assign ongoing PM + Dev contact  ─→ Week 2-3      │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│ PlayTest-side dependencies (I own)                                    │
│                                                                       │
│   Wk 1-2:  Onboarding + research + auth working + First Connect       │
│   Wk 3-4:  Design doc + prototyping + ATG sample prep                 │
│   Wk 5:    Offering CRUD (create / update / delete)                   │
│   Wk 6:    TitleIngestion submission + E2E + CAS/Hydration            │
│   Wk 7:    🎯 Public Preview                                           │
│   Wk 8:    xplaytest portal updates (toggle, link, status)            │
│   Wk 9:    Bayside player updates (link handling, badging)            │
│   Wk 10:   Frontend polish + Final Connect                            │
│   Wk 11:   Dashboards + monitors + TSGs                               │
│   Wk 12:   Final Presentation prep + knowledge transfer               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Risk register

| Risk | Likelihood | Impact | Mitigation / fallback |
|---|---|---|---|
| Auth onboarding (GSAM + Sage + app ID) takes > 1 week | Medium | High — blocks Week 3 prototyping | Run Partner Registry locally with `ASPNETCORE_ENVIRONMENT=Development` (Bruno collection ready). Code/test against localhost; deploy-test in Week 3-4 once auth lands. |
| Rubber-stamp PR workaround slips past Week 7 | Medium | High — blocks prod launch but not PP demo | Manual PR approval for PP; ship public preview with known-issue. Customer-impact is "first audience-change has a manual approval delay" which is tolerable for preview. |
| Jack's TitleIngestion → Offering extension slips past Week 6 | Medium | Medium — blocks clean E2E | Two-step fallback: submit TitleIngestion, then on completion do a manual `PUT /v1/offerings/{id}/titles/{titleId}`. Slightly more code but works. |
| Multi-version GSSV work doesn't ship by Week 11 | High | Low — for v1 | Scope-down v1 to single-version-per-playtest. Document the limitation. Multi-version becomes v1.1. |
| ATG sample build fails / GameConfig editing breaks | Low | Medium — blocks E2E | Pair with David Kushmerick (he's been doing exactly this); fall back to using one of the existing test products from the shared test account that already has a real build. |
| `assets: null` doesn't work for unpublished playtest BigIds (open question §5a) | Medium | High — blocks ingestion entirely | Escalate to Jack + Timi (Timi owns the Microsoft Store integration). If GSSV can't accept unpublished BigIds, this is a deeper GSSV gap that pushes the PP deadline. |
| `sandboxId` value rejected | Low | Medium | Iterate via LinkPad — Jack to help. |
| Audience propagation (group enumeration delay) breaks "within seconds" SLA | Medium | Medium | Measure in Week 7; if it's the bottleneck, ship PP with a relaxed SLA ("within a minute") and pursue group-side optimization in v1.1. |
| Frontend work greenfield vs modify-existing surprise in Week 8 | Medium | Medium | Validate Week 4-5 with Kushmerick/Bec — confirm where the xplaytest portal lives and how much UI exists today. Adjust Weeks 8-10 scope accordingly. |
| `services.permissions` repo is actually a required touch point | Low | Low | Clone Week 1; if it surfaces as a real dep, fold into Week 5 plan. |

---

## Recurring weekly cadence (recommended)

- **Mondays AM**: check in with Jack on GSSV-side progress (rubber-stamp + extension + multi-version). 15 min.
- **Tuesdays + Wednesdays**: implementation focus blocks.
- **Wednesdays PM**: cross-team sync with Retterath's PM + Dev (starting Week 2).
- **Thursdays**: implementation + testing.
- **Fridays AM**: weekly status to Brian + post to the project channel. Update `plan.md` and `design/execution-plan.md` (this doc) with what shifted.

---

## People to know — and who to talk to about what

> Verbatim role labels from PDF where applicable. See [`research/project-spec.md`](../research/project-spec.md) §Key contacts for the full canonical list.

| Person | Role (PDF) | Talk to them about |
|---|---|---|
| **Brian Bowman** | Intern Manager | Scope decisions, project blockers escalation, PlayTest service identity app ID, **Connects** |
| **Aditya Toney** | Onboarding Buddy | Initial setup, equipment, environment problems |
| **Emma Park** | Intern Mentor | Day-to-day support, test account access, Partner Center onboarding, Garrison setup |
| **Jessie Masih** | Administrative Assistant | Workplace resources, office setup, events |
| **David Kushmerick** + **Bec Lyons** | Feature Team Leader (jointly) | Feature accountability, design review, ATG sample workflow (Kushmerick), shadow publishing pipeline, group enumeration delays, xplaytest portal UI scoping |
| **Anthony Keller** | Engineering Expert | S2S allowlists (`RegistryManagementConfig.AllowedS2SAppIds`), original onboarding contact |
| **Timi Bolaji** | Engineering Expert | Microsoft Store integration, rubber-stamp workaround, sandbox / BigId edge cases |
| **Ashton Summer**, **Chuy Galvan** | Engineering Experts | Additional engineering experts — to be brought in as their areas surface |
| **Ellery Charlson** | UR Champ | Program-wide events, Final Presentation scheduling, things the team can't help with |
| **David Retterath** | xCloud (committed PM + Dev to project) | Offerings ↔ DNA groups ↔ titles model, version sourcing from X product, multi-version support, ongoing GSSV contact assignment |
| **Jack Heuberger** | xCloud (built TitleIngestion as his intern project) | TitleIngestion contract, TitleIngestion → Offering extension, rubber-stamp workaround, Dev API / Sage / GSAM / LinkPad onboarding, `assets`/`sandboxId`/`flightId` questions |
| **Retterath's incoming PM + Dev** (TBD) | xCloud | Joining Week 2 for ongoing collaboration — design review, GSSV-side prioritization |

---

## Definition of "done" for Public Preview (end of Week 7 — 6/30 to 7/4)

A PM, sitting in xplaytest portal:
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

That's the PP demo. Frontend polish + dashboards + presentation come after.

---

## Definition of "done" for end of internship (Week 12 — ~early August)

- All P0 goals shipped (P0.1–P0.10)
- P1 goals shipped (CAS/GSSV → Bayside playtest discovery + login-before-reveal)
- xplaytest portal has cloud-streaming toggle + shareable link + status indicator
- Bayside link-handling is smooth + non-leaky + clearly badged as "non-public build"
- Dashboards live; monitors alerting; TSGs published
- Knowledge transfer doc handed to xplaytest team
- Final Presentation delivered

---

## What could go right (best case)

- Week 6 E2E works with ATG sample → recorded demo for stakeholders
- Week 7 rubber-stamp workaround lands → no manual PR drudgery in PP
- Week 7 PP ships clean with multi-second SLA met
- Week 8-10 frontend lands smoothly because the design was scoped in Week 4
- Week 11+ multi-version support lands and we extend to v1.1
- Stretch: Garrison streaming entry point lands by Week 12

## What could go wrong (worst plausible case)

- Auth onboarding takes 2 weeks (not 1) → Week 5 slides to Week 6
- Rubber-stamp workaround doesn't ship by PP → demo with manual approval, prod launch pushed to Week 8-9
- Frontend work in Weeks 8-10 turns out to be greenfield (not modify-existing) → only one of the two surfaces (portal OR Bayside) ships polished
- Result: still ships everything, just with some surfaces less polished + known-issues list

Either way, the project gets to "real users can stream private playtests" — which is the success metric.
