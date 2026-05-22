# Project Spec (Canonical) — Instantly Shareable Playtest

> **Source**: `XD-Chen Melanie-Instantly Shareable Playtest.pdf` (Brian Bowman, Xbox Interns Summer 2026 Intern Project Plan)
> **This is THE authoritative source of truth for project goals, timeline, and success metrics.** Everything else in this repo should align with what's in this file.

## North Star Vision — 8-step flow

1. Creators easily create a playtest using the existing xplaytest portal and **optionally select cloud streaming** as a way to test.
2. Playtests with cloud streaming enabled communicate to **Xbox Game Streaming Services** to create a **private offering** unique to the playtest. (A private offering = a Cloud Gaming concept: a streaming catalog of titles + metadata restricted to specific users.)
3. Only users authorized for the playtest are configured for access to the Private Offering — **likely through DNA group integration in xCloud**.
4. Whenever a new game build is published to a playtest, **a workflow is created to ingest that build into Game Streaming Services** in order to be made available to the private offering. **As part of the workflow, the private offering is updated to have a single title available configured for the new game build.**
5. Playtesters have some way to access content in the private offering via a link (private offering support exists today in web cloud streaming but not Garrison, and not in the smoothest way for playtests) — or having the content "magically show up" once signed in.
6. Testers know it's a non-public version of the game; as much metadata as possible (title, genre, art) from store/playtest is visible.
7. Testers are offered the option to either install locally **or stream now**.
8. Streaming allows a tester to provide feedback on the experience.

## Outcomes (the formal success outcomes)

1. A new **S2S endpoint and relationship** is formed between Game Streaming Services and xplaytest / package publishing / partner center.
2. A new **relationship / endpoint / functionality** exists within **catalog / Content Access / Hydration** to support private offering / playtest content metadata.
3. A **unified experience for creators** enables testing of content regardless of endpoint or local install.
4. A flow exists for creators to **self-serve cloud streaming metadata** by being able to "publish" a playtest.

> **Note on outcome #2**: this is the Content Access / Hydration touch point. The PDF's Weeks-5-7 milestone explicitly mentions "CAS?" as one of the services to work in.

---

## P0 — critical to project success (verbatim from PDF)

| # | Goal | Details / Success Metric |
|---|---|---|
| P0.1 | **Creating a playtest creates a private offering** | Private offerings created within a few seconds of creating the playtest. Offering visible and manageable in the DevApi Reader Portal. |
| P0.2 | **Deleting a playtest deletes any associated private offering** | Private offerings deleted within a few seconds. Offering fully cleaned up; DevApi portal no longer shows it. |
| P0.3 | **Private offerings have a unique name/id for easy handling of multiple titles/sellers** | (Stretch P2: browse playtest offerings separately in DevApi Reader Portal; filter by studio/seller/title.) |
| P0.4 | **Updating private audience access list / groups updates the offering configuration** | Changing to a different DNA group changes the configuration. **Changing DNA group members automatically updates with no config needed.** |
| P0.5 | **Creation of a private offering can be limited to a specific title/seller for internal testing** | Xplaytest "flights" functionality to allowlist specific sellers and limit impact during development. |
| P0.6 | **Private offerings support DNA group access compatible with playtest** | All private offerings should support custom DNA groups checked during offering "login". **Currently "flight"/"DNA Group" auth doesn't show up in a user's /offerings call. This is a problem for the UX.** |
| P0.7 | **Xplaytest portal has a shareable link** | Xplaytest knows created offering details to craft a URL usable for streaming the game. |
| P0.8 | **Members not part of the playtest do not see any information when clicking the link** | Bayside does not leak details about the playtest to members not in it. |
| P0.9 | **When a new build is uploaded for a cloud-enabled playtest, that build is ingested into Game Streaming Services** | A few seconds/minutes after uploading, launching streaming uses the new version. (Stretch P2: xplaytest portal has user feedback about when build is "ready for testing".) |
| P0.10 | **When a new build is uploaded, that build is configured to be used by the private offering** | **At some point, the offering is configured to have a title in it that matches the product id of the uploaded build.** ← *Confirms the two-step ingestion + offering-link pattern Jack described.* |

## P1 — needed before P0 ships

| Goal | Details |
|---|---|
| **A service (CAS or GSSV) informs Bayside/Garrison about which playtests/private offerings should be used** | Today this is `/offerings` from GSSV. The UX client needs to get the list of playtest private offerings to log in and get the titles for. *(Couples with P0.6 — fixing the `/offerings` flight-auth visibility.)* |
| **Clicking a playtest link → Bayside login → stream** | Ensure player gets logged in *first* (before revealing any details). Once logged in, shows playtest details and starts streams successfully. |

## P3 — stretch goals

- **Bastion / Garrison support** for private offering playtests, with a "stream now" button in the playtest details page
- **Launch args** for the game (a nightly playtest can focus on a specific level, enable a dev mode, etc.)
- **Streaming region / touch controls / other streaming config** exposed in xplaytest portal
- **Feedback collection** from playtesters back to the studio via partner center / xplaytest

---

## Official 12-week timeline

| Weeks | Milestone | Category | Deliverable |
|---|---|---|---|
| **1-2** | Onboarding and Welcome | Ramp up | Understand project, know team, get productive (especially with **Agency** / Copilot CLI) |
| **Week 2 Checkpoint** | **First Connect (6/2)** | Impact Conversation | Connect |
| **3-4** | Design Doc and Prototyping / Manual Testing | Architecture and Design | Reviewed Design Doc of E2E work; detailed engineering estimates established |
| **5-7** | **Core Service Work** | Implementation | Services work complete across: **GSSV, xplaytest, CAS?, etc.** |
| **Week 6 Checkpoint** | **Midpoint Connect (6/30)** | Impact Conversation | Connect — **public preview target aligns here** (per Retterath) |
| **8-10** | **Front End Work** | Implementation | **Bayside player updates + xplaytest portal updates** (React, Tailwind) |
| **Week 10 Checkpoint** | **Final Connect (7/28)** | Impact Conversation | Connect |
| **11-12** | Documentation, Metrics, Polish | Engineering Excellence | Dashboards, monitors, TSGs |
| **Weeks 11-12 Presentation** | Final Project Presentation (date TBD by UR Champs) | Shareout | Presentation |

> **Critical reconciliation**: Retterath's "end June / early July public preview" lines up with the **Midpoint Connect** at end of Core Service Work (~Week 6-7), **not** the end of the project. The intern continues for ~5 more weeks after PP doing frontend + polish + presentation.

---

## Skills the project will exercise

| Skill | How |
|---|---|
| **AI Accelerated Ramp Up / Onboarding** | Understanding many repos and services quickly |
| **Micro Service Architecture and Systems Design (.NET)** | Working across services with resilience |
| **Experimentation, Quick Iteration and Prototyping** | Using AI + prototyping to prove out integrations |
| **Full Stack & Front End Development (React, Tailwind)** | Working on the **playtest portal and player endpoints** — Weeks 8-10 |
| **Cross Team Collaboration** | Working with GSSV and xPlayTest dev teams |
| **Cross Functional Collaboration** | Working with PM to understand requirements and coordinate |
| **Working through ambiguity** | Many possibilities; finding best tradeoffs critical to success |
| **Detail-oriented problem solving (edge cases)** | Many edge cases that need clarification |

---

## Key contacts (verbatim from PDF)

| Email Alias | Role | Description |
|---|---|---|
| **Brian Bowman** | Intern Manager | Primary contact; project definition + commitment setting; feedback and Connects |
| **Aditya Toney** | Onboarding Buddy | Initial point of contact to make sure you're settled in and everything is working |
| **Emma Park** | Intern Mentor | Day-to-day support; helps access resources; provides feedback to manager about progress |
| **Jessie Masih** | Administrative Assistant | Workplace resources — office setup, special needs, equipment, events |
| **David Kushmerick** + **Bec Lyons** | Feature Team Leader | **Accountable for the feature this project fits into** |
| **Anthony Keller**, **Timi Bolaji**, **Ashton Summer**, **Chuy Galvan** | Engineering Experts | ICs familiar with the major components to be modified |
| **Ellery Charlson** | UR Champ | Gaming representative to the internship program; program-wide events; escalation for things the team can't help with |

> **Plus** (discovered via calls, not in PDF): **Jack Heuberger** (xCloud — built TitleIngestion as his own intern project; not listed as an Engineering Expert in the PDF but he's the de-facto GSSV expert for our project). **David Retterath** (xCloud — confirmed multi-version gap, committed PM + Dev for ongoing support; also not in PDF but materially involved).

---

## Resources (from PDF)

| Description | URL |
|---|---|
| Intern Web Site | https://microsoft.sharepoint.com/teams/MicrosoftInterns |
| Xbox SharePoint | https://aka.ms/Gaming |
| Agentic AI for Gaming | https://aka.ms/AgentX |
| Team Alias | xbox-juno-dev |
| Team Wiki | https://dev.azure.com/microsoft/Xbox.Streaming/_wiki/wikis/Xbox.Streaming.wiki/327152/Xbox-Game-Streaming-Wiki |
| xCloud Dev API Offering Guidelines | (linked from Team Wiki) |
| Team ADO Sprint Board | https://dev.azure.com/microsoft/Xbox/_sprints/taskboard/Juno/Xbox/26-4W20%20(Ends%205-22)/2W18%20(Ends%205-8) |
| Code Repo: services.permissions | https://dev.azure.com/microsoft/Xbox.Streaming/_git/services.permissions/pullrequests?_a=mine |
| Code Repo: Xbox.JS | https://dev.azure.com/microsoft/Xbox/_git/Xbox.JS/pullrequests?_a=mine |
| Studios Tokens | https://aka.ms/StudiosTokens |

> Note: The PDF lists `services.permissions` and `Xbox.JS` as the code repos to look at. We've been working primarily with `services.partnerregistry` (per Jack's guidance). Worth verifying with Brian whether `services.permissions` is an additional touch point or whether the PDF is outdated.
