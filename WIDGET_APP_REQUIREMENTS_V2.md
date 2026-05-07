# Multi-Widget Shared Status App
## Complete Technical Requirements Document

**Version:** 2.0  
**Date:** May 2026  
**Status:** Ready for Development  
**Primary Focus:** Real-time Status Notifications with Mutually-Controlled Toggles (Initiator-Only-Off)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Core Use Case](#core-use-case)
3. [Architecture Overview](#architecture-overview)
4. [Technology Stack](#technology-stack)
5. [Frontend Requirements (Android/Kotlin)](#frontend-requirements-androidkotlin)
6. [Backend Requirements](#backend-requirements)
7. [Database Schema](#database-schema)
8. [API Specifications](#api-specifications)
9. [State Management](#state-management)
10. [Synchronization Protocol](#synchronization-protocol)
11. [Error Handling & Edge Cases](#error-handling--edge-cases)
12. [Security Requirements](#security-requirements)
13. [Testing Strategy](#testing-strategy)
14. [Deployment & DevOps](#deployment--devops)
15. [Performance Targets](#performance-targets)

---

## Executive Summary

This application enables users to create shared emoji widgets that communicate temporary status changes to other users. The core mechanic is: **Any user with access can turn a widget ON, but only the person who turned it ON can turn it OFF.**

**Real-world use case:** User A creates a "Manager in the room" widget and shares it with User B. If User A turns it ON, User B's phone immediately shows the floating emoji (e.g., 🤫). User B cannot turn it OFF; only User A can. Conversely, if User B turns it ON, User A sees it and cannot turn it OFF until User B does. It is mutually controlled, but strictly "initiator-only-off" for any active session.

### Key Constraints
- **Multiple widgets per user**: Each user owns multiple widgets (e.g., 😴 "sleeping", 🤫 "busy", 🏃 "gym") and can share them.
- **Shared access**: Each widget can be shared with 1+ users. Anyone with access can toggle the state ON.
- **Initiator-controlled OFF**: If a widget is ON, only the specific user who initiated the current ON state can toggle it OFF. Other users see the state but cannot override it.
- **Shared visibility**: All shared users see the same state in real-time
- **Real-time notification**: State changes propagate within 100-500ms (best-effort, not guaranteed)
- **Persistent overlay**: Widget state remains visible when app is closed/backgrounded
- **Offline support**: App queues changes when offline, syncs when network returns
- **Minimal battery impact**: Efficient connection management; avoid aggressive background polling

---

## Core Use Case

### User Flow Example

**Setup Phase:**
1. User A creates widget "😴 sleeping" (emojis chosen for each status)
2. User A shares it with User B (and optionally User C, D, etc.)
3. Users B, C, D install app, authenticate, and see User A's widgets in a "Shared with Me" list

**Active Notification Phase:**
1. Any user (e.g., User A) opens app, sees the shared widgets listed
2. User A taps the 😴 widget → toggles to ON
3. Within 500ms:
   - User A's app updates immediately
   - WebSocket broadcasts to User B and C
   - User B and C's phones receive notification
   - 😴 emoji floats on their screens (top-right corner, or configurable position)
4. User B and C **cannot** tap the emoji to toggle it off
   - Since User A turned it ON, only User A can turn it OFF.
   - If B or C taps it, they get a toast message: "Only User A can turn this off"
   - Any tap from them might just dismiss it temporarily locally (doesn't change global state)
5. User A taps the 😴 widget again → toggles to OFF
6. Within 500ms:
   - All users' overlays disappear
   - User B and C see the notification cleared

**Behavior:**
- Any shared user can toggle a widget state to ON.
- Once a widget is ON, it is "locked" to the initiator. Only the initiator can toggle it OFF.
- Other users are passive recipients of the status information during that active session.
- State is temporary; widgets naturally expire if not explicitly toggled (optional feature)

---

## Architecture Overview

### System Components Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                       Android Device (User A - Owner)           │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Main Activity - Widget List                             │  │
│  │  [😴 Sleeping] [🤫 Busy] [🏃 Gym]                         │  │
│  │  Each widget shows:                                      │  │
│  │  - Current state (ON/OFF, shown as color/grayscale)     │  │
│  │  - Shared user count                                    │  │
│  │  - Last toggled time                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                                                      │
│           │ [User taps widget to toggle]                       │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Floating Widget Service (Foreground Service)            │  │
│  │  - Manages WindowManager overlays for active widgets     │  │
│  │  - One service instance manages all widgets             │  │
│  │  - Listens to state changes (cache + WebSocket)        │  │
│  │  - Shows tappable emoji for accessible widgets         │  │
│  │  - User can tap to toggle (if they have permission)    │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  WebSocket Manager                                       │  │
│  │  - Single persistent connection to backend             │  │
│  │  - Sends toggle requests                              │  │
│  │  - Receives state updates                             │  │
│  │  - Auto-reconnects with exponential backoff           │  │
│  │  - Batches messages for efficiency                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Local State Cache (Room SQLite)                        │  │
│  │  - Widget configs (id, emoji, owner, shared users)    │  │
│  │  - Current state per widget (on/off)                  │  │
│  │  - Last sync timestamp                                │  │
│  │  - Pending changes queue (for offline)               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
                          │
                Real-time WebSocket
                (one connection for all)
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ User A   │    │ User B   │    │ User C   │
    │ (Owner)  │    │ (Viewer) │    │ (Viewer) │
    │ Device   │    │ Device   │    │ Device   │
    └──────────┘    └──────────┘    └──────────┘
          │               │               │
          └───────────────┼───────────────┘
                          │
                ┌─────────▼─────────┐
                │  Backend Server   │
                │  (Node.js)        │
                │  + Socket.io      │
                │  + Redis (state)  │
                │  + PostgreSQL     │
                └─────────────────┘
```

### Device-Level Flow (What Happens When User B Views Shared Widget)

```
┌────────────────────────────────────────────────────────────────┐
│                       Android Device (User B - Viewer)          │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Main Activity - Shared Widgets List                     │  │
│  │  [😴 User A's sleeping] ← User B can SEE and TOGGLE     │  │
│  │  [🤫 User A's busy]                                      │  │
│  │  [🏃 User A's gym]                                       │  │
│  │  Widgets show current state and toggle buttons           │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                                                      │
│           │ User B's phone receives state update via WebSocket │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Floating Widget Service (Interactive Overlays)          │  │
│  │  - Shows emoji overlays for SHARED widgets              │  │
│  │  - Overlays are TAPPABLE                                │  │
│  │  - If OFF, tapping turns it ON and makes User B the     │  │
│  │    active initiator.                                   │  │
│  │  - If ON by another user, tapping shows toast:         │  │
│  │    "Only User A can turn this off"                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  WebSocket Manager                                      │  │
│  │  - Receives state updates                               │  │
│  │  - Sends toggle requests                               │  │
│  │  - Stores in local cache                               │  │
│  │  - Notifies UI of changes                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Local State Cache (Room SQLite)                        │  │
│  │  - Widget configs (accessible)                          │  │
│  │  - Current state per widget                            │  │
│  │  - Last sync timestamp                                 │  │
│  │  - Pending changes queue (for offline toggles)         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
                          │
                Real-time WebSocket
                (read state updates only)
                          │
                ┌─────────▼─────────┐
                │  Backend Server   │
                │  Broadcasts state │
                │  to viewers       │
                └─────────────────┘
```

---

## Technology Stack

### Frontend (Android)
| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Kotlin | Modern, null-safe, Android-first |
| Build System | Gradle | Standard Android tooling |
| Min API Level | 26 | Foreground Services; >95% device coverage |
| Target API Level | 34 | Latest best practices |
| WebSocket | OkHttp3 | Efficient, tested, built-in support |
| State Management | LiveData + Room | Lifecycle-aware, proven patterns |
| Local Database | Room | Type-safe SQLite wrapper |
| Dependency Injection | Hilt | Reduces boilerplate, testability |
| Preferences | DataStore | Async, better than SharedPreferences |
| Network Monitoring | ConnectivityManager | Native API |

**Why NOT Flutter?**
- Floating overlays require native Android APIs (WindowManager)
- Platform channels add complexity for minimal benefit
- Single-emoji UI is overkill for Flutter's strengths
- Native Kotlin is cleaner and more performant

### Backend
| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Runtime | Node.js 18+ LTS | Async, WebSocket-native |
| WebSocket | Socket.io 4.5+ | Auto-reconnect, rooms, broadcasting |
| HTTP Server | Express 4.18+ | Lightweight, middleware ecosystem |
| Cache | Redis 7.0+ | Sub-millisecond state lookups |
| Database | PostgreSQL 14+ | ACID, audit trails, scalability |
| Authentication | JWT | Stateless, standard, scalable |

**Key decision: Why WebSocket over alternatives?**
- Real-time requirement (100-500ms latency needed)
- Push notifications would add 5-30 second latency (delay unacceptable)
- Firebase Realtime Database adds vendor lock-in
- Direct WebSocket is simplest for this use case

### DevOps & Hosting
| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Containerization | Docker | Reproducible deployments |
| Orchestration | Kubernetes (or Docker Compose for MVP) | Scalability |
| Hosting | AWS ECS / DigitalOcean / GCP Cloud Run | Managed services |
| Database | AWS RDS / DigitalOcean Managed DB | Automated backups |
| Cache | AWS ElastiCache / DigitalOcean Redis | High availability |
| Monitoring | Prometheus + Grafana | Real-time metrics |
| Error Tracking | Sentry | Production error visibility |
| CI/CD | GitHub Actions | Automated tests & deploys |

---

## Frontend Requirements (Android/Kotlin)

### 1. Project Structure Overview

The app has three main screens:

**Screen 1: My Widgets (Owner Dashboard)**
- List all widgets the current user owns
- Each widget shows:
  - Emoji
  - Name ("Sleeping", "Busy", etc.)
  - Current state (ON = colored, OFF = grayscale)
  - Who initiated the current ON state (if ON)
  - Shared with N users (count)
  - Last toggled time
- **Action:** Tapping a widget toggles its state (if permitted)
- **Optional buttons:** Create new widget, Edit widget, Delete widget, Manage shares

**Screen 2: Shared with Me (Viewer Dashboard)**
- List all widgets shared by other users
- Same layout as Screen 1, **including toggle buttons**
- **Action:** Tapping a widget toggles its state (if OFF, turns ON; if ON by someone else, toggle is disabled and shows warning toast)
- **Optional:** Shows who owns each widget

**Screen 3: Settings**
- Notification preferences
- Overlay position settings
- Logout

### 2. Permissions Required

```
SYSTEM_ALERT_WINDOW          (Display over other apps)
INTERNET                      (Network access)
FOREGROUND_SERVICE            (Keep service running)
ACCESS_NETWORK_STATE          (Monitor connectivity)
RECEIVE_BOOT_COMPLETED        (Restart on device boot)
```

### 3. Floating Widget Service (Foreground Service)

**Purpose:** Manage all emoji overlays on screen

**Behavior for Active Widgets (Regardless of Ownership):**
- Shows emoji in a configurable corner
- Emoji is tappable
- On tap: 
  - If the user was the one who turned it ON, sends toggle OFF request to backend.
  - If someone else turned it ON, tapping shows a tooltip/toast ("Only <User> can turn this off").
- Visual feedback: Color → Grayscale (or vice versa) when successfully toggled.
- Persists on screen even when app is closed
- Optional: Display tooltip on long-press showing who owns it and who currently turned it ON.

**Service Lifecycle:**
- Starts when app launches and user has at least one widget with overlay enabled
- Shows persistent notification (required by Android 12+)
- Continues running even if app is backgrounded/closed
- Restarts on device boot (via BootReceiver)
- Stops automatically when no widgets are set to "show overlay"
- Uses START_STICKY to survive system kills

**Efficiency:**
- Single service instance manages all overlays (not one per widget)
- Uses minimal CPU/GPU (no animations, just static emojis)
- Subscribes to WebSocket updates; no constant polling
- Gracefully handles network loss

### 4. WebSocket Connection Management

**Connection Behavior:**
- Establishes on app launch
- Maintains persistent connection (one for all widgets)
- Auto-reconnects with exponential backoff (1s → 2s → 5s → 10s → max 30s)
- Subscribes to rooms for all widgets user has access to (owned + shared)

**Message Types (Client → Server):**
- `subscribe_widgets`: ["widget-1", "widget-2", ...] (on app launch)
- `toggle_widget`: {widgetId: "widget-1"} (for any owned or shared widget)

**Message Types (Server → Client):**
- `state_changed`: {widgetId, newState, changedBy, timestamp}
- `error`: {message, code}

**Offline Behavior:**
- If network is unavailable, queue toggle requests locally
- On reconnection, flush queue to server
- Retry with exponential backoff if server rejects

### 5. Local Database (Room)

**Three tables:**
1. **Widgets Table**
   - id (UUID)
   - name (string)
   - emoji (string)
   - ownerId (UUID)
   - sharedUserIds (JSON array or serialized list)
   - isOverlayActive (boolean) - user preference to show overlay
   - overlayPositionX, overlayPositionY (int) - user preference
   - lastSyncAt (timestamp)

2. **Widget State Table**
   - widgetId (UUID, primary key)
   - isOn (boolean)
   - lastModifiedBy (UUID)
   - lastModifiedAt (timestamp)
   - lastSyncedAt (timestamp)

3. **Sync Queue Table** (for offline changes)
   - id (auto-increment)
   - widgetId (UUID)
   - action (string: "toggle")
   - timestamp (when queued)
   - attempts (int, for retry logic)

**Queries to optimize:**
- Get all owned widgets (fast list display)
- Get all shared widgets (fast list display)
- Get state of all widgets (for UI)
- Get pending sync items (on reconnection)

### 6. State Management Pattern

**Optimistic Updates:**
1. User taps widget
2. Immediately update local DB and UI (optimistic)
3. Send request to server asynchronously
4. If server rejects (e.g., permission denied), rollback local UI
5. If server accepts, no additional UI update needed (already updated)

**Offline Handling:**
1. User taps widget (no internet)
2. Update local DB immediately
3. Queue toggle request to SyncQueue
4. When network returns, attempt to flush queue
5. If flush fails, retry with backoff up to 3 times
6. After 3 failures, show notification: "Couldn't sync; please try again"

### 7. Notifications

**Persistent Service Notification (Android 12+):**
- Small, non-obtrusive notification in status bar
- Title: "Widget Manager"
- Text: "Monitoring X shared widgets"
- Action button: "Stop" (to disable service)

**State Change Notification (for Viewers):**
- When a shared widget's state changes, notify user
- Example: "User A is now busy (pressed 🤫)"
- Tap to dismiss or open app
- Should not wake screen; use "silent" category

### 8. Battery Efficiency (Code-level Optimization)

**Practices to follow:**
- Batch multiple operations into single network request
- Use exponential backoff for reconnections (don't retry aggressively)
- Avoid polling; rely on WebSocket push
- Close database connections properly
- Use lifecycle-aware observables (don't leak observers)
- Minimize UI refresh rate (batch updates from server)
- Use hardware acceleration wisely (no smooth animations on overlays)
- Stop service when no widgets are active

### 9. Error Handling

**Network Errors:**
- Connection refused → Show "Offline" indicator, queue changes
- Timeout → Retry with backoff
- 403 Forbidden on toggle → User lost permission, remove widget, show notification

**Permission Errors:**
- Overlay permission revoked → Show notification: "Enable 'Draw over other apps' permission"
- Storage permission → Direct to app settings

**Database Errors:**
- Corruption on startup → Wipe and re-sync from server
- Out of memory → Log and attempt graceful degradation

---

## Backend Requirements

### 1. Server Architecture

The backend is a Node.js server with Express + Socket.io.

**Key endpoints:**

**REST API** (for non-real-time operations):
- `POST /auth/register` - User registration
- `POST /auth/login` - User login, returns JWT
- `POST /auth/refresh-token` - Refresh expired token
- `GET /api/widgets` - List user's owned + shared widgets
- `POST /api/widgets` - Create new widget
- `GET /api/widgets/:id` - Get widget details
- `PUT /api/widgets/:id` - Update widget (name, emoji, shares)
- `DELETE /api/widgets/:id` - Delete widget
- `GET /api/widgets/:id/state` - Get current state (REST fallback)
- `PUT /api/widgets/:id/share` - Add/remove users from widget
- `GET /health` - Health check for monitoring

**WebSocket Events** (real-time):
- `connect` - User connects, authenticated via JWT
- `subscribe_widgets` - User subscribes to widget state updates
- `toggle_widget` - User toggles their widget (owner only)
- `disconnect` - User disconnects
- Broadcast events: `state_changed` sent to all viewers

### 2. Authorization Model

**Rules:**
- User can view (subscribe to) widgets they own OR are shared with
- User can toggle ON any widget they own or are shared with
- User can toggle OFF a widget **only if** they were the one who turned it ON
- Sharing a widget requires ownership
- Removing access from a widget requires ownership

**Implementation:**
- Every API call validates: "Does this user have permission for this action?"
- Query pattern: `SELECT * FROM widgets WHERE id = $1 AND (owner_id = $2 OR $2 = ANY(shared_user_ids))`
- WebSocket events also validate before broadcasting

### 3. State Storage

**Redis (Primary Cache):**
- Key: `widget:{widget_id}:state` → Value: "on" or "off"
- Key: `widget:{widget_id}:lastModifiedBy` → Value: user_id
- Key: `widget:{widget_id}:lastModifiedAt` → Value: timestamp
- All keys expire after 30 days (TTL)
- Sub-millisecond reads for real-time updates

**PostgreSQL (Durable Storage):**
- Tables: `users`, `widgets`, `widget_state_log` (audit)
- Widget state NOT permanently stored; only current state + audit log
- Audit log records every toggle (user, timestamp, old/new state)
- Periodic sync job (every 5 minutes) replicates Redis state to PostgreSQL for durability

### 4. WebSocket Broadcasting

**Connection Pool:**
- One Socket.io connection per client
- Client authenticated via JWT in handshake
- Socket.io manages rooms per widget (e.g., room `widget:123`)

**State Change Flow:**
1. Client sends `toggle_widget` event with widgetId
2. Server validates: 
   - Does user have access (owner or shared)?
   - If toggling OFF: Was this user the one who turned it ON?
3. Server updates Redis atomically
4. Server logs to PostgreSQL audit table (async, non-blocking)
5. Server broadcasts `state_changed` event to room `widget:123`
6. All connected clients in that room receive update
7. Clients update local cache and UI

**Broadcasting Details:**
- Use `io.to('widget:123').emit('state_changed', {...})`
- Include: widgetId, newState, changedBy, timestamp
- All viewers get the update simultaneously

### 5. Database Schema (PostgreSQL)

**Users Table:**
- id (UUID)
- email (unique)
- password_hash (bcrypt)
- name
- created_at, updated_at

**Widgets Table:**
- id (UUID)
- owner_id (FK to users)
- name (varchar 255)
- emoji (varchar 10)
- shared_user_ids (UUID array)
- created_at, updated_at

**Widget State Audit Log:**
- id (bigserial)
- widget_id (FK)
- new_state (boolean)
- changed_by (FK to users)
- changed_at (timestamp)
- (No old_state stored; just records every toggle)

**Indexes:**
- `widgets(owner_id)` - Fast lookup of user's owned widgets
- `widgets(shared_user_ids)` - GIN index for array membership
- `widget_state_log(widget_id, changed_at DESC)` - Fast audit history

### 6. Reliability & Durability

**Redis Persistence:**
- Enable AOF (Append-Only File) to survive crashes
- Replicate to PostgreSQL every 5 minutes as backup
- On Redis failure, fall back to PostgreSQL (slower, but works)

**Database Replication:**
- PostgreSQL primary + read replica in different region
- Automatic failover if primary goes down
- Backups every 6 hours, retain for 30 days

**Message Queue (Optional):**
- For audit logging, use asynchronous job queue (e.g., Bull/BullMQ)
- Don't block toggle responses on audit logging
- If queue backed up, fall back to synchronous logging

### 7. Rate Limiting & Abuse Prevention

**Rate limits:**
- Per user: 10 toggles per second (prevent spam)
- Per IP: 100 requests per 15 minutes (API abuse)
- Per user: 100 WebSocket messages per minute

**Enforcement:**
- Express middleware for REST endpoints
- Manual tracking in Socket.io for WebSocket events
- Return 429 (Too Many Requests) when limit exceeded
- Include `Retry-After` header

### 8. Monitoring & Observability

**Metrics to track:**
- Active WebSocket connections
- Toggle request latency (p50, p95, p99)
- Toggle request success rate
- Redis operation latency
- Database query latency
- Errors per second

**Logging:**
- Structured logging (JSON format)
- Log every toggle with user, widget, timestamp, success/failure
- Log all auth attempts
- Log rate limit hits

**Alerting:**
- Alert if toggle success rate drops below 99%
- Alert if WebSocket connections drop unexpectedly
- Alert if database latency exceeds 1 second
- Alert on 5xx errors

---

## Database Schema

### Detailed Entity Relationships

```
Users (1) ──── (N) Widgets (owns)
             └──── (N) Widgets (shared with)

Widgets ──── (N) Widget State Log (audit trail)

Widgets ──── (1) Widget State (current, cached in Redis)
```

**No complex relationships; simple 3-table schema.**

### Key Considerations

- `shared_user_ids` is a simple UUID array in PostgreSQL (no separate join table)
- This avoids N+1 queries when fetching user's shared widgets
- Widget state is not permanently stored in database; Redis is source of truth
- Audit log is append-only; never updated or deleted

---

## API Specifications

### Authentication

All protected endpoints require JWT in Authorization header:
```
Authorization: Bearer <JWT_TOKEN>
```

JWT includes:
- `userId` (subject)
- `email` (optional)
- `iat` (issued at)
- `exp` (expires at, 7 days from issue)

### REST Endpoints

#### User Management

**POST /auth/register**
- Public endpoint
- Body: `{ email, password, name }`
- Response: `{ token: string, user: { id, email, name } }`
- Error: 400 if validation fails, 409 if email exists

**POST /auth/login**
- Public endpoint
- Body: `{ email, password }`
- Response: `{ token: string, expiresIn: 604800 }`
- Error: 401 if credentials invalid

**POST /auth/refresh-token**
- Requires valid refresh token (in body or header)
- Response: `{ token: string }`
- Error: 403 if token invalid/expired

#### Widget Management

**GET /api/widgets**
- Returns all widgets (owned + shared with user)
- Query params: `?limit=50&offset=0&sort=-created_at` (optional)
- Response: Array of widget objects
  ```
  [{
    id: UUID,
    name: string,
    emoji: string,
    ownerId: UUID,
    sharedUserIds: [UUID],
    state: boolean (current ON/OFF),
    lastModifiedBy: UUID,
    lastModifiedAt: timestamp,
    createdAt: timestamp
  }]
  ```
- Error: 401 if not authenticated

**POST /api/widgets**
- Create new widget (authenticated user becomes owner)
- Body: `{ name: string, emoji: string, sharedUserIds?: [UUID] }`
- Response: Created widget object (same format as GET)
- Error: 400 if validation fails, 409 if name already exists

**GET /api/widgets/:id**
- Get single widget details (only if owned or shared with user)
- Response: Widget object (same format as GET /api/widgets)
- Error: 403 if not authorized, 404 if not found

**PUT /api/widgets/:id**
- Update widget (name, emoji only; owner only)
- Body: `{ name?: string, emoji?: string }`
- Response: Updated widget object
- Error: 403 if not owner, 400 if validation fails

**DELETE /api/widgets/:id**
- Delete widget (owner only)
- Response: `{ success: true }`
- Error: 403 if not owner

**PUT /api/widgets/:id/share**
- Add or remove user from shared list (owner only)
- Body: `{ action: "add" | "remove", userId: UUID }`
- Response: Updated `sharedUserIds` array
- Error: 403 if not owner, 404 if user not found

**GET /api/widgets/:id/state**
- Get current state only (REST fallback for clients without WebSocket)
- Response:
  ```
  {
    widgetId: UUID,
    state: boolean,
    lastModifiedBy: UUID,
    lastModifiedAt: timestamp
  }
  ```
- Error: 403 if not authorized

**PUT /api/widgets/:id/state**
- Update state via REST (fallback for WebSocket failure)
- Anyone with access can toggle ON. Only the initiator can toggle OFF.
- Body: `{ state: boolean }`
- Response: State object (same format as GET)
- Error: 403 if not authorized to turn off

**GET /health**
- Public endpoint
- Response: `{ status: "ok", timestamp: ISO8601 }`
- Used for load balancer health checks

### WebSocket Events

#### Client → Server

**`connect`**
- Fired automatically; client authenticated via JWT in handshake
- No explicit call needed

**`subscribe_widgets`**
- Payload: `{ widgetIds: [UUID, ...] }`
- Client subscribes to state changes for these widgets
- Server validates: User owns or has access to each widget
- Response: `{ success: true, subscribed: [UUID, ...] }`

**`toggle_widget`**
- Payload: `{ widgetId: UUID }`
- Anyone with access can toggle ON. Only the initiator can toggle OFF.
- Server updates Redis, broadcasts to room
- Response: `{ success: true, newState: boolean, timestamp: number }`
- Error: `{ success: false, error: "Not authorized to turn off" }` (403) or `"Widget not found"` (404)

**`unsubscribe_widget`**
- Payload: `{ widgetId: UUID }`
- Client leaves room for this widget
- No response expected

#### Server → Client

**`state_changed`**
- Fired when widget state changes (owner or any viewer)
- Payload:
  ```
  {
    widgetId: UUID,
    state: boolean,
    changedBy: UUID,
    timestamp: number
  }
  ```

**`error`**
- Fired on error
- Payload: `{ message: string, code: string }`
- Examples: "Not authorized", "Widget not found", "Rate limit exceeded"

**`authenticated`**
- Fired on successful connection
- Payload: `{ userId: UUID, connectedAt: timestamp }`

---

## State Management

### Client-Side State Flow

**User who initiates toggle:**
1. User taps widget in UI
2. App validates if user is allowed to toggle (always allowed if OFF; if ON, must be the initiator).
3. App immediately updates local DB (optimistic update)
4. App sends toggle request to server (async)
5. Server broadcasts state change to all users
6. App receives broadcast, confirms state matches
7. If offline, queue request and retry on reconnection

**Other users (Receiving state):**
1. Server broadcasts state change
2. App receives via WebSocket
3. App updates local cache
4. App updates UI (emoji appears or disappears)
5. If offline, app retains last known state

### Conflict Resolution

**First-to-Toggle Priority Rule:**
- If two users try to toggle ON simultaneously:
  - Server processes the first request received.
  - Server accepts the first request, making that user the initiator.
  - Server rejects the second request (403) because the widget is already ON by someone else.
  - Server broadcasts the first user's state change.

**Last-Write-Wins (in case of race condition):**
- If owner sends two toggles rapidly:
  - Server processes in order received
  - Each update overwrites previous
  - Final state is authoritative

### Offline Behavior

**Any User (Offline):**
1. Can still toggle widget locally IF allowed by current local state (optimistic update)
2. Toggle is queued in local DB
3. UI shows state immediately
4. On reconnection, queue is flushed to server
5. If server rejects (e.g. someone else turned it ON while this user was offline), app rolls back local change and shows error

**Receiving Updates (Offline):**
1. App shows last known state
2. On reconnection, app syncs latest state from server
3. If server state differs, app updates UI

### Reconciliation After Extended Offline

**Scenario:** User toggles widget multiple times while offline, then connects.
- App batches toggles into single queue flush
- Server processes in order (if valid)
- All users see final state

**Scenario:** User offline for hours, comes back online while widget is still ON.
- App fetches latest state from server
- State hasn't changed (still ON), so no UI update needed
- If app missed intermediate toggles, that's acceptable (only current state matters)

---

## Synchronization Protocol

### Initial Sync (App Launch)

1. User authenticates
2. App requests `/api/widgets` (REST)
3. Backend returns all owned + shared widgets with current states (from Redis)
4. App stores in local Room database
5. App subscribes to WebSocket and sends `subscribe_widgets` with all widget IDs
6. Server confirms subscription
7. App is now receiving real-time updates via WebSocket

**Duration:** ~500ms total (1 REST call + WebSocket subscription)

### Periodic Sync (Running App)

**Every 30 minutes** (or on app resume):
- App fetches `/api/widgets` to detect permission changes
- Example: Admin removes user B from widget
- App unsubscribes from that widget and removes from UI

**On Network Reconnection:**
- App automatically attempts WebSocket reconnection
- If successful, no additional sync needed (WebSocket will deliver missed updates)
- If failed after retries, app fetches `/api/widgets` to catch up

### Offline Sync (On Network Return)

**User with pending toggles:**
1. Network becomes available
2. App flushes SyncQueue to server
3. Server processes each toggle (validating initiator-only-off rules)
4. Server broadcasts final state to all users
5. App receives broadcast, confirms sync

**User with stale cache (no pending toggles):**
1. Network becomes available
2. App optionally fetches `/api/widgets` to check for permission changes
3. App re-subscribes to WebSocket
4. Receives state updates normally

### Server Durability Sync

**Every 5 minutes:**
- Background job replicates Redis state to PostgreSQL
- Ensures data survives Redis crash
- Not visible to clients; transparent operation

---

## Error Handling & Edge Cases

### 1. Network Errors

**Case: Connection Refused**
- User is offline
- App queues toggle locally
- Shows "Offline" indicator
- On reconnection, flushes queue

**Case: Timeout on Toggle**
- Request takes >5 seconds
- App retries with exponential backoff
- After 3 failures, shows notification: "Couldn't toggle; check your connection"
- User can manually retry

**Case: Server Returns 429 (Rate Limit)**
- User is toggling too fast
- App respects `Retry-After` header
- Queues remaining toggles
- Resumes after delay

### 2. Permission Errors

**Case: User Loses Access to Widget**
- Admin removes user B from widget
- User B's next toggle returns 403
- App removes widget from local DB
- App unsubscribes from WebSocket room
- Shows notification: "You no longer have access to this widget"

**Case: Owner Deleted Widget**
- Widget owner deletes widget
- Server broadcasts deletion notification to shared users
- App receives notification, removes from UI
- Users see notification: "This widget has been deleted"

### 3. Data Consistency Issues

**Case: Local Cache Stale**
- Network was down for extended period
- While offline, a user toggled the widget
- Another user comes back online, cache shows wrong state
- Solution: On network return, app fetches `/api/widgets` or waits for WebSocket update

**Case: Redis Crash**
- Primary state storage is lost
- Server falls back to PostgreSQL (5-minute-old state)
- Clients briefly see stale state
- Auto-recovery: Redis restarted, state replicated back from PostgreSQL
- No data loss; just brief latency

### 4. UI Edge Cases

**Case: User Toggles While App Transitioning States**
- User taps widget, toggles to ON
- Immediately toggles to OFF before state change received from server
- App queues both requests
- Server processes in order: result is OFF
- UI shows OFF (correct)

**Case: Overlay Active, But App Force-Closed**
- Service uses START_STICKY, will restart
- BootReceiver ensures it starts on device reboot
- User loses nothing; overlay resumes when service restarts

**Case: Overlay Permission Revoked**
- User disables "Draw over other apps" in settings
- Service attempts to show overlay, gets SecurityException
- Service catches error, disables overlay
- Shows notification: "Overlay permission required; enable in Settings"
- User can re-enable permission and overlay resumes

### 5. Offline Sync Edge Cases

**Case: Offline Queue Has 100 Toggles**
- User toggled widget repeatedly while offline
- On reconnection, don't send 100 individual messages
- Batch into single request (or final state, if valid)
- Server processes, broadcasts once

**Case: Offline for Days**
- Sync queue exceeds local storage
- App deletes oldest entries, keeps newest
- May lose some history, but final state is preserved
- On sync, app reconciles with server

### 6. Multi-Device Scenarios

**Case: User Uses Two Devices**
- Device A toggles widget to ON
- Device B is still showing OFF (cache stale)
- Solution: Device B receives WebSocket update within 500ms
- Both devices sync to same state

**Case: User Sees Multiple Copies of Same Widget**
- Shouldn't happen; widget ID is unique
- App uses unique ID for deduplication
- Impossible edge case

---

## Security Requirements

### 1. Authentication

**JWT-Based:**
- User registers/logs in with email + password
- Server validates, hashes password (bcrypt)
- Issues JWT with 7-day expiration
- Client stores JWT in encrypted SharedPreferences (Android)
- Client sends JWT with every request/WebSocket connection

**Refresh Token (Optional):**
- Separate refresh token with longer expiration (30 days)
- Client can refresh access token without re-logging in
- Reduces password exposure

### 2. Authorization

**Per-Request Validation:**
- Toggle: `SELECT * FROM widgets WHERE id = $1 AND owner_id = $2` (must match)
- View: `SELECT * FROM widgets WHERE id = $1 AND ($2 = ANY(shared_user_ids) OR owner_id = $2)` (must match)
- Share: Only owner can modify `shared_user_ids`

**WebSocket Validation:**
- JWT verified on connection handshake
- `subscribe_widgets` event validates user has access to all requested widgets
- `toggle_widget` event validates user owns the widget

### 3. Data Protection

**In Transit:**
- All endpoints use HTTPS only
- WebSocket uses WSS (secure WebSocket)
- Set HSTS header

**At Rest:**
- Passwords hashed with bcrypt (salt + iterations)
- Sensitive data in PostgreSQL encrypted (optional, via pgcrypto extension)
- Local Android cache: use EncryptedSharedPreferences

**Database Access:**
- Principle of least privilege: app DB user has only SELECT/INSERT/UPDATE (no DROP)
- Connection pooling with connection timeouts
- SQL parameterized queries (prevent injection)

### 4. Input Validation

**Server-side (mandatory):**
- Widget name: max 255 characters, alphanumeric + spaces
- Emoji: validate it's a valid Unicode emoji, max 10 bytes
- User ID: validate UUID format
- Payload size: max 10KB

**Client-side (convenience):**
- Validate before sending
- Show error message if invalid
- Don't send malformed requests

### 5. Rate Limiting

**REST API:**
- 100 requests per IP per 15 minutes
- 10 toggles per user per second

**WebSocket:**
- 100 messages per user per minute
- 10 toggles per user per second

**Auth endpoints:**
- 5 login attempts per IP per 15 minutes
- 3 registrations per IP per hour

### 6. Audit Logging

**Every toggle is logged:**
- User ID
- Widget ID
- Old state → New state
- Timestamp
- Client IP address
- Success/failure

**Retention:** 90 days in PostgreSQL

### 7. CORS Configuration

**Allowed origins:**
- For native Android: Not strictly needed (no web requests)
- For web dashboard (if added later): Whitelist specific domain

### 8. Secrets Management

**Never commit:**
- JWT secret
- Database password
- Redis password
- API keys

**Use environment variables or secrets manager:**
- AWS Secrets Manager
- Google Cloud Secret Manager
- HashiCorp Vault
- Or simple `.env` file (not committed)

---

## Testing Strategy

### Unit Tests

**Backend:**
- JWT validation
- Authorization checks
- Input validation
- Database queries
- Redis operations
- WebSocket message parsing

**Android:**
- State management logic
- Database queries
- Offline queue handling
- Permission checks
- UI state updates

### Integration Tests

**Backend:**
- REST API endpoints
- WebSocket connections
- Database operations
- Redis fallback (on failure)
- Auth flow (register → login → authenticated request)

**Android:**
- WebSocket connection + state update flow
- Network loss + recovery
- Offline toggle + sync
- Local database → UI sync

### End-to-End Tests

- User toggles widget ON; others see update within 500ms
- User offline, comes back online; sees latest state
- User loses access; widget removed from their UI
- Two users toggle ON same widget simultaneously; first request accepted, second rejected

### Performance Tests

- Toggle latency under normal conditions: <200ms
- Toggle latency under load (1000 concurrent users): <500ms
- WebSocket message delivery: <100ms average

---

## Deployment & DevOps

### Backend Deployment

**Option 1: Docker (Recommended)**
- Containerize Node.js app
- Push to Docker registry
- Deploy to Kubernetes (or Docker Swarm, or managed services)
- Auto-scale based on WebSocket connections

**Option 2: Managed Services**
- AWS Lambda + API Gateway + RDS
- Google Cloud Run + Cloud SQL
- Heroku (simple but less control)

**Database & Cache:**
- PostgreSQL: Managed service (AWS RDS, DigitalOcean, GCP Cloud SQL)
- Redis: Managed service (AWS ElastiCache, DigitalOcean, GCP Memorystore)

**Monitoring:**
- Prometheus for metrics collection
- Grafana for dashboards
- Sentry for error tracking
- CloudWatch (if using AWS)

**CI/CD:**
- GitHub Actions for automated testing + deployment
- Test on every push
- Deploy to staging on PR
- Deploy to production on main branch merge

### Android Deployment

**Release Process:**
1. Update version code in `build.gradle`
2. Build signed APK/AAB
3. Upload to Google Play Console
4. Set rollout percentage (5% → 25% → 100%)
5. Monitor crash reports in Play Console
6. If issues, rollback to previous version

**Testing Before Release:**
- Internal testing (private track)
- Closed beta (1000 devices)
- Open beta (all users, limited release)
- Full release

---

## Performance Targets

### Client-Side (Android)

| Metric | Target |
|--------|--------|
| App startup | <2 seconds |
| Toggle action response (UI feedback) | <200ms |
| State update latency (optimistic) | <100ms |
| WebSocket state update reception | <500ms |
| Memory footprint | <50MB |
| Database query latency | <10ms |
| Battery drain (active use) | <5% per hour |

### Server-Side

| Metric | Target |
|--------|--------|
| Toggle request processing | <100ms |
| WebSocket broadcast latency | <100ms |
| Redis operation | <5ms |
| Database query | <50ms |
| Concurrent WebSocket connections | 10,000+ |
| Toggle requests per second | 1,000+ |
| Error rate | <0.1% |
| Availability | 99.9% |

### Network

| Metric | Target |
|--------|--------|
| Message size | <1KB |
| WebSocket connection time | <2 seconds |
| Reconnection time | <5 seconds |

---

## Summary of Key Decisions

1. **Mutually-controlled toggles (Initiator-Only-Off)**: Anyone with access can turn a widget ON, but only the person who turned it ON can turn it OFF.
2. **WebSocket for real-time**: Push-based updates ensure <500ms latency
3. **Optimistic updates**: UI responds immediately; server confirms later
4. **Offline queue**: Users can toggle offline; changes sync when network returns
5. **Floating overlays**: Persistent visual indicator, interactive for the initiator
6. **Single service instance**: All widgets managed by one foreground service for efficiency
7. **Redis + PostgreSQL**: Fast reads (Redis) + durable storage (PostgreSQL)
8. **JWT authentication**: Stateless, scalable, standard
9. **Permission model**: Simple access validation + initiator-lock for the OFF action
10. **Minimal battery drain**: WebSocket + efficient service lifecycle

---

## Appendix: Technology Choices Explained

### Why Native Android, Not Flutter?

- Floating overlays require WindowManager API; Flutter needs platform channels
- Single-emoji UI doesn't justify Flutter's overhead
- Native Kotlin is more performant for this use case
- Platform channels add complexity

### Why WebSocket, Not Push Notifications?

- Push notifications have 5-30 second latency (Firebase Cloud Messaging)
- WebSocket guarantees <500ms delivery
- Real-time status indicator needs immediate feedback
- WebSocket is bidirectional (can also receive from server)

### Why Redis + PostgreSQL, Not Just PostgreSQL?

- Redis: Sub-millisecond state reads (needed for real-time broadcasts)
- PostgreSQL: Durable storage (survives crashes)
- Combined: Best of both worlds
- PostgreSQL alone would have 10-50ms latency per toggle

### Why Express + Socket.io, Not Just Socket.io?

- Express provides HTTP REST API (for clients without WebSocket)
- Socket.io provides WebSocket with fallbacks
- Together: Robust, battle-tested, production-ready

### Why JWT, Not Session-Based Auth?

- JWT is stateless (no server session storage needed)
- Scales to multiple servers
- Standard, widely supported
- No session table in database

---

## Final Notes

**The core value of this app is speed:**
- User A turns on status indicator
- User B sees floating emoji within 500ms
- This enables immediate "do not disturb" communication

**All technical decisions support this goal:**
- WebSocket for real-time delivery
- Redis for sub-millisecond state access
- Foreground service for persistent visibility
- Optimistic updates for instant UI feedback

Keep it simple. Avoid feature creep (no animations, no complex permissions, no multiple states). Focus on reliability and speed.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.0 | 2026-05-07 | Revised: Owner-controlled toggles, viewers read-only; removed code examples; clarified real-time status use case |
| 2.1 | 2026-05-07 | Revised: Mutually-controlled toggles with initiator-only-off rule |
| 1.0 | 2026-05-06 | Initial version |

---

## Sign-Off

This document defines requirements for a real-time status notification app with mutually-controlled toggles (initiator-only-off).

**Date:** May 7, 2026
