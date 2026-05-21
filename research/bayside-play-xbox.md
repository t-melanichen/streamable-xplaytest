# Research: Bayside / Play Xbox (Xbox.JS)

## Location
- Repo: `Xbox.JS`
- App: `apps/play-xbox/` (this IS Bayside)
- Also: `apps/xboxcom-edgewater/` (Xbox.com — less relevant)

## What It Is
**Play Xbox (Bayside)** is the player-facing web app for Xbox Cloud Gaming (`xbox.com/play`). It handles:
- Game streaming sessions
- Offering enrollment
- Game library / catalog browsing
- Account auth

## Tech Stack
- **React 18** + React Router v7
- **TanStack Query** (data fetching/caching)
- **Vite** (build tooling)
- **Express** (SSR server)
- **Tailwind CSS** + XDS (Xbox Design System)
- Monorepo managed by **Nx** + **Yarn**

## Relevant Packages (for this project)

| Package | Purpose |
|---------|---------|
| `@play-xbox/route-game-stream` | Game streaming route/page |
| `@play-xbox/route-product-detail` | Product detail page |
| `@play-xbox/react-components-offering-enrollment` | Offering enrollment UI |
| `@play-xbox/game-stream-has-streaming-access` | Check if user has streaming access |
| `@play-xbox/route-offers-and-credits` | Offers display |
| `@play-xbox/route-remote-play` | Remote play route |

## How It Connects to Backend
- Express server as BFF (Backend for Frontend)
- React Router loaders fetch data during SSR into TanStack Query
- Calls GSSV's `/offerings` endpoint to get available offerings for the user
- Existing developer mode supports private offering URLs

## Existing Private Offering Support
From `apps/play-xbox/docs/How Tos/Enable Developer Mode.md`:
- Dev mode already allows testing with private offerings
- There's existing infrastructure for passing offering IDs to the streaming flow
- Settings/Developer tab has offering-related configuration

## What Needs to Change (P0)

### 1. Handle Playtest Streaming Link
- When a tester clicks a shareable playtest link (e.g., `xbox.com/play/playtest/{offeringId}`):
  1. Force authentication first (no details revealed pre-login)
  2. Check if user is in the offering's DNA group
  3. If authorized: show playtest details (title, art, metadata) + "Stream Now" button
  4. If not authorized: show generic access denied (no metadata leak)

### 2. DNA Group Check in /offerings
- Currently DNA group auth doesn't surface in `/offerings` responses (known issue)
- Need GSSV or ContentAccess to include playtest private offerings in a user's available offerings
- This might be a GSSV-side fix rather than a Bayside change

### 3. Playtest Metadata Display
- When showing a playtest offering, display as much metadata as possible:
  - Game title, genre, art (from store/playtest data)
  - Indicate this is a "non-public version" / playtest build
  - Show "Install Locally" OR "Stream Now" options (P0 just needs streaming)

### 4. Security: No Information Leakage
- Bayside must NOT reveal playtest details to unauthorized users
- Login must happen BEFORE any offering/game details are shown
- The link itself should not contain game title or other metadata

## Open Questions
- What's the exact URL format for the shareable link? Options:
  - `xbox.com/play/launch/{offeringId}` (existing pattern?)
  - `xbox.com/play/playtest/{playtestId}` (new route)
  - A short URL / redirect service
- Does Bayside need a new route or can it use the existing offering enrollment flow?
- How does the existing `game-stream-has-streaming-access` package work with DNA groups?
