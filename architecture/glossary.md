# Glossary

Key terms used across the Instantly Shareable Playtest project.

| Term | Definition |
|------|-----------|
| **xplaytest** | The existing service/portal that allows game creators to upload private RETAIL-signed builds and manage testers. Lives in Xbox.Xbet.Service (`PlayTest` + `PlayTestFD`). |
| **Private Offering** | An xCloud concept: a streaming catalog of titles + metadata restricted to specific users. Managed by Partner Registry. Each playtest with cloud streaming gets its own private offering. |
| **GSSV** | Game Streaming Services (Xbox Cloud Gaming backend). Hosts game VMs, manages streaming sessions, serves `/offerings` to clients. |
| **DNA Group** | A user group system used for access control. Playtests use DNA groups to define who can test. Private offerings will use the same DNA groups for streaming access. |
| **Partner Registry** | The service that manages xCloud offerings and titles. CRUD at `/v1/offerings`. Stores config in Cosmos DB. |
| **Bayside** | Codename for the **Play Xbox** web app (`xbox.com/play`). The player-facing streaming endpoint. React + Tailwind + Vite. |
| **Garrison** | The Xbox app/client used for local game installs and testing. Currently the primary xplaytest endpoint for testers. |
| **Bastion** | Related to Garrison — another client endpoint. Stretch goal (P3) for private offering support. |
| **PlayTestFD** | "Front Door" — the Partner Center-facing API proxy for playtest operations. Thin layer that forwards to PlayTest core. |
| **PlayTest (core)** | Backend service that owns playtest lifecycle: create, update, delete, publish. Has ServiceBus processors for build workflows. |
| **ContentAccess** | Service in Xbox.Xbet.Service that handles content entitlements. Already knows about playtests — relevant for hydration of playtest content metadata. |
| **Flights** | Feature flags / allowlists used to limit rollout. For this project: limiting private offering creation to specific sellers during development. |
| **ServiceClient** | The typed HTTP client pattern in Xbox.Xbet.Service for S2S calls. Extends `Shared.Common.ServiceClient`. |
| **FabricClient** | Internal service-to-service client for calls within the Xbox.Xbet.Service fabric cluster. Uses `RunRequestAsync`. |
| **OfferingV2** | The Partner Registry data model for an offering. Includes auth config, allocation pools, regions, SUGs, dedicated FQDN, session limits, expiration, etc. |
| **Title** | Within Partner Registry: a game title configuration within an offering. Includes product IDs, platforms, entitlements, flights, availability, server types. |
| **XPackage** | The package format for Xbox game builds. PlayTest has ServiceBus processors that fire when packages are published/deleted. |
| **DedicatedFQDN** | Pattern: `[offeringid].gssv-play-prod.xboxlive.com`. How offerings map to GSSV streaming endpoints. |
| **DevApi Reader Portal** | Internal portal for viewing/managing Partner Registry offerings. Success metric: private offerings visible here. |
