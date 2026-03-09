<p align="center">
  <img src="assets/logo/logo-placeholder.png" alt="AT-IBA-PA MINIMART" width="120" />
</p>

<h1 align="center">AT-IBA-PA MINIMART</h1>
<h3 align="center">Point of Sale & Inventory Management System</h3>

<p align="center">
  A high-performance, offline-first desktop POS system with barcode scanning, real-time inventory tracking, LAN sync, cloud backup, AI-powered analytics, and digital QR receipts.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Tauri-24C8DB?style=for-the-badge&logo=tauri&logoColor=white" alt="Tauri">
  <img src="https://img.shields.io/badge/Rust-ce412b?style=for-the-badge&logo=rust&logoColor=white" alt="Rust">
  <img src="https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB" alt="React">
  <img src="https://img.shields.io/badge/TypeScript-007ACC?style=for-the-badge&logo=typescript&logoColor=white" alt="TypeScript">
  <img src="https://img.shields.io/badge/Vite-646CFF?style=for-the-badge&logo=vite&logoColor=white" alt="Vite">
  <img src="https://img.shields.io/badge/Tailwind_CSS-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white" alt="Tailwind CSS">
  <br>
  <img src="https://img.shields.io/badge/SQLite-07405E?style=for-the-badge&logo=sqlite&logoColor=white" alt="SQLite">
  <img src="https://img.shields.io/badge/Supabase-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white" alt="Supabase">
  <img src="https://img.shields.io/badge/shadcn/ui-000000?style=for-the-badge&logo=shadcnui&logoColor=white" alt="shadcn/ui">
  <img src="https://img.shields.io/badge/Groq-000?style=for-the-badge&logo=groq&logoColor=white" alt="Groq">
</p>

<p align="center">
  <a href="#-features">Features</a> •
  <a href="#-download">Download</a> •
  <a href="#-installation">Installation</a> •
  <a href="#-documentation">Documentation</a> •
  <a href="#-system-requirements">System Requirements</a>
</p>

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| **Barcode Scanning** | Instant product lookup via USB barcode scanner — no focus required |
| **Sales Processing** | Fast checkout with Cash, Card, GCash & Maya payments, keyboard shortcuts for speed |
| **Inventory Management** | Add, edit, delete products with low-stock alerts and category filtering |
| **Reports & Analytics** | Daily/weekly/monthly sales reports, charts, CSV/XLSX export, printable Z-Reports |
| **AI Analytics** | Groq-powered sales trend analysis and stock predictions (Admin only) |
| **Customer Display** | Second-screen support showing live cart and QR digital receipts |
| **Local Network Sync** | Admin and Cashier sync over LAN when internet is unavailable |
| **Cloud Backup** | Automatic background sync to Supabase when online |
| **Dark Mode** | System-aware theme with manual toggle |
| **Offline-First** | Fully operational without internet — zero downtime |

---

## 📥 Download

> Download the latest installers from the [**Releases**](https://github.com/BootlegYouki/at-iba-pa-pos/releases) page.

| App | Description | Download |
|-----|-------------|----------|
| **Admin IMS** | Inventory management, reports, analytics, settings | [Latest Release](https://github.com/BootlegYouki/at-iba-pa-pos/releases) |
| **Cashier POS** | Point-of-sale checkout terminal | [Latest Release](https://github.com/BootlegYouki/at-iba-pa-pos/releases) |

Both apps are built as standalone Windows `.exe` installers.

---

## 🚀 Installation

1. Download the **Admin IMS** and/or **Cashier POS** installer from [Releases](https://github.com/BootlegYouki/at-iba-pa-pos/releases)
2. Run the `.exe` installer and follow the setup wizard
3. Launch the app — on first run, sample data is automatically loaded
4. For multi-terminal setups, install **Admin IMS** on the manager's PC and **Cashier POS** on each checkout terminal

> **Networking:** Both apps must be on the same local network (LAN) for local sync. See the [Networking Guide](docs/04-networking.md) for firewall setup.

---

## 📸 Screenshots

<details>
<summary>Click to expand screenshots</summary>

### Admin System
<img src="assets/screenshots/admin/admin%20dashboard.png" width="400" alt="Admin Dashboard">
<img src="assets/screenshots/admin/inventory%20page.png" width="400" alt="Inventory Page">
<img src="assets/screenshots/admin/admin%20reports%20page.png" width="400" alt="Reports Page">
<img src="assets/screenshots/admin/admin%20transactions%20page.png" width="400" alt="Transactions Page">

### Cashier POS
<img src="assets/screenshots/cashier/cashier%20pos.png" width="400" alt="Cashier POS">
<img src="assets/screenshots/cashier/cashier%20cart.png" width="400" alt="Cashier Cart">

### Customer Display & Receipt
<img src="assets/screenshots/cashier/customer%20screen%20idle.png" width="400" alt="Customer Display - Idle">
<img src="assets/screenshots/cashier/customer%20screen%20cart.png" width="400" alt="Customer Display - Cart">
<img src="assets/screenshots/cashier/customer%20screen%20qr%20receipt.png" width="400" alt="Customer Display - QR">
<img src="assets/screenshots/cashier/e-receipt%20website%20desktop.png" width="400" alt="E-Receipt Desktop">

</details>

---

## 📚 Documentation

| # | Document | Description |
|---|----------|-------------|
| 1 | [Product Overview](docs/01-overview.md) | Background, target users, features, scope |
| 2 | [System Architecture](docs/02-architecture.md) | Tech stack, dual-app design, data flow |
| 3 | [Database & Data Layer](docs/03-database.md) | SQLite schema, DAL pattern, stock algorithm |
| 4 | [Local Network Sync](docs/04-networking.md) | LAN WebSocket sync, auto-discovery protocol |
| 5 | [Cloud Sync](docs/05-cloud-sync.md) | Supabase sync algorithm, conflict resolution |
| 6 | [Barcode Scanner](docs/06-barcode-scanner.md) | Global keystroke hook, timing heuristics |
| 7 | [Customer Display & Receipts](docs/07-customer-display.md) | Multi-window, QR receipts, receipt website |
| 8 | [AI Analytics](docs/08-ai-analytics.md) | Groq integration, use cases, constraints |
| 9 | [Security](docs/09-security.md) | Authentication, encryption, RLS |
| 10 | [User Guide](docs/10-user-guide.md) | Installation, cashier guide, admin guide |
| 11 | [Performance](docs/11-performance.md) | Database tuning, indexes, batch ops, pagination |
| 12 | [Database Schema](docs/database_schema.md) | Full SQLite structure, tables, and fields |

---

## 💻 System Requirements

| Requirement | Minimum |
|-------------|---------|
| **OS** | Windows 10 (64-bit) or later |
| **RAM** | 2 GB |
| **Storage** | 200 MB |
| **Display** | 1280 × 720 |
| **Network** | LAN (for multi-terminal sync) |
| **Internet** | Optional (for cloud sync & AI features) |
| **Barcode Scanner** | Any USB HID scanner (keyboard emulation) |

---

## 🛠️ Tech Stack

```mermaid
graph TB
    subgraph Desktop["Desktop Application (Tauri 2.x)"]
        direction TB
        Frontend["React 18 + TypeScript + Vite"]
        UI["shadcn/ui + Tailwind CSS"]
        Backend["Rust Backend"]
        SQLite["SQLite Database"]
        Frontend --> UI
        Frontend --> Backend
        Backend --> SQLite
    end

    subgraph Cloud["Cloud Services"]
        Supabase["Supabase (Postgres + Realtime)"]
        Groq["Groq Cloud (AI)"]
        EdgeFn["Edge Functions"]
        Supabase --> EdgeFn
        EdgeFn --> Groq
    end

    Backend -->|"Background Sync"| Supabase
    Frontend -->|"AI Queries"| EdgeFn
```

---

## 📄 License

This project is proprietary software developed for AT-IBA-PA MINIMART.

---

## 📝 Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.
