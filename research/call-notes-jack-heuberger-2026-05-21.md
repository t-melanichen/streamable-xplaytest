# Call notes — Jack Heuberger (xCloud / GSSV), 2026-05-21

Jack was an intern on the GSSV team and **built the `TitleIngestion` workflow himself as his own intern project**. He's the primary expert for the ingestion side. Brian (xplaytest manager) connected us.

## TL;DR

- **Three GSSV-side services touch this project**: Partner Registry, Content Ingestion, Dev API. There's a fourth proxy in front called **Sage** (Service API Gateway) for prod S2S traffic.
- **What we have to call** is the `TitleIngestion` workflow + `PUT /v1/offerings/{id}` on Partner Registry. *"We just need a Big ID"* (= `PartnerCenterProductId`).
- **The big P0 blocker** is the rubber-stamp PR problem in prod — Partner Registry writes create ADO PRs, SFI forbids self-approval. Jack + Timi are working on a `playtest`-branch workaround.
- **TitleIngestion currently writes to a TitleCollection, not an Offering** — Jack will extend it ("pretty easy"). PlayTest may be blocked on this.
- **Local prototyping**: Jack will scaffold a **LinkPad** project for Melanie that wraps the GSSV NuGet clients + proxies through Dev API.
- **Auth onboarding**: Jack to add Melanie to the GSAM Services group + register PlayTest's service client app id in Sage + add the `/v1/offerings` route. Brian to help track down the PlayTest app id.
- **No GSSV UI changes required for this project** — the GSSV Content Portal Blazor UI is only used by GSSV PMs.

## Services Jack walked through

### 1. Partner Registry
*"Our database repo thing that we store all of our configuration stuff in."*
- Holds offering / title configuration
- Backed by Cosmos
- Writes go through ADO PRs (see rubber-stamp problem below)
- Already-existing offerings include catalog/feature flags etc — we'll add a per-playtest one (or extend a shared one — open question §5f)

### 2. Content Ingestion
*"Handles pulling product metadata from the store to save it in effectively a database, so we can then tell consoles 'go install this package with this version'."*
- Workflow-based — multiple job types
- The relevant one for this project is `TitleIngestion` (Jack's intern project)
- *"Takes a product id, grabs the titles, grabs the metadata from the store, does that first product ingestion step and then adds that title to a title collection."*

### 3. Dev API
*"It serves two purposes — one is reverse proxy for devs and services who want to communicate with each other, and two is where access to all of the API routes is defined."*
- For dev/debug: lets a developer hit a GSSV endpoint without standing up the whole prod auth path
- For services: defines which routes are exposed to S2S callers
- Sits **in front** of Partner Registry / Content Ingestion etc — you don't call them directly

### 4. Sage (Service API Gateway)
*"Sage is short for Service API Gateway. So basically the things that are allowed to call into our services that are in production go through Sage. Currently I think it's Game Pass, XLab Business Automation, Xbox, etc. all do this. So when those services need to talk to ours, we need to add a route, like in Sage, that allows them to call into our services. So for instance, like to make a put request to V1 offerings, we'd have to add that route to Sage. That's super easy. Like that's not a problem. But it is something that needs to be done."*
- Per-(caller-app-id, target-service, method, route) allowlists
- Production-only S2S path
- We have to ask Jack to register `PUT /v1/offerings/{id}` (and any other routes we use) + register PlayTest's service client app id

## The TitleIngestion / offering relationship (P0)

> *"Like I'd say a title ingestion is sort of the same thing [as product ingestion] with extra steps. So a product ingestion just takes a product ID, grabs the metadata, and then doesn't do anything besides that with it. A title ingestion takes a product ID, grabs the titles, grabs the metadata from the store, does that first product ingestion step and then adds that title to a title collection. There's basically two ways for us to organize the games and stuff in our cloud catalog. You either add them to an offering or you add them to a title collection. So we would also need to expand this to work for offerings, but that's pretty easy to do."*

**Take-aways**:
1. `TitleIngestion` is the right job type for our use case
2. We pass the BigId (= `PartnerCenterProductId`); GSSV pulls everything else from the store
3. `TitleIngestion` currently writes to a `TitleCollection`, not an `Offering` — **GSSV-side extension required**, Jack to do this
4. PlayTest implementation may be blocked on this extension (open question §5g)

## The rubber-stamp PR problem in prod (P0 BLOCKER)

Confirmed by Jack — this is the **biggest** open issue.

> *"In test… we [have an MSI that] creates a pull request. And then like there's a pull request like I just go on rubber stamp it. And then like a minute later this is reflected in our services. The problem in our prod environment we have like SFI security requirements. So we forward your user off. So it will say like Melanie Chen made the pull request, so you can't approve it. … and we don't have a way to like automatically rubber stamp things as they go in today. So that is a problem."*

> *"For the sake of the demo, you'll just need to rubber stamp some PR that gets created to just move the play test part along… in the long run we'd need to figure out a solution for this. … Like in some, like one we're going to be discussed for this is like just for the sake of like playtest and your intern project, having like a playtest branch like that's next to main and you know that branch doesn't have all the same protections that main does like regarding needing a pull request to merge in. In theory, we could create special client methods for you to like that X playtest could call into and be like, 'hey, add this to the playtest branch in partner registry'."*

**Status**: Jack + Timi designing the workaround. **PlayTest is a net consumer** — we wait for the client methods. Demo fallback: manual approval.

## Local prototyping — LinkPad

Jack uses **LinkPad** (a .NET scripting app) to run small C# scripts locally that:
- Configure the GSSV NuGet `PartnerRegistryClient` (and other clients)
- Point at the Dev API URL with the right proxy header (`proxy to partner registry`)
- Let you write `await client.GetOfferingsAsync(...)` instead of crafting raw HTTP

> *"Like all of our services are still on .NET 8, that's a .NET 10 thing, so it's a little annoying… but I'll add that to my to-do list."* — Jack to put together a LinkPad scaffold for Melanie tomorrow.

This is more useful than my Bruno collection for prototyping real flows; Bruno is still useful for one-off probes. Document the LinkPad setup once it lands.

## What needs to happen for Melanie tomorrow

1. **Jack**: add Melanie to GSAM Services group (for Dev API access)
2. **Jack**: scaffold a LinkPad project that calls Partner Registry through Dev API
3. **Melanie**: start poking at offerings via LinkPad to confirm the contracts

## Asks Jack made of us

- Read your project doc and bring back specific questions — Jack hasn't seen the spec doc yet
- Bring the BigId story / which playtest sandbox / packaging questions to him + Timi (Timi owns the Microsoft Store integration)

## Implications for the docs

| Where | Update |
|---|---|
| `architecture/glossary.md` | Add Dev API, Sage, GSAM Services group, LinkPad, Big ID, TitleCollection, SFI, Rubber-stamp PR problem ✅ done |
| `architecture/service-map.md` | Add the Dev API → Sage proxy layer ✅ done |
| `architecture/package-ingestion.md` | Mark the rubber-stamp PR + TitleIngestion-to-Offering as P0 blockers ✅ done |
| `design/field-mapping.md` §6 | Update `assets: null` framing to "likely null per Jack" + caveat on TitleCollection-only ✅ done |
| `design/open-questions-for-team.md` | Update §5, 5a, 5e with answers/in-progress; add 5f, 5g, 5h ✅ done |
| `design/design-doc.md` §14 | Reorder top blockers; lead with rubber-stamp + TitleIngestion-extension ✅ done |
