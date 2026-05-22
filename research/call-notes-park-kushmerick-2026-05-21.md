# Call notes — xplaytest + xCloud onboarding sync, 2026-05-21 (afternoon)

> **Participants** (per Melanie's confirmation 2026-05-21):
> - **David Kushmerick** ("Kush" / "Kushi") — **xplaytest** team
> - **David Retterath** — **xCloud** team
> - **Emma Park** — coordinating Melanie's onboarding (likely xplaytest given the test-account ownership; confirm role later)
> - **Melanie Chen**
>
> *Earlier draft of this file incorrectly identified David Kushmerick as the xCloud dev. The chat shorthand "kush (xplaytest tpm) and david (xcloud dev)" actually referred to two different people: Kush = Kushmerick (xplaytest), and David = Retterath (xCloud).*

## TL;DR

- **Public preview target: end of June or early July 2026** — this is **the** deadline. Concrete and aggressive (~6-7 weeks from today). The current design-doc timeline (Week 11-12 = mid-Dec) needs compression.
- **GSSV-side gap (new)**: today GSSV pulls "what version to stream" live from the X product entry (= latest-ingested-for-productId). For playtests we likely need **multiple versions in flight simultaneously** (different audience groups on different builds) — GSSV doesn't support this yet. **Cloud-side changes needed.** Add to open questions.
- **New architectural concept from xplaytest side**: "**shadow publishing pipeline**" — the parallel path xplaytest builds take through Xbox publishing infra (mirroring real retail publishing but scoped to the playtest audience). Need to learn more.
- **Next week**: a PM + Dev from David Retterath's xCloud team will join for deeper collaboration → finally a named ongoing GSSV-side contact pair.
- **Streaming's value vs install** (and why this project matters): instant access **+** avoids conflicts with the tester's installed retail copy of the same game. Stronger rationale than I had documented.
- **Known performance pain points** in xplaytest today: group enumeration delays + lack of delta uploads for playtest builds. Not directly our problem but useful context.
- **Onboarding workflow** (Emma + Kushmerick): Partner Center access + shared team test account + ATG samples for package creation + Garrison testing path.

---

## Meeting recap — by topic

### 1. Playtest system overview & technical challenges (Kushmerick, ~8:18)
Kushmerick walked through the playtest system architecture and its Xbox-services integration. Key concepts:

- **Shadow publishing pipeline** — the parallel path that playtest builds take through Xbox publishing infrastructure. Mirrors real retail publishing but scoped to the playtest audience. This is presumably what `XPackagePlaytestPublishWorkflow` plugs into, and explains why playtest publishing exists as a separate workflow from regular store publishing.
- **Group enumeration delays** — when a creator adds testers to a playtest audience, those users don't immediately see access. There's a propagation delay. Worth flagging as a constraint for the *"streamable link within seconds of toggling cloud streaming on"* success metric — the audience-side propagation may dominate the offering-side latency.
- **Delta uploads for playtest builds** — Kushmerick wants incremental package uploads (vs re-uploading the whole build for each change). AAA build sizes make full re-uploads painful. Tangential to our project but relevant context for the broader xplaytest roadmap.

### 2. Streaming integration & project goals (~4:57)
Melanie's project aims to connect playtest builds with xCloud, enabling streamable links so testers can play without downloading.

- **Both streaming AND install will be supported** — not a replacement. Toggle "cloud streaming" on a playtest gives an *additional* shareable link; the existing install/sideload path keeps working.
- **Streaming's UX advantages** (Kushmerick's framing):
  - **Instant access** — no multi-GB download wait
  - **No retail-install conflict** — testers may already have the retail copy of the same game installed; installing a playtest build of the same product can collide / require uninstall. Streaming sidesteps this entirely. **This is a strong concrete value prop** — worth adding to the design-doc's "Why" section.

### 3. User access & group management (~28:05)
- Testers are added to audience groups using **MSAs** (Microsoft accounts)
- Personal accounts or burner accounts both work
- **Gmail addresses are valid MSAs** — confirmed. Melanie's Gmail-linked Xbox account should already be able to access an existing playtest today for tester-UX exploration.

### 4. Offering & version management (Retterath, ~23:37) — NEW
The xCloud side of the contract. Most of this is new to my docs:

- *"**Offerings are tied to DNA groups and titles**"* — matches our `OfferingV2.AuthorizationOptions.AllowedFlights` + `Title` mapping. Verified.
- *"**Versions sourced live from X product**"* — at session-creation time, GSSV asks the upstream X product service for the current ingested version for the requested productId, and serves that. This is why `Title.cs` has no `BuildId` field — version resolution is *runtime*, not config. Confirms my earlier inference.
- **Multi-version support gap**: *"the need for cloud-side changes to support multiple playtest versions was identified"* — today GSSV serves the latest-ingested version per productId. For playtests, we likely want **multiple concurrent versions** of the same product (different audience groups on different builds; A/B testing two builds; etc.). **GSSV doesn't support this today — it's new cloud-side work.** Add to open questions §5i.
- **Plans to assign a regular contact from Retterath's team** for ongoing support — concrete commitment. Next-week PM + Dev join (item 5 below) is the start of this.

### 5. Timeline & next steps (~20:57)
- **Public preview target: end of June or early July 2026** — confirmed deadline
- AI tools will be used to accelerate progress
- **Next week**: a PM and Dev from David Retterath's team will join for deeper collaboration

### 6. Onboarding & testing guidance (Emma + Kushmerick, ~31:08)
Already captured in detail from the partial transcript:

- Add Melanie to shared test account → can use the team's existing test products
- Partner Center access for publishing
- **ATG samples + GameConfig editing** as a test-package recipe (see verbatim quotes below — this is the actionable bit)
- Test in Garrison

---

## Verbatim — David Kushmerick on test package creation

> *"I went and downloaded one of the ATG samples from GitHub. And I chose this one, this video texture, just because it's kind of pretty. And then what I've done is — I've modified, I go into the [build folder] after you compile it, you get the built version of it, and I just made copies in here, and the only difference between all these is these are just different products, so that when I go into the Game Config for that title, I just changed the name… The name and publisher… as well as this guy here and this guy there. Well, and these can [be] basically anything that's got an ID, right? Basically the MSA app ID, the tile ID, the store ID. But this is how I basically can iterate on a game package by just using this ATG sample, and then I can publish… publish it to any product that I want."*

> *"To get this working right, we're going to have to upload a package and that package is going to have to make its way all the way over to the xCloud team, so it can be ingested into the xCloud distribution Services."*

> *"Since you're going to have to create a package to do that and make sure that that works, that's a key element of this thing working, it might as well be something fun."*

## Verbatim — Emma Park on the onboarding plan

> *"So she's onboarding and then trying to play with partner center, and then I'm trying to add her to the test account we've been using for some games. So she can [do] like a test, build, create, and then also testing in Garrison."*

## Tester-side access confirmation

David Kushmerick: *"When you sign into Xbox.com, do you use your Gmail address?"*
Melanie: *"Yes."*
David: *"Then cool, then you should be able to access a playtest today."*

→ An existing playtest is already gated open for Melanie's Gmail-linked Xbox account. Use it to explore the tester UX (the entry path testers will hit when streamable playtests ship).

---

## Take-aways for this project

### 🚨 The end-June / early-July deadline reshapes the plan
The current design-doc timeline (Week 11-12 → mid-December) assumed a longer ramp. We need to compress aggressively. Critical-path items:

1. Unblock auth (Jack — GSAM Services + Sage route + LinkPad scaffold)
2. Wait on Jack+Timi's `playtest`-branch workaround for the rubber-stamp PR problem
3. Wait on Jack's TitleIngestion → Offering extension
4. **NEW**: Wait on Retterath's team for multi-version support OR scope-down to single-version-per-playtest for v1
5. Implement PlayTest-side: OfferingV2 builder, TitleIngestion submission, ServiceBus trigger
6. E2E with one test product (using ATG VideoTexture sample + David Kushmerick's GameConfig recipe)

### 🚨 New GSSV-side blocker: multi-version support for playtests
GSSV resolves "what build to stream" live from X product (latest-ingested-per-productId). Playtests need to target specific versions for specific audience groups. **Either**:
- (a) GSSV adds per-offering version-pinning (new cloud-side work, owned by Retterath's team), or
- (b) We scope v1 to "latest version only" — each productId can only have ONE active playtest build at a time — which may be acceptable for public preview.

Worth raising at design review: do we need (a) for v1, or can we ship with (b) and add (a) later?

### Test package strategy is solved
David Kushmerick's ATG-sample + GameConfig-editing recipe gives us a clean way to mint as many fake test products as we want from one sample build. Don't need a real game studio's binary. Recipe:

1. Clone an ATG sample (`Microsoft/Xbox-GDK-Samples` on GitHub) — VideoTexture is his recommendation
2. Build once
3. Duplicate the build folder N times — one per "product" you want to iterate on
4. In each copy, edit `MicrosoftGame.config` (a.k.a. GameConfig) to change:
   - `Name`
   - `Publisher`
   - `Identity / Name` (MSA App ID style)
   - `TileId` (a.k.a. titleId)
   - `StoreId` / `BigId` / `ContentId` — *"basically anything that's got an ID"*
5. Each variant can now be uploaded as a **distinct product** via Partner Center / playtest

Since David is on xplaytest, this is a workflow an internal xplaytest dev already uses for his own iteration.

### Confirms package → xCloud → distribution Services flow (xplaytest's framing)
Kushmerick's phrasing — *"upload a package… make its way over to the xCloud team… ingested into the xCloud distribution Services"* — matches our two-system model (XPackage Ingestion → GSSV Content Ingestion / TitleIngestion).

**"xCloud distribution Services"** is a term I hadn't heard before — likely Kushmerick's shorthand for the downstream side of GSSV that serves builds to streaming VMs (vs the ingestion side). Worth asking Retterath or Jack which subsystem this maps to.

### Streaming value prop now has a concrete second beat
Add to design-doc §1 "Why":
1. Instant access (no install wait)
2. **No retail-install conflict** — testers can keep their retail copy installed and still stream a different build of the same product

---

## Implications for the docs

| Where | Update |
|---|---|
| `architecture/glossary.md` | Add **Shadow Publishing Pipeline**, **ATG samples**, **GameConfig** (`MicrosoftGame.config`), **X product** (the upstream product-version source), **Group enumeration delay**; add David Retterath alongside Kushmerick |
| `architecture/service-map.md` contacts | Replace Kushmerick's team to **xplaytest**; add **David Retterath (xCloud)** + Emma Park + "next-week PM + Dev TBD" placeholder |
| `architecture/package-ingestion.md` | Add a "Test packages for E2E demo" sidebar with David Kushmerick's ATG-sample-+-GameConfig recipe |
| `design/design-doc.md` §1 | Add "no retail-install conflict" as a second value-prop beat |
| `design/design-doc.md` §13 (timeline) | Compress to the end-June / early-July deadline; reorder milestones |
| `design/design-doc.md` §14 | Add the multi-version GSSV-side gap as a top blocker |
| `design/open-questions-for-team.md` | Add §5i: multi-playtest-version support |
| `plan.md` (session) | Pin the end-June / early-July deadline as the primary constraint |

## Asks made / committed to

- **Emma**: add Melanie to the shared test account (in flight)
- **Kushmerick**: try the ATG VideoTexture sample as a starting point
- **Kushmerick**: sign into xbox.com with Gmail account → access the existing playtest today
- **Retterath's team**: send a PM + Dev next week for ongoing collaboration (committed)
- **Retterath's team**: assign a regular ongoing contact (committed)

