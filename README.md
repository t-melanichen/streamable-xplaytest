# Instantly Shareable Playtest — Documentation

Personal documentation repo for the **Instantly Shareable Playtest** intern project (Xbox / Juno team, Summer 2026).

## Project Summary

Let xplaytest creators flip a "cloud streaming" toggle on a playtest, then share a single URL that lets authorized testers stream the build in their browser via xbox.com/play — no install, no console.

## The P0 deliverable

A field-by-field mapping from PlayTest's playtest model to GSSV's `OfferingV2` / `Title` / `Flight` model, plus the build-ingestion call to GSSV ContentIngestion.

➡️ **[`design/field-mapping.md`](design/field-mapping.md)** is the central doc.

## Repo Structure

```
architecture/                       Service maps, the flight model, package ingestion model
  ├─ service-map.md
  ├─ data-flow.mmd                  Mermaid sequence diagram
  ├─ glossary.md
  ├─ flights-and-dna-groups.md      ← How DNA groups become flights (already in code!)
  └─ package-ingestion.md           ← Three systems: XPackage / ContentIngestion / Offering Registry

design/
  ├─ design-doc.md                  Formal design doc (Week 3-4 deliverable)
  ├─ execution-plan.md              ← How this project actually plays out (week-by-week)
  ├─ field-mapping.md               ← THE P0 DELIVERABLE
  └─ open-questions-for-team.md     Questions for Brian / Anthony / Timi / GSSV team

research/
  ├─ playtest-service.md            Xbox.Xbet.Service/src/PlayTest notes
  ├─ partner-registry.md            services.partnerregistry (= GSSV offering registry) notes
  ├─ bayside-play-xbox.md           Xbox.JS/apps/play-xbox notes
  └─ call-notes-jack-heuberger-2026-05-21.md   ← Jack (xCloud, built TitleIngestion) onboarding call

bruno/
  └─ offering-registry/             Bruno collection: exercise GSSV /v1/offerings live
```

## Services Involved (3 local repos)

| Service | Local repo | Role |
|---|---|---|
| **PlayTest core** | `C:\Users\t-melanichen\source\repos\Xbox.Xbet.Service\src\PlayTest` | Backend CRUD; SQL+EF persistence; ServiceBus processors. **Owns the new integration.** |
| **PlayTestFD** | `Xbox.Xbet.Service\src\PlayTestFD` | Partner Center-facing API proxy |
| **GMS Service** | (external) | Group Management Service. PlayTest already calls it. `GmsGroupId → UserDnAGroupIds[]`. |
| **GSSV Offering Registry** | `C:\Users\t-melanichen\OneDrive - Microsoft\Desktop\services.partnerregistry` | The `OfferingsController` (`/v1/offerings`). Namespace `Microsoft.GameStreaming.Partners.Contracts`. **Owned by GSSV team.** |
| **GSSV ContentIngestion** | (external NuGet `Microsoft.GameStreaming.ContentIngestion.Client 1.0.2604.2902`) | Registers a build as streamable. **Owned by GSSV team.** |
| **XPackage / XPackageWorkflow** | `Xbox.Xbet.Service\src\XPackage` | Existing build ingestion pipeline (boxes/segments). **Already runs today.** |
| **Bayside (Play Xbox)** | `C:\Users\t-melanichen\projects\Xbox.JS\apps\play-xbox` | Player-facing streaming web app |

> **Key fact**: xPlayTest and GSSV do **not** talk today. This project is the net-new integration. `services.partnerregistry` IS GSSV — it's where GSSV's offering management API lives.

## Key Links

- **[design/field-mapping.md](design/field-mapping.md)** — the P0 mapping deliverable
- **[design/execution-plan.md](design/execution-plan.md)** — week-by-week project plan synthesized from all call notes
- [design/design-doc.md](design/design-doc.md) — formal design doc
- [design/open-questions-for-team.md](design/open-questions-for-team.md) — clarifying questions
- [architecture/flights-and-dna-groups.md](architecture/flights-and-dna-groups.md) — DNA → Flight (already in code)
- [architecture/package-ingestion.md](architecture/package-ingestion.md) — three systems
- [architecture/service-map.md](architecture/service-map.md)
- [architecture/data-flow.mmd](architecture/data-flow.mmd)
- [architecture/glossary.md](architecture/glossary.md)
- [research/playtest-service.md](research/playtest-service.md)
- [research/partner-registry.md](research/partner-registry.md)
- [research/bayside-play-xbox.md](research/bayside-play-xbox.md)
- [research/call-notes-jack-heuberger-2026-05-21.md](research/call-notes-jack-heuberger-2026-05-21.md) — onboarding call with Jack (xCloud, built TitleIngestion)
- [bruno/offering-registry/README.md](bruno/offering-registry/README.md) — Bruno collection for testing offering create/get/update/delete against GSSV dev
