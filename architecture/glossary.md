# Glossary

Key terms used across the Instantly Shareable Playtest project.

| Term | Definition |
|------|-----------|
| **xplaytest** | The existing service/portal that allows game creators to upload private RETAIL-signed builds and manage testers. Lives in Xbox.Xbet.Service (`PlayTest` + `PlayTestFD`). |
| **Private Offering** | An xCloud concept: a streaming catalog of titles + metadata restricted to specific users. Managed by Partner Registry. Each playtest with cloud streaming gets its own private offering. |
| **GSSV** | Game Streaming Services (Xbox Cloud Gaming backend). Hosts game VMs, manages streaming sessions, serves `/offerings` to clients. |
| **DNA Group** | A user group system used for access control. Playtests use DNA groups to define who can test. Private offerings will use the same DNA groups for streaming access. |
| **Partner Registry** | GSSV's offering/title registry. CRUD at `/v1/offerings`. Per Jack (xCloud): *"basically our like database repo thing that we store all of our configuration stuff in."* Cosmos-backed; changes go through ADO PRs. |
| **Content Ingestion** | GSSV service that pulls product metadata from the Microsoft Store and stores it so consoles can install the package. Workflow-based; the relevant job type for this project is `TitleIngestion` (built by Jack Heuberger as his own intern project). |
| **Dev API** | GSSV reverse-proxy service for developers and other services. Two purposes: (1) lets devs hit GSSV endpoints for debugging; (2) defines which API routes are exposed and to whom. To call GSSV from a browser/script you go through Dev API. |
| **Sage** | Short for **Service API Gateway**. The production S2S traffic proxy. Services like Game Pass, XLab Business Automation, Xbox call into `/v1/offerings` via Sage. Routes have to be registered in Sage per (caller-app-id, target-service, method, path). |
| **GSAM Services group** | The AAD security group that grants dev access to Dev API. Melanie needs to be added (Jack to handle). |
| **LinkPad** | A .NET scripting app Jack uses for local prototyping of GSSV service calls. Wraps Dev API + the GSSV NuGet clients so you can write a `var client = new PartnerRegistryClient(...); await client.GetOfferingsAsync()` script instead of crafting raw HTTP. Jack is building a starter scaffold for Melanie. |
| **Big ID** | The product identifier used everywhere in GSSV. *"We call it a Package ID. Everybody else calls it a big ID, even though it's not very big."* Maps to `playtest.PartnerCenterProductId` on the PlayTest side. **This is all GSSV needs to ingest a game.** |
| **TitleCollection** | A way to organize games in the xCloud catalog. TitleIngestion currently writes a title into a TitleCollection. The other organizational unit is an Offering. **TitleIngestion currently doesn't support writing directly to an Offering — that's GSSV-side work Jack mentioned would be straightforward to add.** Examples of collection ids: `TESTXTESTING`, `ContentValidation`, `OLTEST`, `CERTTITLES`. |
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
| **SFI** | Secure Future Initiative — Microsoft-wide security mandate. The reason GSSV can no longer auto-rubber-stamp ADO PRs for offering changes in prod (you can't approve your own PR). Drives the rubber-stamp problem (P0 blocker). |
| **Rubber-stamp PR problem** | Every change to an offering (e.g. adding a title for a new playtest) creates an ADO PR that must be approved manually. SFI prevents the requester from self-approving. Jack + Timi are working on a workaround — likely a separate `playtest` branch in Partner Registry with relaxed protections that PlayTest's S2S identity can write to without a PR. Affects **every playtest-with-streaming**, not just initial setup. |
| **Content Access / Hydration (CAS)** | Per project spec Outcome #2: *"A new relationship / endpoint / functionality exists within catalog / Content Access / Hydration to support private offering / playtest content metadata."* One of the services explicitly listed in the Weeks-5-7 Core Service Work milestone. Likely surfaces playtest private-offering metadata to Bayside so testers can browse/discover available playtests. |
| **Agency** | The AI ramp-up tool called out by the official PDF for Weeks 1-2 ("Especially with Agency / Copilot CLI"). Refers to GitHub Copilot CLI in an agent-capable mode — used to read through unfamiliar repos and draft the implementation plan quickly. |
| **Connect** | The Microsoft impact-conversation cadence with the manager (Brian). Three project-defined checkpoints: **First Connect (6/2)**, **Midpoint Connect (6/30)**, **Final Connect (7/28)**. |
| **Feature Team Leader** | PDF-defined role: *"Accountable for the feature this project fits into."* For this project: David Kushmerick + Bec Lyons (jointly). |
| **Onboarding Buddy** | PDF-defined role: *"Initial point of contact to make sure you're settled in and everything is working."* For this project: Aditya Toney. |
| **UR Champ** | PDF-defined role: *"Gaming representative to the internship program; program-wide events; escalation for things the team can't help with."* For this project: Ellery Charlson. (UR = University Recruiting.) |
