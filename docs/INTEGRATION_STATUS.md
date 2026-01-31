# Integration Status - balados.app ↔ balados.sync

**Last updated:** 2026-01-31

---

## Executive Summary

The backend (`balados.sync`) has a complete API for synchronization. The frontend (`balados.app`) has a sync client implementation on the `feature/sync` branch but lacks the UI and conflict resolution components to complete the integration.

**Overall Progress: ~60%**

---

## Component Status Matrix

### Backend (balados.sync) - 🟡 95% Ready

| Component | Endpoint | Status | Notes |
|-----------|----------|--------|-------|
| Health Check | `GET /api/v1/health` | ❌ Missing | Frontend expects this |
| Full Sync | `POST /api/v1/sync` | ✅ Ready | Bidirectional merge |
| Subscriptions List | `GET /api/v1/subscriptions` | ✅ Ready | |
| Subscribe | `POST /api/v1/subscriptions` | ✅ Ready | |
| Unsubscribe | `DELETE /api/v1/subscriptions/{feed}` | ✅ Ready | Soft delete |
| Record Play | `POST /api/v1/play` | ✅ Ready | |
| Update Position | `PUT /api/v1/play/{item}/position` | ✅ Ready | |
| Get Play Status | `GET /api/v1/play` | ✅ Ready | |
| RSS Proxy | `GET /api/v1/rss/proxy/{feed}` | ✅ Ready | 5-min cache |
| Trending | `GET /api/v1/public/trending/podcasts` | ✅ Ready | Public |
| WebSocket Sync | `GET /api/v1/live` | ✅ Ready | Real-time |
| Playlists API | `/api/v1/playlists/*` | ✅ Ready | Full CRUD |
| Collections API | `/api/v1/collections/*` | ✅ Ready | Full CRUD |
| Privacy API | `/api/v1/privacy` | ✅ Ready | Per-feed settings |
| JWT Auth | OAuth flow | ✅ Ready | RS256, scopes |

### Frontend (balados.app) - 🟡 40% Complete

| Component | File | Status | Notes |
|-----------|------|--------|-------|
| Offline Queue | `storage/syncQueue.ts` | ✅ Complete | On main branch |
| Sync Client | `sync/client.ts` | ✅ Complete | On feature/sync |
| Encoding Helpers | `sync/client.ts` | ✅ Complete | On feature/sync |
| Type Converters | `sync/client.ts` | ✅ Complete | On feature/sync |
| Client Tests | `sync/client.test.ts` | ✅ Complete | 29 tests |
| Sync Settings UI | `settings/SyncSettings.tsx` | ❌ Missing | Issue #13 |
| Conflict Resolver | `sync/merger.ts` | ❌ Missing | Issue #14 |
| useSync Hook | `hooks/useSync.ts` | ❌ Missing | Issue #14 |
| Proxy Integration | `rss/proxyManager.ts` | ❌ Missing | Use server proxy |
| OAuth Flow Handler | - | ❌ Missing | Issue #13 |
| Sync Status Indicator | - | ❌ Missing | Issue #13 |

---

## Data Flow Verification

### Subscription Flow

```
Frontend                                    Backend
--------                                    -------
1. User clicks "Subscribe"
2. subscriptionService.subscribe(url)
   ├── Save to IndexedDB
   └── queueSubscribe(url)
3. If online & connected:
   └── POST /api/v1/subscriptions ────────► 4. Dispatch(Subscribe command)
                                            5. Emit UserSubscribed event
                                            6. Project to subscriptions table
   ◄──────────────────────── 201 Created
7. Remove from sync queue
```

### Play Position Flow

```
Frontend                                    Backend
--------                                    -------
1. Audio plays, position updates
2. Every 10s: save position locally
3. If connected, every 30s (throttled):
   └── queuePlayStatus(position)
4. If online:
   └── POST /api/v1/play ─────────────────► 5. Dispatch(RecordPlay command)
                                            6. Emit PlayRecorded event
                                            7. Project to play_statuses
   ◄──────────────────────── 200 OK
8. Remove from sync queue
```

### Full Sync Flow

```
Frontend                                    Backend
--------                                    -------
1. User clicks "Sync Now" or auto-sync
2. Gather local changes since lastSync
3. POST /api/v1/sync { subscriptions,      ► 4. Merge with user's data
                       play_statuses,         5. Apply last-write-wins
                       playlists }            6. Return merged state
   ◄──────────────────────────────────────── { subscriptions, play_statuses,
                                                playlists, synced_at }
7. Apply remote changes locally
8. Update lastSync timestamp
```

---

## Known Issues & Gaps

### Critical (Must Fix)

1. **No OAuth Flow Handler**
   - Frontend cannot obtain JWT tokens
   - Need to handle redirect from server's `/authorize` page
   - Store tokens securely in IndexedDB

2. **No Sync UI**
   - Users cannot connect to a server
   - No visual feedback for sync status
   - No error display for sync failures

3. **No Conflict Resolution**
   - `merger.ts` not implemented
   - Can't handle concurrent edits
   - Position conflicts need "higher wins" logic

### Important (Should Fix)

4. **Proxy Manager Not Integrated**
   - When connected, should use server's CORS proxy first
   - Current code always uses public proxies

5. **No Background Sync**
   - Service Worker sync not triggered
   - Queue only processes on explicit online event

6. **Missing Error Codes**
   - Backend returns error codes but frontend ignores them
   - Should display user-friendly messages

### Nice to Have

7. **No Playlist Sync**
   - Client has types but no implementation
   - Backend ready, frontend needs work

8. **No Real-time Sync**
   - WebSocket endpoint exists on backend
   - Frontend doesn't connect to it

---

## API Compatibility Checklist

| Feature | Frontend Expects | Backend Provides | Match |
|---------|------------------|------------------|-------|
| Health check | `GET /api/v1/health` | ❌ Not implemented | ⚠️ |
| Sync | `POST /api/v1/sync` | `POST /api/v1/sync` | ✅ |
| Subscriptions | `GET/POST/DELETE /api/v1/subscriptions` | Same | ✅ |
| Play status | `POST /api/v1/play` | `POST /api/v1/play` | ✅ |
| Get play status | `GET /api/v1/play/{feed}/{item}` | `GET /api/v1/play` (list only) | ⚠️ |
| RSS proxy | `GET /api/v1/rss/proxy/{feed}` | Same | ✅ |
| Trending | `GET /api/v1/public/trending/podcasts` | Same | ✅ |
| Token refresh | `POST /api/v1/auth/refresh` | ❌ Not implemented | ⚠️ |
| Base64 encoding | `btoa(feedUrl)` | Same | ✅ |
| Episode encoding | `btoa(guid,enclosureUrl)` | Same | ✅ |
| Timestamps | ISO 8601 | ISO 8601 | ✅ |

### Backend Gaps to Fix

1. **`GET /api/v1/health`** - Simple endpoint returning `{ "ok": true }`
2. **`GET /api/v1/play/{feed}/{item}`** - Get specific episode play status (or adjust frontend)
3. **`POST /api/v1/auth/refresh`** - Token refresh endpoint (or remove from frontend)

---

## Test Scenarios

### Scenario 1: First-Time Sync
1. User has local data (subscriptions, play positions)
2. Connects to server
3. All local data uploaded
4. Server data merged (if any)
5. ✅ Expected: No data loss

### Scenario 2: Multi-Device
1. Phone and desktop connected to same server
2. Listen on phone
3. Open desktop
4. ✅ Expected: Position synced

### Scenario 3: Offline Usage
1. Go offline
2. Subscribe to podcast, play episodes
3. Actions queued
4. Go online
5. ✅ Expected: Queue processed, data synced

### Scenario 4: Server Disconnect
1. Connected to server
2. User clicks "Disconnect"
3. ✅ Expected: Local data preserved, no sync

### Scenario 5: Conflict
1. Phone at position 100s
2. Desktop at position 200s
3. Both sync
4. ✅ Expected: Both at 200s (higher wins)

---

## Next Steps

### Phase 1: Complete Core Sync (Issue #13, #14)

1. **Create SyncSettings.tsx**
   - Server URL input
   - Connect/disconnect buttons
   - Sync status display
   - Last sync timestamp

2. **Implement OAuth callback**
   - Handle redirect from `/authorize`
   - Store JWT token
   - Trigger initial sync

3. **Create merger.ts**
   - Last-write-wins logic
   - Position conflict resolution
   - Soft delete handling

4. **Create useSync.ts hook**
   - Expose sync status
   - Manual sync trigger
   - Auto-sync on interval

### Phase 2: Polish (Future)

5. **Integrate with proxyManager**
6. **Add Service Worker background sync**
7. **Implement WebSocket real-time sync**
8. **Add playlist sync**

---

## Files to Create/Modify

### New Files (Frontend)

```
src/
├── components/
│   └── settings/
│       └── SyncSettings.tsx      # Sync connection UI
├── hooks/
│   └── useSync.ts                # React hook for sync
└── services/
    └── sync/
        └── merger.ts             # Conflict resolution
```

### Files to Modify (Frontend)

```
src/
├── components/
│   └── settings/
│       └── Settings.tsx          # Add SyncSettings section
├── services/
│   └── rss/
│       └── proxyManager.ts       # Use server proxy when connected
└── App.tsx                       # Add sync status indicator
```
