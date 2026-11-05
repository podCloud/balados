# Roadmap - Balados Ecosystem

**Last updated:** 2026-11-05

The detailed, task-level plan lives in the [master plan](superpowers/plans/2026-11-05-plan-directeur.md) (mirrored as an editable Claude Doc). This file is the high-level view: what shipped, what is in flight, what comes next.

---

## Overview

- **Backend (`balados.sync`)**: production-ready CQRS/ES API, all sync endpoints complete, 890 tests green.
- **Frontend (`balados.app`)**: offline-first PWA with full sync integration (client, settings UI, merger, hook, background sync, likes, listening history), 483 tests green.
- **Integration**: Phases 1 and 2 of the original integration roadmap are complete (see [INTEGRATION_STATUS.md](INTEGRATION_STATUS.md)).

---

## Phase A: Core Integration - ✅ COMPLETE

Users can connect to a sync server and sync their data.

- OAuth popup + manual token flow (`SyncSettings.tsx`)
- Sync Settings UI (#23, PR #22)
- Conflict resolution merger (#24, PR #22)
- `useSync` hook (#25, PR #22)
- Health endpoint (sync #202, PR #204) and auth refresh (sync #203, PR #205)

## Phase B: Enhanced Sync - ✅ COMPLETE

- Server proxy integration (PR #34), sync status indicator (PR #48), background sync via Service Worker (PR #37), trending page (PR #49)
- Likes system: backend (sync #154, PR #255), frontend (PR #64), atomic Dexie transactions (#65, PR #66)
- Listening history page with filters, stats and streaks (#62, PR #67)

---

## Phase C: Security and debt - ✅ COMPLETE (2026-11-05)

Master plan Phase 1 and 2. Three independent security fixes shipped the same day, then repository hygiene.

| Item | Repo | Issue / PR | Follow-ups |
|------|------|------------|------------|
| DOMPurify 3.4.17 bump + `renderShowNotes` tests | app | #68 / PR #69 | #71 (`rel=noopener`) |
| Proxy 401/403/429 surfaced, persisted circuit breaker, transactional settings | app | #70 / PR #72 | #73 auth flow + refresh token, #74 no-public-proxy option, #75 Retry-After in aggregate pages, #76 flaky tests, #77 save/clear race |
| RSS proxy requires JWT, 120 req/min/user, server-wide CORS plug, private cache, unpadded ids | sync | #264 / PR #265 | #267 last-comma ids, #268 IP limiter, #269 gettext template, #270 CORS before router, #271 anonymous rate limit on public pages, #272 pre-commit formatter |
| Git cleanup: stale worktrees and branches removed, `erl_crash.dump` deleted | both | plan 2.1 | — |
| Root docs refresh, `development-workflow` skill modernised | root | plan 2.2 | — |

---

## Phase D: Dependency upgrades - 🔜 NEXT

Master plan Phase 3. No repo runs its tests in CI yet, so the safety net comes first.

1. **3.0 Safety net**: `test.yml` workflows in both repos, `.tool-versions` (sync) and `.nvmrc` (app). One PR per repo.
2. **3.1 Backend minors**: `commanded`, `eventstore`, `ecto_sql`, `finch`, `joken`, `swoosh`, …
3. **3.2 Phoenix 1.8 + LiveView 1.x**: one PR, Wallaby suite included.
4. **3.3 Hammer 7, gettext 1.0, httpoison 3**: one PR each.
5. **3.4 Frontend**: Vite/Vitest/Biome minors, then React Router, TanStack Query, Tailwind majors one at a time.

---

## Phase E: Third-party apps and product specs

Master plan Phase 4. Opens with the third-party app contract because **PodCloud will be the first application importing data into a Balados account**.

- **4.0 Third-party app contract**: fix the OAuth popup flow mismatch (app #73), persist a refresh token, document the server-side contract and client kit.
- **4.1 PodCloud one-way import**: subscriptions, likes, playlists, play positions pushed to Balados; never reads back.
- **4.2 Community tags** (sync #262) and **4.3 episode reactions** (sync #263): brainstorming with Pof, design doc, then a dedicated plan each.

---

## Phase F: Backlog (unscheduled)

- Error codes surfaced as user-friendly messages in the app
- Playlist and collection sync in the app (backend ready)
- Real-time sync via WebSocket (`/api/v1/live`, backend ready)
- Extended OPML import/export (positions, last listen, played, liked, playlists, subscription date)
- Discovery and social: collaborative playlists (sync #153), follow users (#155), hashtags/topics (#158), MinHash recommendations (#201)
- Token management: auto-refresh, revoked token handling (partly covered by 4.0)
- Multi-account support, privacy controls in the app

---

## Dependencies

### Frontend
- [x] React 19, Vite 7, TypeScript 5.9, Vitest 4, Dexie 4, TanStack Query v5, Workbox 7
- [ ] Node pinned (`.nvmrc`) and tests in CI (Phase D)

### Backend
- [x] Phoenix 1.7.21, LiveView 0.20, Commanded 1.4, EventStore 1.4, Hammer 6, Cachex 3
- [ ] Elixir/OTP pinned (`.tool-versions`) and tests in CI (Phase D)
- [ ] Phoenix 1.8 / LiveView 1.x (Phase D)

### Shared
- [x] URL-safe base64 without padding for feed and item ids
- [x] ISO 8601 timestamps
- [x] Last-write-wins conflict resolution
- [x] JWT RS256 with hierarchical scopes; RSS proxy requires `user.sync` or `user.subscriptions.read`

---

## Success Metrics

- Both repos green in CI on every PR (Phase D)
- Zero known security debt (Phase C: done)
- PodCloud can push a full account into Balados without data loss (Phase E)
- Works across mobile and desktop, offline changes sync when online
