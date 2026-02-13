# Integration Roadmap - Balados Ecosystem

**Last updated:** 2026-02-12

---

## Overview

This roadmap covers completing the integration between `balados.app` (frontend) and `balados.sync` (backend).

### Current State

- **Backend**: Production-ready API with full CQRS/ES architecture, all sync endpoints complete
- **Frontend**: Phase 1 complete (sync client, settings UI, merger, useSync hook all merged to main). Additional features shipped: local stats, event snapshots, in-progress page.

---

## Phase 1: Core Integration (Priority: Critical) - ✅ COMPLETE

**Goal:** Users can connect to a sync server and sync their data.

All Phase 1 items have been implemented and merged to main.

### 1.1 OAuth Flow Handler - ✅ Done

**Files:** `balados.app/src/components/settings/SyncSettings.tsx`

- Popup + manual token OAuth flow
- Token stored securely in IndexedDB

### 1.2 Sync Settings UI - ✅ Done (Issue #23, PR #22)

**Files:** `balados.app/src/components/settings/SyncSettings.tsx`

- Server URL input with validation
- Connect/Disconnect buttons
- Connection status indicator
- Last sync timestamp
- "Sync Now" manual trigger

### 1.3 Conflict Resolution Logic - ✅ Done (Issue #24, PR #22)

**Files:** `balados.app/src/services/sync/merger.ts` (22 tests)

- Last-write-wins with timestamps
- Higher position wins for play positions
- Soft delete preservation

### 1.4 useSync Hook - ✅ Done (Issue #25, PR #22)

**Files:** `balados.app/src/hooks/useSync.ts`

- Components can access sync status
- Manual sync triggerable
- Error states exposed

### 1.5 Backend Endpoints - ✅ Done

- Health endpoint (Issue #202, PR #204)
- Auth refresh endpoint (Issue #203, PR #205)

---

## Phase 2: Enhanced Sync (Priority: High)

**Goal:** Robust syncing with better UX and reliability.

### 2.1 Server Proxy Integration

**Status:** Not started
**Effort:** Small
**Files:** `balados.app/src/services/rss/proxyManager.ts`

When connected to sync server:
1. Use server's CORS proxy first (`/api/v1/rss/proxy/{feed}`)
2. Fall back to configured proxies on failure
3. Benefits: 5-min cache, no rate limits, reliability

**Acceptance Criteria:**
- [ ] Connected users use server proxy
- [ ] Fallback works when proxy fails
- [ ] Disconnected users use public proxies

### 2.2 Sync Status Indicator

**Status:** Not started
**Effort:** Small
**Files:** `balados.app/src/components/ui/SyncIndicator.tsx`

Visual indicator in app header/navigation:
- ○ Not connected (grey)
- ● Connected, up to date (green)
- ◐ Syncing in progress (animated)
- ◑ Pending items (orange)
- ● Error (red, clickable for details)

**Acceptance Criteria:**
- [ ] Indicator visible on all screens
- [ ] Accurate reflection of sync state
- [ ] Click shows sync details/errors

### 2.3 Background Sync via Service Worker

**Status:** Not started
**Effort:** Medium
**Files:** `balados.app/src/workers/sw.ts`

- Register for Background Sync API
- Process sync queue when connectivity restored
- Periodic sync when app is open
- Handle sync failures gracefully

**Acceptance Criteria:**
- [ ] Offline actions sync when back online
- [ ] Works even if app is closed
- [ ] No duplicate sync attempts

### 2.4 Error Handling & User Feedback

**Status:** Not started
**Effort:** Small
**Files:** Various components

- Toast notifications for sync events
- Clear error messages (not technical jargon)
- Retry options for failed syncs
- Offline mode indication

**Acceptance Criteria:**
- [ ] Users know when sync succeeds/fails
- [ ] Actionable error messages
- [ ] Retry button for failures

---

## Phase 3: Advanced Features (Priority: Medium)

**Goal:** Full feature parity with backend capabilities.

### 3.1 Playlist Sync

**Status:** Types exist, no implementation
**Effort:** Medium

- Sync playlists bidirectionally
- Handle playlist item ordering
- Support playlist deletion (soft delete)

### 3.2 Collection Sync

**Status:** Not started
**Effort:** Medium

- Sync collections (feed groups)
- Maintain collection order
- Handle nested feeds

### 3.3 Real-time Sync via WebSocket

**Status:** Backend ready, frontend missing
**Effort:** Medium
**Endpoint:** `wss://{server}/api/v1/live`

- Connect WebSocket when app opens
- Receive instant updates from other devices
- Send play events in real-time
- Reconnect on connection loss

### 3.4 Trending Integration

**Status:** Backend ready, frontend partial
**Effort:** Small

- Fetch trending from connected server
- Display in Explorer/Discover section
- Show subscriber counts

---

## Phase 4: Polish & Optimization (Priority: Low)

**Goal:** Production-ready, performant, accessible.

### 4.1 Token Management

- Auto-refresh before expiration
- Secure storage best practices
- Handle revoked tokens

### 4.2 Sync Optimization

- Delta sync (only changed items)
- Compression for large payloads
- Batch operations

### 4.3 Multi-Account Support

- Switch between servers
- Separate data per account
- Account management UI

### 4.4 Privacy Controls

- Sync privacy settings
- Per-feed privacy levels
- Anonymous mode

---

## Timeline Estimate

| Phase | Scope | Complexity |
|-------|-------|------------|
| Phase 1 | Core Integration | Medium |
| Phase 2 | Enhanced Sync | Medium |
| Phase 3 | Advanced Features | High |
| Phase 4 | Polish | Low |

---

## Dependencies

### Frontend Requirements
- [x] React 19 with hooks
- [x] Dexie.js for IndexedDB
- [x] TanStack Query
- [x] Service Worker setup
- [x] Merge feature/sync branch (PR #22 merged)

### Backend Requirements
- [x] All sync endpoints
- [x] JWT authentication
- [x] CORS proxy
- [x] WebSocket support
- [x] Rate limiting

### Shared
- [x] Base64 encoding convention
- [x] Timestamp format (ISO 8601)
- [x] Conflict resolution strategy

---

## Success Metrics

### Phase 1 Complete When:
- User can connect to sync.balados.app
- Subscriptions sync bidirectionally
- Play positions sync bidirectionally
- Offline changes sync when online
- Disconnect preserves all local data

### Full Integration Complete When:
- All phases implemented
- E2E tests pass
- 0 data loss scenarios
- <100ms sync latency (connected)
- Works across mobile/desktop
