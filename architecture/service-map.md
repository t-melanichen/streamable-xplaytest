# Service Map

## Overview

The Instantly Shareable Playtest project spans **5 services** across **3 repos**. This document maps how they connect.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Xbox.Xbet.Service (monorepo)                       │
│                                                                           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐       │
│  │  PlayTestFD  │───▶│   PlayTest   │───▶│   ContentAccess      │       │
│  │  (front-door)│    │   (core)     │    │   (entitlements)     │       │
│  └──────────────┘    └──────────────┘    └──────────────────────┘       │
│         ▲                   │                                            │
│         │                   │ ServiceBus                                  │
│         │                   ▼                                             │
│         │            ┌──────────────┐                                    │
│         │            │  Workflows   │                                    │
│         │            │  (publish/   │                                    │
│         │            │   delete)    │                                    │
│         │            └──────────────┘                                    │
└─────────┼──────────────────┼────────────────────────────────────────────┘
          │                  │
          │                  │ NEW: S2S HTTP calls
          │                  ▼
┌─────────┴──────────────────────────────────────────────────────────────┐
│              services.partnerregistry                                    │
│                                                                           │
│  ┌──────────────────────────────────────────┐                           │
│  │  Partner Registry Service                 │                           │
│  │  • OfferingsController (/v1/offerings)    │                           │
│  │  • Titles management                      │                           │
│  │  • Cosmos DB + Azure DevOps PR sync       │                           │
│  └──────────────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
          │
          │ Offering config propagates to GSSV
          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Xbox.JS (monorepo)                                           │
│                                                                           │
│  ┌──────────────────────────────────────────┐                           │
│  │  Play Xbox (Bayside)                      │                           │
│  │  • Offering enrollment UI                 │                           │
│  │  • Game streaming routes                  │                           │
│  │  • React + Tailwind + Vite + Express      │                           │
│  └──────────────────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## GSSV access layer (clarified after Jack Heuberger call, 2026-05-21)

You don't call Partner Registry / Content Ingestion directly. All traffic goes through one of two proxies:

```
        ┌──────────────────────────────────────────────────────────────┐
        │                                                              │
        │     Dev / debug client (Melanie, browser, LinkPad script)    │
        │                              │                               │
        │                              ▼                               │
        │                  ┌──────────────────────┐                    │
        │                  │      Dev API         │                    │
        │                  │  (reverse proxy +    │                    │
        │                  │   route registry)    │                    │
        │                  └──────────────────────┘                    │
        │                                                              │
        │     PlayTest service (prod S2S)                              │
        │                              │                               │
        │                              ▼                               │
        │                  ┌──────────────────────┐                    │
        │                  │   Sage (Service      │                    │
        │                  │   API Gateway)       │                    │
        │                  │   • per-(caller-app, │                    │
        │                  │     method, route)   │                    │
        │                  │     entries          │                    │
        │                  └──────────────────────┘                    │
        │                              │                               │
        │              ┌───────────────┴────────────────┐              │
        │              ▼                                ▼              │
        │   ┌──────────────────┐          ┌──────────────────────┐    │
        │   │ Partner Registry │          │  Content Ingestion   │    │
        │   │  /v1/offerings   │          │  (TitleIngestion     │    │
        │   │  /v1/titles      │          │   workflow)          │    │
        │   └──────────────────┘          └──────────────────────┘    │
        │                                                              │
        └──────────────────────────────────────────────────────────────┘
```

**Implications for PlayTest**:
- For dev/testing: Melanie hits Dev API (after being added to **GSAM Services group**); can use **LinkPad** to write C# scripts that pretend to be Dev API and call the GSSV NuGet clients
- For prod: PlayTest's service identity needs to be registered in **Sage**, with explicit route entries for `PUT /v1/offerings/{id}`, `DELETE /v1/offerings/{id}`, `POST {ingestion workflow path}`, etc.
- Both onboarding steps are tracked by Jack (xCloud) and Brian (xplaytest manager)
- See `architecture/package-ingestion.md` and `design/open-questions-for-team.md` §5e

## Communication Patterns

### S2S HTTP (ServiceClient / FabricClient)
- **PlayTestFD → PlayTest**: FabricClient calls within Xbox.Xbet.Service
- **PlayTest → Partner Registry (via Sage)**: NEW — ServiceClient to call `/v1/offerings` CRUD
- **PlayTest → Content Ingestion (via Sage)**: NEW — submit `TitleIngestion` workflow jobs at build-publish time
- **Bayside → GSSV**: Existing `/offerings` call to get available offerings

### ServiceBus Topics
- **PlayTest** already has topic processors:
  - `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` — triggers when a build is published
  - `XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor` — triggers when a playtest is deleted
- **NEW**: On publish, also kick off GSSV `TitleIngestion` workflow (decoupled from offering create — see `architecture/package-ingestion.md`)

### Cosmos DB
- PlayTest core stores playtest state in Cosmos
- Partner Registry stores offering configs in Cosmos
- Both are independent data stores — no shared DB

### Azure DevOps PR Sync (Partner Registry specific)
- Partner Registry syncs offering changes via ADO PRs (used for auditing/approval of production offering changes)
- **PROBLEM** (per Jack, 2026-05-21): SFI forbids self-approval, and PlayTest's S2S identity gets forwarded as the requester user, so it cannot auto-merge. This blocks every playtest-with-streaming.
- Jack + Timi are working on a fix — likely a separate `playtest` branch with relaxed protections. See `design/open-questions-for-team.md` §5.

## Service Ownership

| Service | Team | What they own |
|---------|------|---------------|
| PlayTest / PlayTestFD | xplaytest (CORS) | Playtest lifecycle, audiences, packages |
| Partner Registry | GSSV / Game Streaming | Offering definitions, titles, streaming config |
| Content Ingestion | GSSV / Game Streaming | `TitleIngestion` / `ProductIngestion` workflows |
| Dev API | GSSV / Game Streaming | Dev reverse proxy + API route registry |
| Sage (Service API Gateway) | GSSV / Game Streaming | Prod S2S proxy + per-caller allowlists |
| ContentAccess | XBET / Commerce | Entitlements, content access rules |
| Play Xbox (Bayside) | Xbox.JS / Juno | Player-facing streaming web experience |

## Key contacts

> Canonical list (with PDF-verbatim role labels) is in [`research/project-spec.md`](../research/project-spec.md) §Key contacts.

| Person | Team / Role | Role for this project |
|---|---|---|
| **Brian Bowman** | xplaytest — **Intern Manager** | Scope decisions, project blockers escalation, PlayTest service identity app ID, Connects |
| **Aditya Toney** | **Onboarding Buddy** | Initial setup, equipment, environment problems |
| **Emma Park** | **Intern Mentor** | Day-to-day support, test account access, Partner Center onboarding, Garrison setup |
| **Jessie Masih** | **Administrative Assistant** | Workplace resources, office setup, events |
| **David Kushmerick** + **Bec Lyons** | xplaytest — **Feature Team Leader** (jointly) | Feature accountability, design review, ATG sample workflow (Kushmerick), shadow publishing pipeline, group enumeration delays, xplaytest portal UI scoping |
| **Anthony Keller** | xCloud (GSSV) — **Engineering Expert** | S2S onboarding / app id allowlists; original onboarding contact |
| **Timi Bolaji** | xCloud (GSSV) — **Engineering Expert** | Microsoft Store integration; co-owns rubber-stamp PR workaround |
| **Ashton Summer**, **Chuy Galvan** | **Engineering Experts** | Additional ICs to be brought in as their areas surface |
| **Ellery Charlson** | **UR Champ** | Program-wide events, Final Presentation scheduling, escalation for things the team can't help with |
| **Jack Heuberger** | xCloud (GSSV) — discovered via call | Built `TitleIngestion` as his own intern project — primary expert. Owns onboarding into GSAM Services / LinkPad / Sage routes. Working with Timi on rubber-stamp workaround. |
| **David Retterath** | xCloud — discovered via call | Offerings ↔ DNA groups ↔ titles model; multi-version gap; committed ongoing PM + Dev support |
| **Retterath's PM + Dev** (TBD) | xCloud | Joining Week 2 for ongoing collaboration — design review, GSSV-side prioritization |

## Integration touch points (per official spec)

Per Outcome #2 in [`research/project-spec.md`](../research/project-spec.md): a new relationship / endpoint / functionality exists within **catalog / Content Access / Hydration** to support private offering / playtest content metadata. Weeks-5-7 milestone explicitly lists "CAS?" as a service to work in. **Content Access Service / Hydration** is a real integration target — likely surfacing playtest private-offering metadata to Bayside.
