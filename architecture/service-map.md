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

## Communication Patterns

### S2S HTTP (ServiceClient / FabricClient)
- **PlayTestFD → PlayTest**: FabricClient calls within Xbox.Xbet.Service
- **PlayTest → Partner Registry**: NEW — ServiceClient to call `/v1/offerings` CRUD
- **Bayside → GSSV**: Existing `/offerings` call to get available offerings

### ServiceBus Topics
- **PlayTest** already has topic processors:
  - `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` — triggers when a build is published
  - `XPackagePlaytestDeleteWorkflowJobStatusTopicProcessor` — triggers when a playtest is deleted
- **NEW**: On publish, also trigger GSSV build ingestion + update private offering title config

### Cosmos DB
- PlayTest core stores playtest state in Cosmos
- Partner Registry stores offering configs in Cosmos
- Both are independent data stores — no shared DB

### Azure DevOps PR Sync (Partner Registry specific)
- Partner Registry syncs offering changes via ADO PRs (used for auditing/approval of production offering changes)
- For playtest private offerings, may need a fast-path that bypasses PR approval

## Service Ownership

| Service | Team | What they own |
|---------|------|---------------|
| PlayTest / PlayTestFD | xplaytest (CORS) | Playtest lifecycle, audiences, packages |
| Partner Registry | GSSV / Game Streaming | Offering definitions, titles, streaming config |
| ContentAccess | XBET / Commerce | Entitlements, content access rules |
| Play Xbox (Bayside) | Xbox.JS / Juno | Player-facing streaming web experience |
