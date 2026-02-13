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
npm test             # Run Vitest tests

# Backend (balados.sync)
cd balados.sync
mix deps.get         # Install dependencies
mix db.create        # Create databases
mix db.init          # Initialize event store + migrate
mix phx.server       # Start server (http://localhost:4000)
mix test             # Run tests
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
| GET | `/rss/proxy/{feed}` | CORS proxy | JWT |
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
```

### Conflict Resolution

Last-write-wins with timestamps. Special cases:
- Same episode, different positions → higher position wins
- Subscribe vs unsubscribe → most recent timestamp wins
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

### Pending (Phase 2: Polish)

| Component | Location | Status |
|-----------|----------|--------|
| Proxy manager integration | `balados.app/src/services/rss/proxyManager.ts` | ✅ Complete (PR #34) |
| RSS encoding util | `balados.app/src/utils/rssEncoding.ts` | ✅ Complete (PR #34) |
| Background sync (SW) | Service Worker sync | Not started |
| Sync status indicator | App header icon | Not started |
| Trending page UI | `src/components/` (Issue #16) | Not started |

---

## Development Keys

Test RSA keypair for local development is in `balados.sync_keys.md`.

---

## Git Workflow

Both projects require:
- Feature branches: `feature/issue-<number>-<slug>`
- Conventional commits
- Author: `--author="Claude <noreply@anthropic.com>"`
- **No human review required.** The CI bot review (Claude Code Review workflow) is sufficient.
  - If the bot review flags critical/must-fix/should-fix issues: **fix them first**, then merge
  - If the bot review approves (no blocking issues): **merge directly**
  - After pushing fix commits, wait for the new bot review before merging
- **Follow-up issues**: When the bot review suggests nice-to-have or optional improvements, create follow-up issues for them.
  - Exception: if the PR's source issue is already a follow-up itself, do NOT create follow-up issues (no follow-up of follow-up)

See individual CLAUDE.md files for project-specific rules.

---

## Current Work

### Completed Integration Milestones

**Backend (balados.sync) - All Done:**
- ✅ [#202](https://github.com/podCloud/balados.sync/issues/202) - `/api/v1/health` endpoint (PR #204)
- ✅ [#203](https://github.com/podCloud/balados.sync/issues/203) - `/api/v1/auth/refresh` endpoint (PR #205)

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

### Open Work

**Frontend (balados.app):**
- [#16](https://github.com/podCloud/balados.app/issues/16) - Trending page (Phase 5.2, needs sync server)

**Backend (balados.sync):**
- [PR #206](https://github.com/podCloud/balados.sync/pull/206) - Simplify EventStore API in SnapshotWorker (#146)
- [PR #198](https://github.com/podCloud/balados.sync/pull/198) - E2E UI testing with Wallaby (#197)
- Follow-up issues from PR #206: #210, #211, #212

See [docs/INTEGRATION_STATUS.md](docs/INTEGRATION_STATUS.md) for detailed status and [docs/ROADMAP.md](docs/ROADMAP.md) for full roadmap.

---

## Documentation Index

### balados.app
- [docs/VISION.md](balados.app/docs/VISION.md) - Project vision
- [docs/ARCHITECTURE.md](balados.app/docs/ARCHITECTURE.md) - Technical architecture
- [docs/SYNC.md](balados.app/docs/SYNC.md) - Sync strategy
- [docs/OFFLINE.md](balados.app/docs/OFFLINE.md) - PWA & offline
- [docs/ROADMAP.md](balados.app/docs/ROADMAP.md) - Development phases

### balados.sync
- [docs/GOALS.md](balados.sync/docs/GOALS.md) - Project objectives
- [docs/FEATURES.md](balados.sync/docs/FEATURES.md) - Implemented features
- [docs/technical/ARCHITECTURE.md](balados.sync/docs/technical/ARCHITECTURE.md) - System architecture
- [docs/technical/AUTH_SYSTEM.md](balados.sync/docs/technical/AUTH_SYSTEM.md) - JWT auth
- [docs/technical/CQRS_PATTERNS.md](balados.sync/docs/technical/CQRS_PATTERNS.md) - CQRS/ES patterns
