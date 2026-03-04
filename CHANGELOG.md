# Changelog

All notable changes to the AT-IBA-PA MINIMART POS system will be documented in this file.

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
