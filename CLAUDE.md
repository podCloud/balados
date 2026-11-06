# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Balados** is a federated podcast ecosystem consisting of two independent projects:

| Project | Type | Stack | Purpose |
|---------|------|-------|---------|
| **balados.app** | Frontend PWA | React 19 + Vite + TypeScript | Offline-first podcast player |
| **balados.sync** | Backend Server | Elixir + Phoenix + CQRS/ES | Sync server with API + Web UI |

The projects follow a Bluesky/Mastodon-like model: the app works standalone but gains sync capabilities when connected to a server.

---

## Quick Reference

### Development Commands

```bash
# Frontend (balados.app)
cd balados.app
npm run dev          # Start dev server (http://localhost:5173)
npm run build        # Type-check + build
npx vitest run       # Run Vitest tests once (npm test = watch mode)

# Backend (balados.sync)
cd balados.sync
mix deps.get         # Install dependencies
mix db.create        # Create databases
mix db.init          # Initialize event store + migrate
mix phx.server       # Start server (http://localhost:4000)
mix test --exclude e2e  # Run tests (browser e2e tests need ChromeDriver: --include e2e)
```

### Running Both Together

```bash
# Terminal 1 - Backend
cd balados.sync && mix phx.server

# Terminal 2 - Frontend
cd balados.app && npm run dev
```

Frontend at `http://localhost:5173`, Backend at `http://localhost:4000`.

---

## Architecture Summary

### Data Flow (Connected Mode)

```
┌─────────────────────────────────────────────────────────────────┐
│  balados.app (Frontend PWA)                                     │
│                                                                 │
│  ┌──────────────┐     ┌───────────────┐     ┌──────────────┐   │
│  │ React UI     │ ──► │ TanStack Query│ ──► │ IndexedDB    │   │
│  │ (Components) │     │ (Data Fetch)  │     │ (Dexie.js)   │   │
│  └──────────────┘     └───────────────┘     └──────────────┘   │
│         │                    │                     │            │
│         │                    ▼                     │            │
│         │           ┌───────────────┐              │            │
│         │           │  SyncQueue    │◄─────────────┘            │
│         │           │  (Offline)    │                           │
│         │           └───────┬───────┘                           │
└─────────┼───────────────────┼───────────────────────────────────┘
          │                   │
          │                   ▼
          │     ┌─────────────────────────┐
          │     │   REST API / WebSocket  │
          │     └───────────┬─────────────┘
          │                 │
┌─────────┼─────────────────┼─────────────────────────────────────┐
│         ▼                 ▼                                     │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────────────┐   │
│  │ Phoenix      │  │ Commanded   │  │ EventStore           │   │
│  │ Controllers  │─►│ (CQRS)      │─►│ (Source of Truth)    │   │
│  └──────────────┘  └──────┬──────┘  └──────────────────────┘   │
│                           │                    │                │
│                           ▼                    │                │
│                    ┌─────────────┐             │                │
│                    │ Projectors  │◄────────────┘                │
│                    └──────┬──────┘                              │
│                           ▼                                     │
│                    ┌─────────────┐                              │
│                    │ PostgreSQL  │                              │
│                    │ Projections │                              │
│                    └─────────────┘                              │
│  balados.sync (Backend CQRS/ES)                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Sync Modes

| Mode | Description | Features |
|------|-------------|----------|
| **Standalone** | No server connection | Local storage, public CORS proxies |
| **Connected** | Linked to balados.sync | Cross-device sync, server proxy, trending |

---

## API Contract

### Base URL

```
https://{server}/api/v1/
```

### Data Encoding Convention

All encoding uses **URL-safe base64** (RFC 4648 §5: `-` instead of `+`, `_` instead of `/`, no `=` padding). Shared functions in `balados.app/src/utils/rssEncoding.ts`.

```typescript
import { encodeRssFeed, encodeRssItem } from "./utils/rssEncoding";

// Feed URL → base64url
const rssFeed = encodeRssFeed(feedUrl)
// Example: "https://example.com/feed.xml" → "aHR0cHM6Ly9..."

// Episode ID → base64url(guid,enclosureUrl)
const rssItem = encodeRssItem(guid, enclosureUrl)
// IMPORTANT: Decode with lastIndexOf(",") because guid may contain commas
```

### Authentication

JWT Bearer tokens (RS256 asymmetric). OAuth-style flow:
1. App generates JWT with public key
2. User approves on server's `/authorize` page
3. Server stores public key for verification
4. App signs API requests with private key

### Key Endpoints

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/health` | Health check | No |
| POST | `/sync` | Full/incremental sync | JWT |
| GET | `/subscriptions` | List subscriptions | JWT |
| POST | `/subscriptions` | Add subscription | JWT |
| DELETE | `/subscriptions/{feed}` | Remove subscription | JWT |
| POST | `/play` | Update play position | JWT |
| GET | `/likes` | List user's likes (paginated) | JWT |
| POST | `/likes` | Like a podcast or episode | JWT |
| DELETE | `/likes/{feed}` | Unlike a podcast | JWT |
| GET | `/rss/proxy/{feed}` | CORS proxy (scope `user.sync` or `user.subscriptions.read`, 120 req/min/user, 401/403/429) | JWT |
| POST | `/auth/refresh` | Token validation/refresh | JWT |
| GET | `/public/trending/podcasts` | Trending | No |

### Sync Data Types

```typescript
// Subscription
interface SubscriptionSync {
  rss_source_feed: string;     // base64url(feedUrl)
  subscribed_at: string;       // ISO date
  unsubscribed_at?: string;    // ISO date (if unsubscribed)
}

// Play Status
interface PlayStatusSync {
  rss_source_feed: string;     // base64url(feedUrl)
  rss_source_item: string;     // base64url(guid,enclosureUrl)
  position: number;            // seconds
  played: boolean;             // completed
  updated_at: string;          // ISO date
}

// Like
interface LikeSync {
  rss_source_feed: string;     // base64url(feedUrl)
  liked_at: string;            // ISO date
  unliked_at?: string;         // ISO date (if unliked)
}
```

### Conflict Resolution

Last-write-wins with timestamps. Special cases:
- Same episode, different positions → higher position wins
- Subscribe vs unsubscribe → most recent timestamp wins
- Like vs unlike → most recent timestamp wins
- Deleted playlists → soft delete preserved for 45 days

---

## Current Integration Status

### Phase 1: Core Sync - ✅ Complete

| Component | Location | Status |
|-----------|----------|--------|
| Offline sync queue | `balados.app/src/services/storage/syncQueue.ts` | ✅ Complete |
| Sync client API | `balados.app/src/services/sync/client.ts` | ✅ Complete (merged) |
| Conflict resolver | `balados.app/src/services/sync/merger.ts` | ✅ Complete (merged) |
| useSync hook | `balados.app/src/hooks/useSync.ts` | ✅ Complete (merged) |
| Sync Settings UI | `balados.app/src/components/settings/SyncSettings.tsx` | ✅ Complete (merged) |
| Backend sync endpoint | `balados.sync POST /api/v1/sync` | ✅ Complete |
| Backend subscriptions API | `balados.sync /api/v1/subscriptions` | ✅ Complete |
| Backend play API | `balados.sync /api/v1/play` | ✅ Complete |
| RSS CORS proxy | `balados.sync /api/v1/rss/proxy/{feed}` | ✅ Complete |
| Trending API | `balados.sync /api/v1/public/trending/podcasts` | ✅ Complete |
| Health endpoint | `balados.sync GET /api/v1/health` | ✅ Complete |
| Auth refresh endpoint | `balados.sync POST /api/v1/auth/refresh` | ✅ Complete |

### Phase 2: Polish - ✅ Complete

| Component | Location | Status |
|-----------|----------|--------|
| Proxy manager integration | `balados.app/src/services/rss/proxyManager.ts` | ✅ Complete (PR #34) |
| RSS encoding util | `balados.app/src/utils/rssEncoding.ts` | ✅ Complete (PR #34) |
| Background sync (SW) | `balados.app/src/workers/sw.ts` | ✅ Complete (PR #37) |
| Sync status indicator | `balados.app/src/components/library/SyncStatusIcon.tsx` | ✅ Complete (PR #48) |
| Trending page UI | `balados.app/src/components/explorer/Trending.tsx` | ✅ Complete (PR #49) |
| Likes system (backend) | `balados.sync /api/v1/likes` | ✅ Complete (#154, PR #255) |
| Likes system (frontend) | `balados.app/src/hooks/useLike.ts` | ✅ Complete (PR #64) |

---

## Development Keys

Test RSA keypair for local development is in `balados.sync_keys.md`.

---

## Development Workflow Skill

The shared development workflow skill lives in `.claude/skills/development-workflow/`. Trigger it with "continue le workflow".

- **Review is local**: adversarial subagents run the review and post verdicts as PR comments. There is no `needs-claude-review` label and no CI review job.
- Each submodule provides `.claude/workflow-stack.md` with its own project-specific configuration (exact commands, known traps).
- Work sources, in order: open PRs, then the next unchecked task of the [master plan](docs/superpowers/plans/2026-11-05-plan-directeur.md), then open issues (`follow-up`/`bug` first). Feature ideas without a spec go through `superpowers:brainstorming` with Pof first.
- Review worktrees live in each submodule's `.worktrees/` (git-ignored) and install their own dependencies.
- Merge gate includes CI: `~/.config/podclaude/gh.sh pr checks <n>` must show the CI job green on the PR's head commit (`Compile, format and test` in balados.sync, `Lint, build and test` in balados.app, both in the workflow named `Tests`) (not enforced by GitHub yet, see app #80).
- After every merge, the root `CLAUDE.md` and the master plan (file + Claude Doc) are updated before looping.
- The interactive `@claude` workflow (`claude.yml`) in each repo remains active for ad-hoc questions.

---

## Git Workflow

Both projects require:
- **Never commit code changes directly to main.** Always use feature branches, even for fixes. Only tooling config and documentation changes go directly on main.
- Feature branches: `feature/issue-<number>-<slug>`
- Conventional commits
- Author: `--author="Claude <noreply@anthropic.com>"`
- **No human review required.** The local `development-workflow` skill review is sufficient.
  - If the review flags critical/must-fix/should-fix issues: **fix them first**, then merge
  - If the review approves (no blocking issues): **merge directly**
  - After pushing fix commits, re-run the `development-workflow` skill to trigger a new review and wait for approval before merging
- **Review suggestions are never ignored.** Every item from a review must be either:
  - Fixed in the PR before merging (preferred for must-fix/should-fix), OR
  - Tracked as a follow-up issue (for nice-to-have items)
  - Exception: if the PR's source issue is already a follow-up itself, do NOT create follow-up issues (no follow-up of follow-up). In that case, fix everything in the PR.

- **Never ignore failing tests.** If running tests reveals failures (even pre-existing and unrelated to current work), create a GitHub issue to track them.

See individual CLAUDE.md files for project-specific rules.

---

## Current Work

### Completed Integration Milestones

**Backend (balados.sync) - All Done:**
- ✅ [#202](https://github.com/podCloud/balados.sync/issues/202) - `/api/v1/health` endpoint (PR #204)
- ✅ [#203](https://github.com/podCloud/balados.sync/issues/203) - `/api/v1/auth/refresh` endpoint (PR #205)
- ✅ [#197](https://github.com/podCloud/balados.sync/issues/197) - E2E UI testing with Wallaby (PR #198)
- ✅ Follow-up issues from PR #238: [#239](https://github.com/podCloud/balados.sync/issues/239), [#240](https://github.com/podCloud/balados.sync/issues/240)

**Frontend (balados.app) - All Done (merged to main):**
- ✅ [#22](https://github.com/podCloud/balados.app/pull/22) - Sync client API (PR merged)
- ✅ [#23](https://github.com/podCloud/balados.app/issues/23) - Sync Settings UI
- ✅ [#24](https://github.com/podCloud/balados.app/issues/24) - Conflict resolution merger
- ✅ [#25](https://github.com/podCloud/balados.app/issues/25) - useSync React hook

**Recent Frontend Additions (on main):**
- ✅ Local stats page with event logging (#15, PR #28)
- ✅ Event snapshot system for bounded storage (#30, PR #31)
- ✅ "In Progress" page for partially listened episodes (#29, PR #32)
- ✅ Sync server CORS proxy in ProxyManager + URL-safe base64 encoding (#33, PR #34)
- ✅ Background sync via Service Worker (#36, PR #37)
- ✅ Sync status indicator (PR #48)
- ✅ Trending page (PR #49)
- ✅ Enhanced show notes with markdown rendering (#52, PR #53)
- ✅ Likes system with sync integration (PR #64), UI closed as duplicate (#63)
- ✅ Atomic like/queue Dexie transactions (#65, PR #66)
- ✅ Listening history page — filters, stats, streak, pagination (#62, PR #67)

**Recent Backend Additions:**
- ✅ Bounded context aggregate split - User aggregate → 4 aggregates (#148, PR #238)
- ✅ Likes system - Like aggregate, LikeProjector, PopularityProjector, LikeController API (#154, PR #255) — follow-ups: #256, #257, #258, #259, #260 (all closed)
- ✅ Detailed listening history page (#200, PR #252) with async feed titles (#253, PR #254)

### Open Work

State as of 2026-11-06 — Phases 1 (security), 2 (git/docs hygiene) and 3 (dependency updates) of the [master plan](docs/superpowers/plans/2026-11-05-plan-directeur.md) are complete: tasks 3.0 (CI safety net in both repos), 3.1 (backend minor dependency updates), 3.2 (backend majors outside Phoenix: hammer, timex, cachex, gettext, dns_cluster, httpoison → Req), 3.3 (Phoenix 1.8 + LiveView 1.2), 3.4 (frontend minor updates) and 3.5 (frontend majors: Vite 8, vite-plugin-pwa 2, Vitest 5, i18next 26, lucide 1.x, marked 18, jsdom 31, TypeScript 7) are merged — **Phase 3 is complete**; next is Phase 4 (third-party app contract and PodCloud import). Both repos now run their suite on every PR; a PR merges only with the CI check green on its head commit:

- ✅ balados.app [#68](https://github.com/podCloud/balados.app/issues/68) DOMPurify 3.4.17 bump + `renderShowNotes` tests (PR #69, merged 2026-11-05) — follow-up [#71](https://github.com/podCloud/balados.app/issues/71) (force `rel=noopener`)
- ✅ balados.app [#70](https://github.com/podCloud/balados.app/issues/70) sync RSS proxy 401/403/429 surfaced instead of silent public fallback, persisted circuit breaker, transactional settings (PR #72, merged 2026-11-05) — follow-ups [#73](https://github.com/podCloud/balados.app/issues/73) auth flow/refresh token, [#74](https://github.com/podCloud/balados.app/issues/74) no-public-proxy option, [#75](https://github.com/podCloud/balados.app/issues/75) Retry-After in aggregate pages, [#76](https://github.com/podCloud/balados.app/issues/76) flaky ListeningHistory tests, [#77](https://github.com/podCloud/balados.app/issues/77) save/clear race
- ✅ balados.app [#78](https://github.com/podCloud/balados.app/issues/78) CI: lint, build, Vitest on every PR, Node 24.22.2 pinned (PR #79, merged 2026-11-05) — follow-ups [#80](https://github.com/podCloud/balados.app/issues/80) ruleset decision on `main`, [#81](https://github.com/podCloud/balados.app/issues/81) npm allowScripts for esbuild
- ✅ balados.sync [#285](https://github.com/podCloud/balados.sync/issues/285) timex removed, native RFC 1123; Quantum jobs moved to `timezone: :utc` (without tzdata they were silently dropped from the schedule) (PR #287, merged 2026-11-06) — follow-ups [#290](https://github.com/podCloud/balados.sync/issues/290), [#295](https://github.com/podCloud/balados.sync/issues/295)
- ✅ balados.sync [#288](https://github.com/podCloud/balados.sync/issues/288) Cachex 4 (`expire:`, scheduled 500-entry bound; the bump alone silently dropped TTL and bound) (PR #291, merged 2026-11-06)
- ✅ balados.sync [#284](https://github.com/podCloud/balados.sync/issues/284) gettext 1.0 (PR #296, merged 2026-11-06) — found in passing: [#299](https://github.com/podCloud/balados.sync/issues/299) LiveView locale
- ✅ balados.app [#116](https://github.com/podCloud/balados.app/issues/116) TypeScript 7.0.2 (Go compiler), pinned `~7.0.2`, same files and diagnostics as 5.9, type-check 4.1 s → 0.7 s; no tsserver/compiler API any more (PR #117, merged 2026-11-06)
- ✅ balados.app [#113](https://github.com/podCloud/balados.app/issues/113) jsdom 31 + @testing-library/jest-dom 7, `engines.node >= 24.15` (PR #114, merged 2026-11-06) — found in passing [#115](https://github.com/podCloud/balados.app/issues/115)
- ✅ balados.app [#108](https://github.com/podCloud/balados.app/issues/108) artwork (reported by Pof): `itunes:image`/`itunes:duration` were never parsed (CSS selector vs namespaced XML), `Artwork` component with fallback chain (episode → show itunes → show RSS), stale-while-revalidate cache of cross-origin artwork, quota recovery for downloads (PR #109, merged 2026-11-06) — follow-ups [#110](https://github.com/podCloud/balados.app/issues/110) titles/links from namespaced tags, [#111](https://github.com/podCloud/balados.app/issues/111), [#112](https://github.com/podCloud/balados.app/issues/112) show artwork evicted from the 50-entry cache
- ✅ balados.app [#105](https://github.com/podCloud/balados.app/issues/105) marked 18.1.3, no visible change on 1,380 real show notes (PR #106, merged 2026-11-06) — follow-up [#107](https://github.com/podCloud/balados.app/issues/107)
- ✅ balados.app [#103](https://github.com/podCloud/balados.app/issues/103) lucide-react 1.70, canonical icon names; classic podcast logo kept as a local `PodcastIcon` (Pof's choice) after lucide redrew it (PR #104, merged 2026-11-06)
- ✅ balados.app [#97](https://github.com/podCloud/balados.app/issues/97) i18next 26.7 + react-i18next 17.1, first real-i18n integration test (PR #98, merged 2026-11-06) — follow-ups [#99](https://github.com/podCloud/balados.app/issues/99) French `_many`, [#100](https://github.com/podCloud/balados.app/issues/100) regional locales, [#101](https://github.com/podCloud/balados.app/issues/101) pre-commit hook never blocks; found in passing [#102](https://github.com/podCloud/balados.app/issues/102)
- ✅ balados.app [#94](https://github.com/podCloud/balados.app/issues/94) Vitest 5.1 + coverage-v8 5.1, no test change (PR #95, merged 2026-11-06) — follow-up [#96](https://github.com/podCloud/balados.app/issues/96) coverage over all source files
- ✅ balados.app [#91](https://github.com/podCloud/balados.app/issues/91) vite-plugin-pwa 2.0.0 (PR #92, merged 2026-11-06) — follow-up [#93](https://github.com/podCloud/balados.app/issues/93): `registerType: autoUpdate` never activates a new service worker (pre-existing)
- ✅ balados.app [#86](https://github.com/podCloud/balados.app/issues/86) Vite 8.3.6 (Rolldown) + @vitejs/plugin-react 6.2.1; new default browser target Chrome/Edge 111, Firefox 114, Safari 16.4 (PR #88, merged 2026-11-06) — follow-ups [#89](https://github.com/podCloud/balados.app/issues/89) episode list virtualisation, [#90](https://github.com/podCloud/balados.app/issues/90) `<html lang>`; found while checking: [#87](https://github.com/podCloud/balados.app/issues/87) both public CORS proxies are down (corsproxy.io needs an API key, allorigins times out); [#74](https://github.com/podCloud/balados.app/issues/74) default decided by Pof: no public proxy while connected, on by default
- ✅ balados.sync [#302](https://github.com/podCloud/balados.sync/issues/302) Phoenix 1.8.16, LiveView 1.2.13, LiveDashboard 0.9, floki → lazy_html; errors via `used_input?/1`, HEEx curly interpolation, filter form opts out of form recovery, Phoenix 1.8 default CSP pinned (PR #303, merged 2026-11-06) — follow-ups [#304](https://github.com/podCloud/balados.sync/issues/304), [#306](https://github.com/podCloud/balados.sync/issues/306); found in passing [#305](https://github.com/podCloud/balados.sync/issues/305)
- ✅ balados.sync [#292](https://github.com/podCloud/balados.sync/issues/292) httpoison → Req behind `BaladosSyncCore.SafeHttp`, SSRF: redirects re-validated, validator bypasses closed, feed credentials/tokens kept out of logs (PR #298, merged 2026-11-06) — follow-ups [#297](https://github.com/podCloud/balados.sync/issues/297) DNS resolution, [#300](https://github.com/podCloud/balados.sync/issues/300) body cap/deadline/remaining log hygiene, [#301](https://github.com/podCloud/balados.sync/issues/301) joken_jwks without hackney
- ✅ balados.sync [#293](https://github.com/podCloud/balados.sync/issues/293) dns_cluster 0.3; `DNS_CLUSTER_QUERY` was read from the wrong config key and never reached DNSCluster (PR #294, merged 2026-11-06)
- ✅ balados.sync [#286](https://github.com/podCloud/balados.sync/issues/286) Hammer 7: `BaladosSyncWeb.RateLimit` module, exact `Retry-After`, limiter started before the endpoint, `auth_refresh` 100/h now genuinely enforced; flaky [#266](https://github.com/podCloud/balados.sync/issues/266) fixed (PR #289, merged 2026-11-06)
- ✅ balados.app [#82](https://github.com/podCloud/balados.app/issues/82) grouped minor frontend updates (React 19.3, Vite 7.3, Vitest 4.1, Tailwind 4.3, Dexie 4.4…) + [#83](https://github.com/podCloud/balados.app/issues/83) fresh-profile crash fixed with a regression test (PR #84, merged 2026-11-05) — follow-up [#85](https://github.com/podCloud/balados.app/issues/85)
- ✅ balados.sync [#281](https://github.com/podCloud/balados.sync/issues/281) minor dependency updates within current constraints, 40 lock entries incl. bandit 1.14 / plug 1.20 / phoenix 1.7.24 / ecto_sql 3.14 security releases, Elixir floor `~> 1.16` (PR #283, merged 2026-11-05) — pre-existing Wallaby failures tracked in [#282](https://github.com/podCloud/balados.sync/issues/282), push-run anomaly in [#280](https://github.com/podCloud/balados.sync/issues/280)
- ✅ balados.sync [#273](https://github.com/podCloud/balados.sync/issues/273) CI: format, compile `--warnings-as-errors`, full suite on postgres 16, Elixir 1.20.1 / OTP 28 pinned (PR #275, merged 2026-11-05) — follow-ups [#274](https://github.com/podCloud/balados.sync/issues/274) flaky RateLimiterTest, [#276](https://github.com/podCloud/balados.sync/issues/276) live HTTP in tests, [#277](https://github.com/podCloud/balados.sync/issues/277) feed page item without enclosure, [#278](https://github.com/podCloud/balados.sync/issues/278) test-file warnings
- ✅ balados.sync [#264](https://github.com/podCloud/balados.sync/issues/264) RSS proxy requires JWT (`user.sync` or `user.subscriptions.read`), 120 req/min/user, server-wide CORS plug, private cache, unpadded base64url ids (PR #265, merged 2026-11-05) — follow-ups [#267](https://github.com/podCloud/balados.sync/issues/267) last-comma episode ids, [#268](https://github.com/podCloud/balados.sync/issues/268) IP limiter/exact Retry-After, [#269](https://github.com/podCloud/balados.sync/issues/269) gettext template stale, [#270](https://github.com/podCloud/balados.sync/issues/270) CORS before the router, [#271](https://github.com/podCloud/balados.sync/issues/271) rate limit anonymous fetches on public pages (fetching itself is by design), [#272](https://github.com/podCloud/balados.sync/issues/272) pre-commit formatter hook

**Frontend (balados.app):** follow-ups #71, #73–#77 listed above (#73 also covers the OAuth popup flow not matching the server's `/authorize?token=` contract).

**Backend (balados.sync):** besides the #265 follow-ups above and the flaky test [#266](https://github.com/podCloud/balados.sync/issues/266), open issues are longer-horizon discovery/social features not yet scheduled — [#262](https://github.com/podCloud/balados.sync/issues/262) community tags, [#263](https://github.com/podCloud/balados.sync/issues/263) episode reactions (both need a design spec first, see master plan Phase 4), [#153](https://github.com/podCloud/balados.sync/issues/153) collaborative playlists, [#155](https://github.com/podCloud/balados.sync/issues/155) follow other users, [#158](https://github.com/podCloud/balados.sync/issues/158) hashtags/topics, [#201](https://github.com/podCloud/balados.sync/issues/201) MinHash discovery recommendations.

See [docs/INTEGRATION_STATUS.md](docs/INTEGRATION_STATUS.md) for detailed status and [docs/ROADMAP.md](docs/ROADMAP.md) for full roadmap.

---

## Documentation Index

### balados.app
- [docs/VISION.md](balados.app/docs/VISION.md) - Project vision
- [docs/ARCHITECTURE.md](balados.app/docs/ARCHITECTURE.md) - Technical architecture
- [docs/SYNC.md](balados.app/docs/SYNC.md) - Sync strategy
- [docs/OFFLINE.md](balados.app/docs/OFFLINE.md) - PWA & offline
- [docs/BACKGROUND_SYNC.md](balados.app/docs/BACKGROUND_SYNC.md) - Background Sync via Service Worker
- [docs/ROADMAP.md](balados.app/docs/ROADMAP.md) - Development phases

### balados.sync
- [docs/GOALS.md](balados.sync/docs/GOALS.md) - Project objectives
- [docs/FEATURES.md](balados.sync/docs/FEATURES.md) - Implemented features
- [docs/technical/ARCHITECTURE.md](balados.sync/docs/technical/ARCHITECTURE.md) - System architecture
- [docs/technical/AUTH_SYSTEM.md](balados.sync/docs/technical/AUTH_SYSTEM.md) - JWT auth
- [docs/technical/CQRS_PATTERNS.md](balados.sync/docs/technical/CQRS_PATTERNS.md) - CQRS/ES patterns
