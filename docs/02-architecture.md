# System Architecture

## High-Level Overview

The system is built as an **offline-first desktop application** using a local-first architecture. Every terminal runs its own local database and can operate without any network connectivity. When available, data syncs — either over the local network (LAN) to the Admin, or over the internet to the cloud.

```mermaid
graph TB
    subgraph CashierApp["Cashier App (Tauri)"]
        direction TB
        CashierUI["React Frontend<br/>POS Interface"]
        CashierRust["Rust Backend"]
        CashierDB[("SQLite<br/>pos-cashier.db")]
        CashierUI --> CashierRust
        CashierRust --> CashierDB
    end

    subgraph AdminApp["Admin App (Tauri)"]
        direction TB
        AdminUI["React Frontend<br/>Dashboard / Inventory / Reports"]
        AdminRust["Rust Backend"]
        AdminDB[("SQLite<br/>pos-admin.db")]
        WSServer["WebSocket Server<br/>:3080"]
        UDPBeacon["UDP Beacon<br/>:3081"]
        AdminUI --> AdminRust
        AdminRust --> AdminDB
        AdminRust --> WSServer
        AdminRust --> UDPBeacon
    end

    subgraph Cloud["Cloud (Supabase)"]
        Postgres[("Postgres Database")]
        Realtime["Realtime Engine"]
        EdgeFn["Edge Functions"]
        Groq["Groq AI"]
        Postgres --> Realtime
        EdgeFn --> Groq
    end

    CashierRust -->|"LAN Sync (WebSocket)"| WSServer
    CashierRust -.->|"Auto-discover (UDP)"| UDPBeacon
    CashierRust -->|"Cloud Sync (HTTPS)"| Postgres
    AdminRust -->|"Cloud Sync (HTTPS)"| Postgres
    AdminUI -->|"AI Queries"| EdgeFn
```

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Desktop Shell** | Tauri 2.x (Rust) | Native window management, system tray, IPC, file system |
| **Frontend** | React 18 + TypeScript | UI components, state management, routing |
| **Build Tool** | Vite 5 | Fast HMR development, production bundling |
| **UI Framework** | shadcn/ui + Tailwind CSS | Consistent, accessible component library |
| **Animations** | Framer Motion | 60FPS micro-interactions and transitions |
| **Charts** | Recharts | Sales trend visualizations, bar/line/pie charts |
| **Local Database** | SQLite (via sqlx) | Persistent local storage, WAL mode |
| **Cloud Database** | Supabase (Postgres) | Centralized data, Realtime subscriptions |
| **AI Engine** | Groq Cloud | Fast LLM inference for analytics |
| **Networking** | axum + tokio-tungstenite | WebSocket server/client for LAN sync |
| **Discovery** | socket2 (UDP broadcast) | Automatic Admin discovery on LAN |

---

## Dual-App Architecture

Both the Admin and Cashier apps are built from the **same codebase** but compiled with different configurations. The build system uses environment variables and separate Tauri config files to produce two distinct applications.

```mermaid
graph LR
    subgraph Codebase["Shared Codebase"]
        SharedLib["src/lib/<br/>Database, DAL, Types, Utils"]
        SharedUI["src/components/ui/<br/>shadcn components"]
        RustBackend["src-tauri/src/<br/>Rust backend modules"]
    end

    subgraph AdminBuild["Admin Build"]
        AdminConf["admin.conf.json"]
        AdminEntry["src/admin/main.tsx"]
        AdminPages["Dashboard, Inventory,<br/>Reports, Transactions,<br/>Settings"]
    end

    subgraph CashierBuild["Cashier Build"]
        CashierConf["cashier.conf.json"]
        CashierEntry["src/cashier/main.tsx"]
        CashierPages["POS, Login,<br/>Customer Display"]
    end

    SharedLib --> AdminBuild
    SharedLib --> CashierBuild
    SharedUI --> AdminBuild
    SharedUI --> CashierBuild
    RustBackend --> AdminBuild
    RustBackend --> CashierBuild
```

| Aspect | Admin App | Cashier App |
|--------|-----------|-------------|
| **Identifier** | `com.pos.admin` | `com.pos.cashier` |
| **Window Title** | Admin IMS | Cashier POS |
| **Entry Point** | `admin.html` → `src/admin/main.tsx` | `cashier.html` → `src/cashier/main.tsx` |
| **Pages** | Dashboard, Inventory, Transactions, Reports, Settings | POS, Login, Customer Display |
| **Network Role** | WebSocket server + UDP beacon broadcaster | WebSocket client + UDP listener |
| **Database File** | `pos-admin.db` | `pos-cashier.db` |
| **Cloud Sync** | Pulls products + pushes admin-received transactions | Pushes transactions + pulls products |

---

## Data Flow

```mermaid
flowchart LR
    subgraph DataTypes["Data Flow by Type"]
        direction TB
        Products["📦 Products & Prices"]
        Transactions["💰 Transactions"]
        Stock["📊 Stock Levels"]
    end

    subgraph Sources
        SupabaseCloud["☁️ Supabase"]
        AdminLocal["🖥️ Admin Local"]
        CashierLocal["💻 Cashier Local"]
    end

    SupabaseCloud -->|"Downstream"| AdminLocal
    SupabaseCloud -->|"Downstream"| CashierLocal
    CashierLocal -->|"Upstream"| AdminLocal
    AdminLocal -->|"Upstream"| SupabaseCloud
    CashierLocal -.->|"Direct (if online)"| SupabaseCloud
```

| Data Type | Primary Source | Sync Direction |
|-----------|---------------|----------------|
| **Products / Prices** | Admin / Supabase | Supabase → Admin → Cashier (downstream) |
| **Transactions / Sales** | Cashier | Cashier → Admin → Supabase (upstream) |
| **Stock Levels** | Supabase (atomic) | Bidirectional across all terminals |
| **Users / Settings** | Admin | Admin → Cashier (via LAN initial sync) |

---

## IPC Boundary

The React frontend never accesses the database directly. All database operations flow through a strict boundary:

```mermaid
sequenceDiagram
    participant UI as React Frontend
    participant DAL as Data Access Layer
    participant IPC as Tauri IPC (invoke)
    participant Rust as Rust Backend
    participant DB as SQLite

    UI->>DAL: getAllProducts()
    DAL->>IPC: invoke('db_query', { sql, params })
    IPC->>Rust: Command handler
    Rust->>DB: sqlx::query()
    DB-->>Rust: Rows
    Rust-->>IPC: JSON result
    IPC-->>DAL: Parsed data
    DAL-->>UI: Product[]
```

The **Data Access Layer (DAL)** encapsulates all SQL queries in TypeScript functions. Pages never write raw SQL — they call DAL functions like `getAllProducts()`, `createTransaction()`, and `getDailyRevenue()`.

---

## Application Startup Sequence

```mermaid
sequenceDiagram
    participant App as Tauri App
    participant Splash as Splash Screen
    participant DB as SQLite Init
    participant Sync as Cloud Sync
    participant LAN as LAN Server/Client
    participant Main as Main Window

    App->>Splash: Show splash screen
    App->>DB: Initialize SQLite (create tables, WAL mode)
    DB-->>App: Pool ready
    App->>Sync: Initial cloud sync (with timeout)
    Sync-->>App: Sync complete or timeout
    App->>Splash: Close splash (min 3s shown)
    App->>Main: Show main window
    
    alt Admin Mode
        App->>LAN: Start WebSocket server (:3080)
        App->>LAN: Start UDP beacon (:3081)
    else Cashier Mode
        App->>LAN: Start WebSocket client (discover/connect)
    end
    
    App->>Sync: Start background sync loop (every 5s)
```

1. **Splash screen** appears with theme-aware background color
2. **Database initializes** — creates tables if first launch, enables WAL mode
3. **Initial sync** attempts to pull latest data from Supabase (max 10s timeout)
4. **Splash closes** after minimum 3 seconds
5. **LAN services start** — Admin launches WebSocket server + beacon; Cashier discovers and connects
6. **Background sync** runs every 5 seconds, pushing local changes and pulling remote updates
