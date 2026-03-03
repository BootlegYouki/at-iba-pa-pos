# Changelog

All notable changes to the AT-IBA-PA MINIMART POS system will be documented in this file.

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
