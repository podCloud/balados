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

```typescript
// Feed URL → base64
const rssFeed = btoa(feedUrl)
// Example: "https://example.com/feed.xml" → "aHR0cHM6Ly9..."

// Episode ID → base64(guid,enclosureUrl)
const rssItem = btoa(`${guid},${enclosureUrl}`)
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
| GET | `/public/trending/podcasts` | Trending | No |

### Sync Data Types

```typescript
// Subscription
interface SubscriptionSync {
  rss_source_feed: string;     // base64(feedUrl)
  subscribed_at: string;       // ISO date
  unsubscribed_at?: string;    // ISO date (if unsubscribed)
}

// Play Status
interface PlayStatusSync {
  rss_source_feed: string;     // base64(feedUrl)
  rss_source_item: string;     // base64(guid,enclosureUrl)
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

### Implemented

| Component | Location | Status |
|-----------|----------|--------|
| Offline sync queue | `balados.app/src/services/storage/syncQueue.ts` | Complete |
| Sync client API | `balados.app feature/sync` branch | Complete (PR #22) |
| Backend sync endpoint | `balados.sync POST /api/v1/sync` | Complete |
| Backend subscriptions API | `balados.sync /api/v1/subscriptions` | Complete |
| Backend play API | `balados.sync /api/v1/play` | Complete |
| RSS CORS proxy | `balados.sync /api/v1/rss/proxy/{feed}` | Complete |
| Trending API | `balados.sync /api/v1/public/trending/podcasts` | Complete |

### Pending (Frontend)

| Component | Location | Status |
|-----------|----------|--------|
| Sync Settings UI | `src/components/settings/SyncSettings.tsx` | Not started |
| Conflict resolver | `src/services/sync/merger.ts` | Not started |
| useSync hook | `src/hooks/useSync.ts` | Not started |
| Proxy manager integration | Update `proxyManager.ts` | Not started |

---

## Development Keys

Test RSA keypair for local development is in `balados.sync_keys.md`.

---

## Git Workflow

Both projects require:
- Feature branches: `feature/issue-<number>-<slug>`
- PR reviews before merge
- Conventional commits
- Author: `--author="Claude <noreply@anthropic.com>"`

See individual CLAUDE.md files for project-specific rules.

---

## Current Work - Integration Roadmap

### Open Issues to Complete Integration

**Backend (balados.sync):**
- [#202](https://github.com/podCloud/balados.sync/issues/202) - Add `/api/v1/health` endpoint
- [#203](https://github.com/podCloud/balados.sync/issues/203) - Add `/api/v1/auth/refresh` endpoint

**Frontend (balados.app):**
- [#23](https://github.com/podCloud/balados.app/issues/23) - Sync Settings UI component
- [#24](https://github.com/podCloud/balados.app/issues/24) - Conflict resolution merger
- [#25](https://github.com/podCloud/balados.app/issues/25) - useSync React hook

### Priority Order

1. **Backend #202** (health endpoint) - Quick win, unblocks frontend testing
2. **Frontend #23** (SyncSettings UI) - Core user-facing feature
3. **Frontend #24** (merger) - Required for sync logic
4. **Frontend #25** (useSync hook) - Ties everything together
5. **Backend #203** (auth refresh) - Nice to have, can work without

### Sync Feature Branch

The frontend sync client is on `feature/sync` branch (PR #22). Work on issues #23-25 should be done on that branch or merged after it.

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
