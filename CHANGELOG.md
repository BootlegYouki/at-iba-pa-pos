# Changelog

All notable changes to the AT-IBA-PA MINIMART POS system will be documented in this file.

---

## [1.3.1] - 2026-03-23

### Changed

**Sync Integrity and Stock Consistency**
- Cashier and Admin can both push sales to Supabase when appropriate, but the routing stays role-aware:
  - Cashier pushes transactions and inventory logs directly only when LAN is not active
  - Admin remains the single upstream source for LAN-delivered sales
- Cloud stock is now derived from `inventory_logs` through a Supabase SQL trigger instead of relying on product stock pushes from sale sync
- Users and shared settings now respect `updated_at` freshness when pulling from cloud, preventing older cloud rows from overwriting newer local edits
- Frontend Supabase client lifecycle was hardened so restart, connect, disconnect, and reconnect flows rebind listeners correctly

### Fixed

**LAN + Cloud Mixed-Mode Duplication**
- LAN-delivered sales now preserve their original inventory log identifiers all the way to the Admin and cloud sync layer
- This keeps mixed LAN/cloud retries idempotent and prevents duplicate inventory log creation for the same sale
- Stock in Supabase is now reduced exactly once per accepted inventory log instead of drifting through duplicate sale propagation

**Immediate LAN Stock Refresh**
- When the Admin accepts a cashier sale over LAN, it now emits a fresh database-change event immediately
- Other connected cashier terminals can refresh stock sooner instead of waiting for an unrelated later sync

### Documentation
- Updated [Database & Data Layer](docs/03-database.md) to describe inventory logs as the cloud stock event source
- Updated [Local Network Sync](docs/04-networking.md) with the full LAN sale payload and immediate stock refresh behavior
- Updated [Cloud Sync](docs/05-cloud-sync.md) with trigger-based stock synchronization, idempotent sale flow, and freshness safeguards
- Updated [User Guide](docs/10-user-guide.md) with the current cashier/Admin cloud behavior for LAN and non-LAN setups

---

## [1.3.0] — 2026-03-19

### ✨ Added

**AI Agent — Full Overhaul**
- Multi-provider support: choose between **Groq**, **Mistral AI**, or **Local Ollama**
  - Groq (default): `llama-3.3-70b-versatile` and all active Groq chat models
  - Mistral: curated model list (`mistral-large`, `mistral-small`, `pixtral-large`, `codestral`, etc.)
  - Local: Ollama running at `localhost:11434` — no API key, fully offline
- **Persistent conversation history** stored in SQLite (`ai_conversations`, `ai_messages` tables)
  - ConversationHistoryPanel — browse, load, and delete past chats
  - Auto-generated conversation titles after the 3rd assistant reply (AI-generated, max 6 words)
- **File attachments** — attach up to 5 files (300 KB each) per message; text files are injected into the prompt, binary files are referenced by name
  - Attach via file picker button or paste from clipboard
- **Fullscreen mode** — toggle sidebar to expand and fill the full dashboard width
- **Auto-retry** — automatically retries once on empty-response errors (900ms delay)
- **Response regeneration** — regenerate the last assistant reply on demand
- AI config (provider, API keys, model) stored encrypted in the `settings` table

**Export Reports — New Advanced Configuration UI**
- New dedicated export screen (`/reports/export`) replacing simple CSV/XLSX buttons
- **Workbook Settings**: global date range picker + layout choice (Separate Sheets / Combined Sheet)
- **Section management**: enable/disable and drag-to-reorder 5 report sections (Sales Summary, Transactions, Top Products, Inventory, Hourly Sales)
- **Per-section config**: date range override, sort metric/order, cashier filter, category filter, text search, stock filter, visible column selection
- Settings auto-saved to `localStorage` and restored on next visit
- Responsive layout (stacked below 1100px, wide above)

**Customer Window Toggle (Cashier Settings)**
- Added **Display** section in Cashier Settings dialog with a toggle to enable or disable the Customer Window (second monitor display)
- Preference persisted to `localStorage`; opens/closes the Tauri customer display window accordingly

### 📝 Documentation
- Rewrote [AI Analytics](docs/08-ai-analytics.md): multi-provider architecture, persistent conversations, file attachments, fullscreen mode, auto-retry, security model
- Added [Export Reports](docs/12-export-reports.md): full documentation of the new export configuration screen
- Updated [User Guide](docs/10-user-guide.md): updated AI Agent section to cover new provider setup, attachments, fullscreen, and conversation history

---

## [1.2.0] — 2025-07-17


### ✨ Added

**Cloud Credential Auto-Propagation via LAN**
- Admin automatically broadcasts Supabase credentials (`CloudCredentials { url, key }`) to all connected cashiers when:
  - A cashier connects and sends `InitialSyncRequest`
  - Admin updates cloud config in Settings
- Cashier stores received credentials encrypted (AES-256-GCM) in Tauri Store
- Cashier emits `cloud-credentials-received` event for frontend to pick up and start cloud sync
- Eliminates manual cloud configuration on every cashier terminal

**Cloud Disconnect Propagation**
- Added `CloudCredentialsCleared` LAN message variant
- When Admin clears cloud configuration, all connected cashiers:
  - Clear stored encrypted credentials
  - Abort active cloud sync tasks
  - Emit `cloud-credentials-cleared` event to frontend

### 🛠 Changed

**Manual LAN Disconnect Behavior**
- Manual disconnect now triggers a LAN scan to show available Admin servers
- Added `manuallyDisconnectedRef` guard to prevent auto-reconnect after manual disconnect
- Auto-connect resumes only when user explicitly reconnects or app restarts

### 🐛 Fixed

**Disconnect Button Not Working**
- Fixed `disconnect_admin` Tauri command that was immediately restarting `run_local_client_loop_with_discovery` after clearing connection state, causing reconnection within seconds
- Removed the auto-discovery restart from `disconnect_admin` — disconnect now fully disconnects

### 📝 Documentation
- Updated [Networking](docs/04-networking.md): added `CloudCredentials` and `CloudCredentialsCleared` to message protocol, new Scenario D (cloud credential propagation), updated reconnection table with manual disconnect and cloud credential behaviors
- Updated [Cloud Sync](docs/05-cloud-sync.md): added LAN credential propagation section, updated runtime commands with LAN broadcast behavior
- Updated [Security](docs/09-security.md): added AES-256-GCM encrypted cloud credential storage, updated threat model
- Updated [User Guide](docs/10-user-guide.md): added cloud config auto-propagation to setup steps, manual disconnect behavior, cloud sync troubleshooting

---

## [1.1.0] — 2026-03-05

### 🛠 Changed

**Runtime Cloud Configuration**
- Removed compile-time `.env` / `env!()` Supabase credential embedding
- Supabase URL and Anon Key are now configured at runtime via Settings UI
- Credentials stored in Tauri Store (`.settings.dat`) — persists across restarts
- App starts in offline mode when no credentials are set
- Added `update_cloud_config`, `clear_cloud_config` Tauri commands for runtime credential management
- Sync loop dynamically starts/stops when credentials are added/removed
- Added auto-provisioning flow: app detects missing tables and shows copyable SQL migration

**Sync Engine Improvements**
- Paginated cloud fetch (1,000 records per page) to avoid PostgREST timeouts
- Flat queries instead of embedded resources for faster pulls
- Batched push with LIMIT 50 per cycle
- Bulk upserts wrapped in single SQL transactions
- Smart conflict avoidance: cloud data never overwrites local pending changes
- LAN-aware sync: Cashier skips cloud push when LAN-connected to Admin
- Selective entity sync (`entity` parameter) for targeted sync cycles

**Performance Optimizations**
- Batch write operations via `db_execute_batch` (single IPC round-trip per sale)
- Single JOIN queries replacing N+1 patterns for transaction + items loading
- Server-side pagination with LIMIT/OFFSET and SQL-level aggregation
- Lazy-loaded transaction line items (fetched on click, not on page load)
- Debounced sync-driven UI refreshes (800ms)
- Shallow equality checks to prevent unnecessary React re-renders
- VACUUM after factory reset for accurate DB size reporting

### 📝 Documentation
- Added [Performance Optimizations](docs/11-performance.md) document
- Updated [Cloud Sync](docs/05-cloud-sync.md) with runtime credential configuration, paginated pulls, batched pushes, and LAN-aware sync
- Updated [Security](docs/09-security.md) to reflect runtime Supabase key storage

---

## [1.0.0] — 2026-03-03

### 🚀 Initial Release

**Admin IMS (Inventory Management System)**
- Dashboard with real-time sales summary, revenue charts, and low-stock alerts
- Full inventory management (add, edit, delete products with barcode lookup)
- Transaction history with search, filtering, pagination, and detail view
- Reports page with sales trends, peak hours, payment breakdown, top products
- CSV/XLSX export and printable Z-Report
- AI-powered analytics sidebar (Groq)
- System settings (store name, theme, user management)
- LAN WebSocket server for local cashier sync
- UDP auto-discovery beacon for cashier auto-connect
- Background cloud sync to Supabase

**Cashier POS (Point of Sale)**
- Barcode scanning with global keystroke hook (focus-free)
- Manual product selection via category grid
- Multiplier syntax support (e.g., `3*4800016641234`)
- Inline payment flow (cash & GCash) with quick-tender buttons
- Customer-facing display window (second screen)
- QR code digital receipts
- Keyboard shortcuts (F4 Cancel, F8 Pay, Space focus scanner)
- LAN auto-discovery and sync to Admin
- Background cloud sync to Supabase
- Offline-first operation — works without internet

**Shared Infrastructure**
- Local SQLite database with WAL mode
- Supabase cloud backend with Postgres triggers
- Three-tier sync: Online (cloud) → Local (LAN) → Offline
- PIN-based cashier authentication
- Dark/light mode with OS detection
