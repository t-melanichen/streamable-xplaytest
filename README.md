# Instantly Shareable Playtest — Documentation

Personal documentation repo for the **Instantly Shareable Playtest** intern project (Xbox / Juno team, Summer 2026).

## Project Summary

Combine xplaytest with Xbox Cloud Gaming so creators can share a **streaming link** for private builds — no download or install required. Testers stream via a private offering restricted to their DNA group.

## Repo Structure

```
architecture/        → Service maps, data flow diagrams, glossary
design/              → Formal design doc and API contracts
research/            → Per-service exploration notes
```

## Services Involved

| Service | Repo | Role |
|---------|------|------|
| PlayTest (core) | Xbox.Xbet.Service/src/PlayTest | Backend CRUD for playtests, ServiceBus workflows |
| PlayTestFD | Xbox.Xbet.Service/src/PlayTestFD | Partner Center-facing API proxy |
| ContentAccess | Xbox.Xbet.Service/src/ContentAccess | Content entitlements, knows about playtests |
| Partner Registry | services.partnerregistry | xCloud offerings & titles CRUD (`/v1/offerings`) |
| Play Xbox (Bayside) | Xbox.JS/apps/play-xbox | Player-facing web app with streaming |

## Key Links

- [Service Map](architecture/service-map.md)
- [Data Flow Diagram](architecture/data-flow.mmd)
- [Glossary](architecture/glossary.md)
- [Design Doc](design/design-doc.md)
- [API Contracts](design/api-contracts.md)
- [Research: PlayTest Service](research/playtest-service.md)
- [Research: Partner Registry](research/partner-registry.md)
- [Research: Bayside / Play Xbox](research/bayside-play-xbox.md)