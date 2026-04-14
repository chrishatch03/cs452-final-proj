# New User Login Flow — Network Sequence

Complete network flow from the moment a brand new user clicks "Get Started" on the landing page to seeing their completed epic.

---

```
BROWSER                          KINARRATIVE BACKEND                    FAMILYSEARCH / EXTERNAL
  |                                     |                                       |
  |  1. GET /auth/login?app=web         |                                       |
  | ----------------------------------> |                                       |
  |  <-- 307 Redirect to auth.fhtl.org  |                                       |
  |                                     |                                       |
  |  2. GET https://auth.fhtl.org?redirect=...                                  |
  | ----------------------------------------------------------------------->    |
  |  (User authenticates with FamilySearch)                                     |
  |  <-- Redirect to /auth/callback?fstoken=...&app=web                         |
  |                                     |                                       |
  |  3. GET /auth/callback?fstoken=...  |                                       |
  | ----------------------------------> |                                       |
  |         Backend:                    |                                       |
  |         - Decode fstoken JWT        |                                       |
  |         - Create User row           |                                       |
  |         - Create ExternalIdentityLink                                       |
  |         - Store encrypted FS credential                                     |
  |         - Create AuthSession (8hr expiry)                                   |
  |         - Fire async: launch_initialization_pipeline()                      |
  |  <-- 307 Redirect to /             |                                       |
  |      Set-Cookie: kn_session=...    |                                       |
  |                                     |                                       |
  |  4. GET /                           |                                       |
  | ----------------------------------> |  (Vite serves React app)              |
  |  <-- HTML + JS bundle               |                                       |
  |                                     |                                       |
  |  5. GET /auth/session               |                                       |
  | ----------------------------------> |                                       |
  |  <-- { authenticated: true,         |                                       |
  |        userId: "KWVX-YQJ",          |                                       |
  |        sessionId: "...",             |                                       |
  |        displayName: "Chris" }        |                                       |
  |                                     |                                       |
  |  --- Three parallel requests ---    |                                       |
  |                                     |                                       |
  |  6a. GET /v1/app/users/{id}/bootstrap/shell                                |
  | ----------------------------------> |                                       |
  |  <-- { user, welcomeBootstrap:      |                                       |
  |        { isFirstLogin: true,        |                                       |
  |          inProgress: true },        |                                       |
  |        defaultEpic, quickStarts }   |                                       |
  |                                     |                                       |
  |  6b. GET /v1/app/users/{id}/bootstrap                                      |
  | ----------------------------------> |                                       |
  |  <-- { user, overview, chats,       |                                       |
  |        primaryArtifact,             |                                       |
  |        welcomeBootstrap }           |                                       |
  |                                     |                                       |
  |  FRONTEND: Renders AppShell with    |                                       |
  |  synthetic data from shell (6a),    |                                       |
  |  hydrates with full data (6b)       |                                       |
  |                                     |                                       |
  |  7. SSE: GET /v1/app/users/{id}/initialization-stream                      |
  | ----------------------------------> |                                       |
  |  <-- event: generation_discovered   |                                       |
  |      data: { totalAncestors: 200 }  |                                       |
  |                                     |                                       |
  |                                     |  8. PIPELINE (async background):      |
  |                                     |  GET /platform/tree/ancestry          |
  |                                     |      ?person=KWVX-YQJ&generations=8  |
  |                                     | ------------------------------------> |
  |                                     |  <-- 8 generations of ancestry data   |
  |                                     |                                       |
  |  <-- event: generation_processed    |  9. For each generation (0-8):        |
  |      data: { generation: 0,         |     - Parse persons                   |
  |        label: "You",                |     - Resolve places                  |
  |        ancestorCount: 19,           |     - Match historical events         |
  |        geographies: [...],          |     - Emit SSE event                  |
  |        movements: [...],            |                                       |
  |        events: [...] }              |                                       |
  |                                     |                                       |
  |  <-- event: generation_processed    |                                       |
  |      data: { generation: 1,         |                                       |
  |        label: "Parents", ... }      |                                       |
  |                                     |                                       |
  |  <-- event: generation_processed    |                                       |
  |      data: { generation: 2,         |                                       |
  |        label: "Grandparents", ... } |                                       |
  |                                     |                                       |
  |  ... (generations 3-8) ...          |                                       |
  |                                     |                                       |
  |  <-- event: epic_building           |  10. Build epic from discovered data  |
  |      data: {}                       |      - Algorithmic structure           |
  |                                     |      - LLM narrative generation        |
  |                                     |      - Store EpicRecord + overview     |
  |                                     |                                       |
  |  <-- event: epic_ready              |  11. Set user.onboarding_status =     |
  |      data: { epicId: "...",         |      "ready"                          |
  |        title: "Your Family Epic" }  |                                       |
  |                                     |                                       |
  |  <-- event: done                    |                                       |
  |      data: { status: "ready" }      |                                       |
  |                                     |                                       |
  |  FRONTEND: InitializingPlaceholder  |  12. BACKGROUND ENRICHMENT (async):   |
  |  closes, welcome overlay plays      |      - BFS tree expansion             |
  |                                     |      - Detail fetching per person     |
  |                                     |      - Place geocoding (18k+ places)  |
  |  13. POST /v1/app/users/{id}/       |      - Full epic rebuild              |
  |         welcome/complete            |                                       |
  | ----------------------------------> |                                       |
  |  <-- { completed: true }            |                                       |
  |                                     |                                       |
  |  USER SEES: Interactive globe with  |                                       |
  |  ancestor locations, migration      |                                       |
  |  routes, chat interface ready       |                                       |
  |                                     |                                       |
```

---

## Summary of Requests

| # | Method | URL | Initiated By | Purpose |
|---|--------|-----|-------------|---------|
| 1 | GET | `/auth/login?app=web` | Browser click | Start OAuth flow |
| 2 | GET | `https://auth.fhtl.org?redirect=...` | 307 redirect | FamilySearch authentication |
| 3 | GET | `/auth/callback?fstoken=...` | FS redirect | Create user, session, launch pipeline |
| 4 | GET | `/` | 307 redirect | Load React app |
| 5 | GET | `/auth/session` | Frontend JS | Validate session, get userId |
| 6a | GET | `/v1/app/users/{id}/bootstrap/shell` | Frontend JS | Fast shell data for immediate render |
| 6b | GET | `/v1/app/users/{id}/bootstrap` | Frontend JS | Full bootstrap with epic + chats |
| 7 | GET | `/v1/app/users/{id}/initialization-stream` | Frontend JS (SSE) | Real-time pipeline progress |
| 8 | GET | `/platform/tree/ancestry?person=...&generations=8` | Backend async | Fetch ancestry from FamilySearch |
| 9 | — | SSE events (generation_processed x9) | Backend → Frontend | Stream each generation's data |
| 10 | — | SSE event (epic_ready) | Backend → Frontend | Signal epic is built |
| 11 | — | SSE event (done) | Backend → Frontend | Pipeline complete |
| 12 | — | Background tasks | Backend async | BFS expansion, detail fetch, geocoding |
| 13 | POST | `/v1/app/users/{id}/welcome/complete` | Frontend JS | Acknowledge onboarding finished |

## Key Technical Details

- **Session**: `kn_session` httpOnly cookie with UUID, 8-hour expiry
- **Credentials**: FamilySearch access token encrypted with Fernet symmetric cipher
- **Progressive rendering**: Shell bootstrap renders the app frame in ~100ms, full data hydrates after
- **SSE**: Server-Sent Events via in-memory ProgressChannel, 15-second heartbeat, catch-up for late connections
- **Pipeline**: Fire-and-forget async task, frontend follows independently via SSE
- **Background enrichment**: Continues after user sees their epic — BFS discovers 16,000+ extended family members over minutes
