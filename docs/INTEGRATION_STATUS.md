# Integration Status - balados.app ↔ balados.sync

**Last updated:** 2026-02-12

---

## Executive Summary

The backend (`balados.sync`) has a complete API for synchronization including health, auth/refresh, sync, subscriptions, play, RSS proxy, and trending endpoints. The frontend (`balados.app`) has full Phase 1 sync implementation **merged to main** including the sync client, settings UI, conflict resolution, and React hook. Additional features (local stats, event snapshots, in-progress page) have also been shipped.

**Phase 1 (Core Sync): ✅ Complete**
**Overall Progress: ~90%** (Phase 2 polish items remaining)

---

## Component Status Matrix

### Backend (balados.sync) - ✅ 100% Ready

| Component | Endpoint | Status | Notes |
|-----------|----------|--------|-------|
| Health Check | `GET /api/v1/health` | ✅ Ready | Returns `{ ok: true, version: "..." }` |
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

### Frontend (balados.app) - 🟢 Phase 1 Complete (on main)

| Component | File | Status | Notes |
|-----------|------|--------|-------|
| Offline Queue | `storage/syncQueue.ts` | ✅ Complete | Merged to main |
| Sync Client | `sync/client.ts` | ✅ Complete | Merged (PR #22) |
| Encoding Helpers | `sync/client.ts` | ✅ Complete | Merged (PR #22) |
| Type Converters | `sync/client.ts` | ✅ Complete | Merged (PR #22) |
| Client Tests | `sync/client.test.ts` | ✅ Complete | 29 tests |
| Sync Settings UI | `settings/SyncSettings.tsx` | ✅ Complete | Issue #23, merged |
| Conflict Resolver | `sync/merger.ts` | ✅ Complete | Issue #24, 22 tests, merged |
| useSync Hook | `hooks/useSync.ts` | ✅ Complete | Issue #25, merged |
| OAuth Flow Handler | `SyncSettings.tsx` | ✅ Complete | Popup + manual token |
| Local Stats | `components/stats/` | ✅ Complete | Issue #15, PR #28 |
| Event Snapshots | `services/storage/` | ✅ Complete | PR #31 |
| In Progress Page | `components/inProgress/` | ✅ Complete | PR #32 |
| Proxy Integration | `rss/proxyManager.ts` | ❌ Missing | Phase 2 - Use server proxy |
| Sync Status Indicator | - | ❌ Missing | Phase 2 - Nice to have |

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
7. Apply remote changes locally (merger.ts)
8. Update lastSync timestamp
```

---

## Known Issues & Gaps

### Resolved ✅

1. ~~**No OAuth Flow Handler**~~ → Implemented in SyncSettings.tsx
2. ~~**No Sync UI**~~ → SyncSettings.tsx with status, connect/disconnect
3. ~~**No Conflict Resolution**~~ → merger.ts with full test coverage
4. ~~**No Health Endpoint**~~ → Backend now has `/api/v1/health`

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

9. **No Sync Status Indicator**
   - Would be nice in app header
   - Shows connected/syncing/pending status

---

## API Compatibility Checklist

| Feature | Frontend Expects | Backend Provides | Match |
|---------|------------------|------------------|-------|
| Health check | `GET /api/v1/health` | `GET /api/v1/health` | ✅ |
| Sync | `POST /api/v1/sync` | `POST /api/v1/sync` | ✅ |
| Subscriptions | `GET/POST/DELETE /api/v1/subscriptions` | Same | ✅ |
| Play status | `POST /api/v1/play` | `POST /api/v1/play` | ✅ |
| Get play status | `GET /api/v1/play/{feed}/{item}` | `GET /api/v1/play` (list only) | ⚠️ |
| RSS proxy | `GET /api/v1/rss/proxy/{feed}` | Same | ✅ |
| Trending | `GET /api/v1/public/trending/podcasts` | Same | ✅ |
| Token refresh | `POST /api/v1/auth/refresh` | `POST /api/v1/auth/refresh` | ✅ |
| Base64 encoding | `btoa(feedUrl)` | Same | ✅ |
| Episode encoding | `btoa(guid,enclosureUrl)` | Same | ✅ |
| Timestamps | ISO 8601 | ISO 8601 | ✅ |

### Remaining Backend Gaps

1. **`GET /api/v1/play/{feed}/{item}`** - Get specific episode play status (or adjust frontend)

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

## Completed Implementation

### Sync Files (all merged to main)

```
src/
├── components/
│   ├── settings/
│   │   ├── SyncSettings.tsx      # ✅ Sync connection UI
│   │   └── Settings.tsx          # ✅ Added SyncSettings section
│   ├── stats/                    # ✅ Local stats page (PR #28)
│   └── inProgress/               # ✅ In progress page (PR #32)
├── hooks/
│   └── useSync.ts                # ✅ React hook for sync
├── services/
│   ├── sync/
│   │   ├── client.ts             # ✅ API client
│   │   ├── client.test.ts        # ✅ 29 tests
│   │   ├── merger.ts             # ✅ Conflict resolution
│   │   └── merger.test.ts        # ✅ 22 tests
│   └── i18n/locales/
│       ├── en.json               # ✅ Sync + stats translations
│       └── fr.json               # ✅ Sync + stats translations
└── types/
    └── index.ts                  # ✅ Added lastSyncAt to AppSettings
```

---

## Next Steps

### Phase 1: Complete Core Sync ✅

1. ~~**Create SyncSettings.tsx**~~ → Done
2. ~~**Implement OAuth callback**~~ → Done
3. ~~**Create merger.ts**~~ → Done
4. ~~**Create useSync.ts hook**~~ → Done

### Phase 2: Polish (Future)

5. **Integrate with proxyManager** - Use server proxy when connected
6. **Add Service Worker background sync** - Process queue periodically
7. **Implement WebSocket real-time sync** - Live updates
8. **Add playlist sync** - Full CRUD
9. **Add sync status indicator** - App header icon

---

## PR Status

| PR | Title | Branch | Status |
|----|-------|--------|--------|
| #22 | feat(sync): add balados.sync API client | feature/sync | ✅ Merged |
| #26 | fix(sync): critical bugs | fix/sync-critical-bugs | ✅ Merged |
| #27 | fix: PR #26 review followup | fix/pr26-review-followup | ✅ Merged |
| #28 | feat(stats): local statistics page | feature/issue-15-local-stats | ✅ Merged |
| #31 | feat(events): snapshot system | feature/issue-30-event-snapshots | ✅ Merged |
| #32 | feat(ui): in progress page | feature/issue-29-in-progress-page | ✅ Merged |

All sync functionality and recent features are on main.
