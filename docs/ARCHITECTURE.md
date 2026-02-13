# Balados Ecosystem Architecture

**Last updated:** 2026-02-12

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                           USER DEVICES                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │   Mobile    │  │   Desktop   │  │   Tablet    │                  │
│  │    PWA      │  │    PWA      │  │    PWA      │                  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                  │
│         │                │                │                          │
│         └────────────────┼────────────────┘                          │
│                          │                                           │
│                   balados.app (React PWA)                            │
│                          │                                           │
└──────────────────────────┼───────────────────────────────────────────┘
                           │
                           │ HTTPS (REST + WebSocket)
                           │
┌──────────────────────────┼───────────────────────────────────────────┐
│                          ▼                                           │
│                   ┌─────────────┐                                    │
│                   │   Phoenix   │                                    │
│                   │   Router    │                                    │
│                   └──────┬──────┘                                    │
│                          │                                           │
│         ┌────────────────┼────────────────┐                          │
│         │                │                │                          │
│         ▼                ▼                ▼                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │  REST API   │  │  WebSocket  │  │   Web UI    │                  │
│  │ Controllers │  │   Handler   │  │   Pages     │                  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                  │
│         │                │                │                          │
│         └────────────────┼────────────────┘                          │
│                          │                                           │
│                          ▼                                           │
│                   ┌─────────────┐                                    │
│                   │  Commanded  │                                    │
│                   │ (Dispatcher)│                                    │
│                   └──────┬──────┘                                    │
│                          │                                           │
│                          ▼                                           │
│                   ┌─────────────┐                                    │
│                   │    User     │                                    │
│                   │  Aggregate  │                                    │
│                   └──────┬──────┘                                    │
│                          │                                           │
│         ┌────────────────┼────────────────┐                          │
│         │                │                │                          │
│         ▼                ▼                ▼                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │ EventStore  │  │ Projectors  │  │  Snapshots  │                  │
│  │ (Postgres)  │  │             │  │             │                  │
│  └─────────────┘  └──────┬──────┘  └─────────────┘                  │
│                          │                                           │
│                          ▼                                           │
│                   ┌─────────────┐                                    │
│                   │ Projections │                                    │
│                   │ (Postgres)  │                                    │
│                   └─────────────┘                                    │
│                                                                      │
│                   balados.sync (Elixir/Phoenix)                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Frontend Architecture (balados.app)

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| UI | React 19 | Component framework |
| Build | Vite 7 | Bundler with HMR |
| Styling | Tailwind CSS 4 | Utility-first CSS |
| State (Server) | TanStack Query | Data fetching, caching |
| State (Local) | React Context | Global UI state |
| Storage | Dexie.js (IndexedDB) | Persistent local data |
| Offline | Workbox | Service Worker, caching |
| i18n | i18next | Internationalization |

### Component Architecture

```
App.tsx
├── PlayerProvider (Context)
│   └── DownloadProvider (Context)
│       ├── TabBar
│       │   └── [Navigation]
│       ├── Library
│       │   └── SubscriptionItem[]
│       ├── PodcastDetail
│       │   └── EpisodeList
│       ├── EpisodePlayer
│       │   ├── PlayerControls
│       │   └── MiniPlayer
│       ├── Explorer
│       ├── InProgress
│       ├── Stats
│       ├── Settings
│       │   ├── SyncSettings
│       │   └── StorageSettings
│       └── Debug
```

### Data Flow

```
User Action
    │
    ▼
React Component
    │
    ├──► TanStack Query (for RSS fetches)
    │         │
    │         ▼
    │    ProxyManager ──► RSS Feed
    │         │
    │         ▼
    │    feedCache (IndexedDB)
    │
    └──► Storage Service
              │
              ├──► IndexedDB (Dexie)
              │    ├── subscriptions
              │    ├── playStatuses
              │    ├── downloads
              │    ├── events
              │    └── settings
              │
              └──► SyncQueue
                       │
                       ▼ (when online + connected)
                   SyncClient ──► balados.sync API
```

### IndexedDB Schema

```typescript
// Dexie database schema
db.version(1).stores({
  subscriptions: '++id, url, addedAt',
  playStatuses: 'episodeId, feedUrl, updatedAt',
  events: '++id, type, feedUrl, timestamp',
  feedCache: 'url, cachedAt',
  downloads: 'episodeId, feedUrl, downloadedAt',
  settings: 'key',
  syncQueue: '++id, action, createdAt'
});
```

### Offline-First Strategy

1. **Read Operations**
   - Check IndexedDB first
   - Fall back to network if stale
   - Cache network responses

2. **Write Operations**
   - Always write to IndexedDB immediately
   - Queue for sync if connected to server
   - Process queue when online

3. **Service Worker**
   - Precache static assets
   - Runtime cache for RSS feeds
   - Background sync for queue

---

## Backend Architecture (balados.sync)

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Runtime | Elixir/OTP 25+ | Functional, concurrent |
| Web | Phoenix 1.7 | REST API, WebSocket |
| CQRS | Commanded | Command dispatch |
| Events | EventStore | Event persistence |
| Database | PostgreSQL 14+ | Projections, system data |
| Cache | Cachex | RSS feed caching |
| Auth | Joken (JWT) | RS256 tokens |

### Umbrella Structure

```
balados.sync/
├── apps/
│   ├── balados_sync_core/        # Domain layer
│   │   ├── aggregates/           # User aggregate
│   │   ├── commands/             # Subscribe, RecordPlay, etc.
│   │   ├── events/               # UserSubscribed, PlayRecorded, etc.
│   │   ├── dispatcher.ex         # Command routing
│   │   └── event_store.ex        # EventStore config
│   │
│   ├── balados_sync_projections/ # Read model layer
│   │   ├── projectors/           # Event handlers
│   │   ├── repo/                 # Ecto repositories
│   │   │   ├── system_repo.ex    # Users, tokens
│   │   │   └── projections_repo.ex # Read models
│   │   └── schemas/              # Ecto schemas
│   │
│   ├── balados_sync_web/         # Interface layer
│   │   ├── controllers/          # REST + Web
│   │   ├── plugs/                # Auth middleware
│   │   ├── router.ex             # Routes
│   │   └── channels/             # WebSocket
│   │
│   └── balados_sync_jobs/        # Background workers
│       ├── snapshot_worker.ex
│       ├── cleanup_worker.ex
│       └── popularity_worker.ex
│
└── config/                       # Shared configuration
```

### CQRS/Event Sourcing Flow

```
1. API Request
       │
       ▼
2. Controller validates input
       │
       ▼
3. Dispatcher.dispatch(command)
       │
       ▼
4. User Aggregate handles command
       │ (validates business rules)
       ▼
5. Emit event(s)
       │
       ▼
6. EventStore persists (immutable)
       │
       ├─────────────────────────────┐
       ▼                             ▼
7. Projectors handle           8. Snapshots
   (update read models)           (periodic)
       │
       ▼
9. Projections (PostgreSQL)
       │
       ▼
10. Query via Ecto
```

### Database Architecture

```
PostgreSQL Instance
├── Schema: events (EventStore)
│   ├── streams
│   ├── events
│   ├── snapshots
│   └── subscriptions
│
├── Schema: system (Permanent data)
│   ├── users
│   ├── app_tokens
│   └── play_tokens
│
└── Schema: public (Projections)
    ├── subscriptions
    ├── play_statuses
    ├── playlists
    ├── playlist_items
    ├── collections
    ├── collection_subscriptions
    ├── user_privacy
    ├── podcast_popularity
    ├── episode_popularity
    ├── public_events
    ├── enriched_podcasts
    └── podcast_ownership_claims
```

### Event Types

| Event | Trigger | Data |
|-------|---------|------|
| UserSubscribed | Subscribe command | feed_url, timestamp |
| UserUnsubscribed | Unsubscribe command | feed_url, timestamp |
| PlayRecorded | RecordPlay command | item_id, position, played |
| PositionUpdated | UpdatePosition command | item_id, position |
| PlaylistCreated | CreatePlaylist command | id, name, type |
| PlaylistItemAdded | AddToPlaylist command | playlist_id, item |
| CollectionCreated | CreateCollection command | id, title |
| PrivacyChanged | ChangePrivacy command | feed_url, level |
| UserCheckpoint | Background worker | Snapshot data |

---

## API Design

### RESTful Endpoints

```
/api/v1/
├── health                    # GET - Health check
├── sync                      # POST - Full sync
├── subscriptions             # GET, POST - List, add
│   └── /{feed}               # DELETE - Remove
├── play                      # GET, POST - Status, record
│   └── /{item}/position      # PUT - Update position
├── playlists                 # GET, POST - List, create
│   └── /{id}                 # GET, PATCH, DELETE
│       └── /items            # POST - Add item
├── collections               # GET, POST - List, create
│   └── /{id}                 # GET, PATCH, DELETE
│       └── /feeds            # POST - Add feed
├── privacy                   # GET, PUT - Settings
├── auth/
│   └── refresh               # POST - Token validation/refresh
├── rss/
│   └── proxy/{feed}          # GET - CORS proxy
└── public/
    ├── trending/podcasts     # GET - Public trending
    └── timeline              # GET - Public activity
```

### WebSocket Protocol

```
Connection: wss://{server}/api/v1/live

Messages (Client → Server):
{ "type": "auth", "token": "jwt..." }
{ "type": "record_play", "feed": "...", "item": "...", "position": 120 }

Messages (Server → Client):
{ "type": "auth_ok" }
{ "type": "sync_update", "data": {...} }
{ "type": "error", "message": "..." }
```

---

## Authentication Flow

```
┌─────────────┐     1. Generate keypair    ┌─────────────────────────┐
│ balados.app │ ─────────────────────────► │ Local Storage (private) │
└─────────────┘                            └─────────────────────────┘
       │
       │ 2. Create auth JWT (signed with private key)
       │    { iss: "app_id", sub: "auth_request", public_key: "..." }
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Redirect to: {server}/authorize?token={jwt}                        │
└─────────────────────────────────────────────────────────────────────┘
       │
       │ 3. User logs in, approves app
       │
       ▼
┌──────────────┐    4. Store public key     ┌─────────────────────────┐
│ balados.sync │ ──────────────────────────►│ app_tokens table        │
└──────────────┘                            │ (public_key, scopes)    │
       │                                    └─────────────────────────┘
       │
       │ 5. Redirect to: {callback}?approved=true
       │
       ▼
┌─────────────┐
│ balados.app │
└─────────────┘
       │
       │ 6. For each API request:
       │    Create JWT signed with private key
       │    Authorization: Bearer {jwt}
       │
       ▼
┌──────────────┐    7. Verify signature     ┌─────────────────────────┐
│ balados.sync │ ◄─────────────────────────►│ app_tokens table        │
└──────────────┘    using stored public key └─────────────────────────┘
```

---

## Sync Protocol

### Full Sync Request

```json
POST /api/v1/sync
{
  "subscriptions": [
    {
      "rss_source_feed": "base64(feed_url)",
      "subscribed_at": "2025-01-01T00:00:00Z",
      "unsubscribed_at": null
    }
  ],
  "play_statuses": [
    {
      "rss_source_feed": "base64(feed_url)",
      "rss_source_item": "base64(guid,enclosure_url)",
      "position": 120,
      "played": false,
      "updated_at": "2025-01-15T10:00:00Z"
    }
  ]
}
```

### Full Sync Response

```json
{
  "subscriptions": [...],
  "play_statuses": [...],
  "synced_at": "2025-01-15T10:05:00Z"
}
```

### Incremental Sync

Add `since` parameter to get only changes after that timestamp:

```json
POST /api/v1/sync
{
  "since": "2025-01-15T10:00:00Z",
  "subscriptions": [...only_changed...],
  "play_statuses": [...only_changed...]
}
```

### Conflict Resolution

| Data Type | Strategy | Winner |
|-----------|----------|--------|
| Subscription | Timestamp | Most recent `subscribed_at` or `unsubscribed_at` |
| Play Position | Hybrid | Higher position OR more recent `updated_at` |
| Playlist | Merge | Items merged, metadata uses most recent |
| Deletion | Soft Delete | Preserved for 45 days, then purged |

---

## Caching Strategy

### Frontend (balados.app)

| Data | Cache Location | TTL | Strategy |
|------|----------------|-----|----------|
| RSS Feeds | IndexedDB (feedCache) | 30 min | Stale-while-revalidate |
| Subscriptions | IndexedDB | ∞ | Cache first |
| Play Status | IndexedDB | ∞ | Cache first |
| Static Assets | Service Worker | ∞ | Precache |
| Audio Files | Cache Storage | ∞ | On-demand |

### Backend (balados.sync)

| Data | Cache Location | TTL | Strategy |
|------|----------------|-----|----------|
| Raw RSS XML | Cachex | 5 min | Read-through |
| Parsed Metadata | Cachex | 5 min | Read-through |
| Popularity Scores | PostgreSQL | 5 min | Background refresh |

---

## Error Handling

### Backend Error Codes

| Code | Meaning | HTTP Status |
|------|---------|-------------|
| AUTH_INVALID_TOKEN | Invalid JWT | 401 |
| AUTH_EXPIRED_TOKEN | Expired JWT | 401 |
| AUTH_INSUFFICIENT_SCOPE | Missing scope | 403 |
| RESOURCE_NOT_FOUND | Entity not found | 404 |
| VALIDATION_ERROR | Invalid input | 400 |
| RATE_LIMITED | Too many requests | 429 |
| INTERNAL_ERROR | Server error | 500 |

### Frontend Error Recovery

```typescript
// Retry strategy for API calls
const RETRY_CONFIG = {
  maxRetries: 3,
  baseDelay: 1000,
  retryableCodes: [408, 429, 500, 502, 503, 504],
  exponentialBackoff: true
};
```

---

## Security Considerations

### Authentication
- RS256 asymmetric JWT (public key stored server-side)
- Scoped permissions (read/write per resource)
- Token expiration with refresh capability

### Data Protection
- CORS configured for allowed origins
- Rate limiting (30 req/min for sync)
- Request body size limits
- SSRF prevention for RSS proxy

### Privacy
- Per-feed privacy levels (public/anonymous/private)
- Public events filtered by privacy
- User profile visibility toggle
