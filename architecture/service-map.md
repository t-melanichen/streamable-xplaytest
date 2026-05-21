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

## Key contacts (so far)

| Person | Team | Role for this project |
|---|---|---|
| **Jack Heuberger** | xCloud (GSSV) | Built `TitleIngestion` as his own intern project — primary expert. Owns onboarding into GSAM Services / LinkPad / Sage routes. Working with Timi on the rubber-stamp PR workaround. |
| **Timi** | xCloud (GSSV) | Microsoft Store integration expert; co-owns rubber-stamp PR workaround. |
| **Anthony Keller** | xCloud (GSSV) | S2S onboarding / app id allowlists (per earlier onboarding notes). |
| **Brian (Bowman)** | xplaytest (manager) | Will help track down PlayTest service client app id for Sage registration. |
| **David** | xplaytest / org | Sponsored the project; messaged Jack months ago about the intern project idea. |
