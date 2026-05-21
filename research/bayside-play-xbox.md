# Research: Play Xbox (Bayside) in Xbox.JS

## Location
- Repo root: `C:\Users\t-melanichen\projects\Xbox.JS`
- App: `apps/play-xbox/`
- Game streaming package: `packages/@play-xbox/route/game-stream/`
- Offering enrollment: `packages/@play-xbox/feature/offering-enrollment/`

## What It Does
Play Xbox (codename **Bayside**) is the player-facing web app at `xbox.com/play`. It handles cloud game streaming, catalog browsing, offering enrollment, and (for Game Pass tiers) library/quests/streaming-room features. Built on **React Router v7** (`react-router: ^7.13.1`).

## Verified Routes (file `apps/play-xbox/src/app/routes.ts`)

| Route | File | Notes |
|---|---|---|
| `/stream/:productId/:productName?` | `CloudConsoleStreamRoute.tsx` (lines 203-219) | Primary streaming entry point. Used today for Game Pass streaming. **This is the most likely target for shareable playtest links.** |
| `/_internal/dev-tools/offering/:offeringId` | `OfferingBrowserRoute.ts` (lines 363-379) | Internal dev tool for browsing offerings — useful for testing |
| `/play` | home | catalog/browse |
| (Game Pass library / quests routes) | various | Game Pass tier-gated UI |

## Loader Behavior on `/stream/:productId`

File: `packages/@play-xbox/route/game-stream/src/loader.server.ts:28-83`

```ts
const { activeOfferingInfo } = await gameStream.authentication.queries.activeOfferingInfo({ productId });
const { detailed } = await catalog.queries.productInfo.detailed({ productId });
// ...
if (activeOfferingInfo.isPrivate) {
  return { type: 'redirectingToAuth', ... };   // avoids hydration leak
}
```

Key behaviors:
- The loader already handles `isPrivate` offerings — it returns an early redirect to avoid leaking the title's metadata before auth.
- The catalog query (`productInfo.detailed`) fetches product metadata using the Universal Store ProductId — works for unreleased playtest products **iff** the product is in the catalog.

## Existing Auth Check

File: `packages/@play-xbox/feature/cloud-console-stream/src/access/useHasStreamingAccess.ts:11-33`

```ts
const hasUltimateAccess = hasUltimateAccess(user);
const hasStandardAccess = hasStandardAccess(user);
const hasCoreAccess = hasCoreAccess(user);
return hasUltimateAccess || hasStandardAccess || hasCoreAccess;
```

**Today, the streaming gate is Game Pass tier only.** There is no DNA-group or flight check in this hook. For playtest streaming, we'll need either:
- A new branch in this hook that also returns `true` when the user is in the playtest's flight, OR
- A separate "private offering enrollment" path that bypasses this hook (since the user already enrolled via the link).

## Private Offering Dev Mode

File: `packages/@play-xbox/route/game-stream/src/loader.server.ts` and adjacent

Dev/test pattern for overriding the offering:
```
?configs=streaming.auth.offering:<offeringId>
?configs=auth.xbox.sandboxId:<sandboxId>
```

This is the existing knob for forcing a specific offering on a stream URL. **It's a strong candidate for the production shareable link format**, e.g.:
```
https://www.xbox.com/play/stream/{productId}?configs=streaming.auth.offering:xpt-{playtestId}
```

But it's a dev-mode pattern today, so the team may want a cleaner production URL (e.g., a dedicated `/playtest/{token}` route) — worth confirming with PM.

## Offering Enrollment

File: `packages/@play-xbox/feature/offering-enrollment/src/components/OfferingEnrollment.tsx:27-167`

- Renders a QR code + an external enrollment URL constructed from the user's XUID + an offering enrollment URL.
- Has its own gated UI for offering-specific flows.

This is reusable for the "user clicks shareable link → enroll in offering → stream" flow, **if** offering enrollment is the right model (vs. the flight membership being checked at session-creation time).

## GSSV URL Pattern (used by streaming client)

Format: `https://{prefix}.gssv-play-{httpEnv}.xboxlive.com`

Matches the `DedicatedFQDN` pattern on `OfferingV2` in Partner Registry: when `DedicatedFQDN = true`, the offering gets a hostname like `{offeringid}.gssv-play-prod.xboxlive.com`.

## What's NOT in Bayside today

- No DNA-group / flight membership check in the client
- No "playtest" route or component (greenfield)
- No "this is a private playtest, here's the metadata" preview component
- No `xplaytest` route — the xplaytest portal is **not in Xbox.JS** at all (likely a Partner Center frontend in a separate repo)

## Recommended Pattern for the Shareable Link

1. Tester receives link: `https://www.xbox.com/play/stream/{productId}?configs=streaming.auth.offering:xpt-{playtestId}`
2. Bayside loader fetches `activeOfferingInfo` — recognizes `isPrivate`, redirects to auth if not signed in
3. After auth, loader resolves offering, GSSV checks the user is in the offering's `AllowedFlights[].FlightId`
4. If allowed, render the streaming UI with the playtest's metadata (Name, GameArtUrl, Description from PlayTest contract)
5. If not allowed, render a "you don't have access to this playtest" page (must NOT leak title metadata for unreleased games)

**Open UX question**: should there be an explicit "claim your playtest invite" intermediate page (consume the link once), or should the link be replayable for the duration of the playtest?
