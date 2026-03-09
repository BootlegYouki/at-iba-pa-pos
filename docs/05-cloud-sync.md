# Cloud Sync Algorithm

## Overview

When internet connectivity is available, the system syncs data with **Supabase** — a cloud Postgres database that serves as the "Global Truth." This enables data backup, cross-device visibility, and cloud-powered features like AI analytics.

Cloud sync is a **background process** that runs alongside normal POS operations. It never blocks the UI or interrupts transactions.

---

## Runtime Cloud Configuration

Supabase credentials are **not compiled into the binary**. Instead, each store owner configures their own Supabase project at runtime through the Settings UI.

```mermaid
flowchart LR
    Settings["Settings UI"] --> Store["Tauri Store<br/>(.settings.dat)"]
    Store --> Rust["Rust Backend<br/>(DbState)"]
    Store --> Frontend["Frontend<br/>(supabase.ts)"]
    Rust --> SyncLoop["Sync Loop<br/>(starts/stops dynamically)"]
    Frontend --> Realtime["Realtime Subscriptions"]
```

### Credential Storage

| Key | Storage | Description |
|-----|---------|-------------|
| `supabaseUrl` | `.settings.dat` (JSON) | The Supabase project URL (e.g., `https://xxxxx.supabase.co`) |
| `supabaseAnonKey` | `.settings.dat` (JSON) | The Supabase anon/public key |

Credentials are read from the Tauri Store on app startup. If none are configured, the app starts in **offline mode** and cloud sync is skipped entirely.

### Runtime Commands

| Command | Description |
|---------|-------------|
| `update_cloud_config(url, key)` | Saves new credentials, cancels the existing sync loop, starts a new one, and **broadcasts `CloudCredentials` to all connected cashiers via LAN** |
| `clear_cloud_config()` | Removes credentials, stops the sync loop, and **broadcasts `CloudCredentialsCleared` to all connected cashiers via LAN** |
| `db_sync(entity?)` | Triggers a manual sync cycle (reads credentials from `DbState`) |

### Auto-Provisioning

When a user connects to a fresh Supabase project, the app detects missing tables (HTTP `42P01` error) and shows a **copyable SQL migration** that the user pastes into the Supabase SQL Editor. Once the tables exist, sync starts automatically.

---

## Cloud Credential Propagation via LAN

Cashier terminals do not need to be individually configured with Supabase credentials. When the Admin is connected to cashiers via LAN, cloud credentials are **automatically propagated**.

### How It Works

```mermaid
flowchart TD
    AdminConfig["Admin configures Supabase\nin Settings UI"] --> BroadcastCreds["Broadcast CloudCredentials\n{url, key} to all cashiers"]
    BroadcastCreds --> CashierReceive["Cashier receives credentials"]
    CashierReceive --> Encrypt["Encrypt with AES-256-GCM\n(derived from machine-specific key)"]
    Encrypt --> Store["Store encrypted in Tauri Store\n(.settings.dat)"]
    Store --> UpdateState["Update DbState\n(cloud_url, cloud_key)"]
    UpdateState --> EmitEvent["Emit 'cloud-credentials-received'\nto frontend"]
    EmitEvent --> StartSync["Frontend picks up creds\nand starts cloud sync"]
```

### Propagation Triggers

| Trigger | Message Sent | Behavior |
|---------|-------------|----------|
| Cashier sends `InitialSyncRequest` | `CloudCredentials` | Admin checks if cloud config exists and includes it in the initial sync response |
| Admin updates cloud config | `CloudCredentials` | Broadcast to all currently-connected cashiers immediately |
| Admin clears cloud config | `CloudCredentialsCleared` | Broadcast to all cashiers; each clears stored creds, aborts active cloud sync, and emits `'cloud-credentials-cleared'` to frontend |

### Credential Storage on Cashier

| Field | Storage Key | Encryption |
|-------|------------|------------|
| Supabase URL | `cloud_url` in Tauri Store | AES-256-GCM |
| Supabase Anon Key | `cloud_key` in Tauri Store | AES-256-GCM |

Credentials are encrypted at rest using AES-256-GCM with a machine-derived key before being written to the Tauri Store. On app startup, if encrypted credentials exist, they are decrypted and loaded into `DbState` for use by the sync loop.

> **Important:** Cashiers never expose the Supabase credentials in their Settings UI. The cloud configuration section shows only the connection status and is read-only — all configuration is managed centrally from the Admin.

---

## Sync Direction by Data Type

```mermaid
flowchart LR
    subgraph Cloud["☁️ Supabase (Postgres)"]
        CloudProducts[("Products")]
        CloudTransactions[("Transactions")]
        CloudStock[("Stock Levels")]
    end

    subgraph Local["💻 Local (SQLite)"]
        LocalProducts[("Products")]
        LocalTransactions[("Transactions")]
        LocalStock[("Stock Levels")]
    end

    CloudProducts -->|"⬇️ Downstream"| LocalProducts
    LocalTransactions -->|"⬆️ Upstream"| CloudTransactions
    CloudStock <-->|"↔️ Bidirectional"| LocalStock
```

| Data Type | Direction | Description |
|-----------|-----------|-------------|
| **Products & Prices** | Downstream (Cloud → Local) | Admin manages products in Supabase; terminals pull updates |
| **Transactions & Sales** | Upstream (Local → Cloud) | Sales are created locally and pushed to the cloud |
| **Stock Levels** | Bidirectional | Local deduction on sale + cloud trigger on sync |
| **Users & Settings** | Downstream on first sync | Pulled during initial bootstrap |

---

## Background Sync Loop

```mermaid
flowchart TD
    Start["Sync Loop Start"]
    
    Start --> AcquireLock["Acquire sync lock<br/>(prevents concurrent syncs)"]
    AcquireLock --> RunCycle["Run sync cycle"]
    
    RunCycle --> Push["PUSH: Find pending transactions<br/>→ Upsert to Supabase"]
    Push --> Pull["PULL: Fetch updated products<br/>← From Supabase"]
    Pull --> PullUsers["PULL: Fetch users & settings<br/>← From Supabase"]
    
    PullUsers --> Success{"Cycle<br/>succeeded?"}
    
    Success -->|Yes| ResetCounter["Reset failure counter<br/>sync_mode = 'online'"]
    Success -->|No| IncrCounter["Increment failure counter<br/>sync_mode = 'local' or 'offline'"]
    
    ResetCounter --> Delay["Wait 5 seconds"]
    IncrCounter --> BackoffDelay["Wait with backoff<br/>(5s × failures, max 30s)"]
    
    Delay --> AcquireLock
    BackoffDelay --> AcquireLock
```

### Timing

| State | Interval |
|-------|----------|
| **Online** | Every 5 seconds |
| **Failing** | Exponential backoff: `min(5 × (failures + 1), 30)` seconds |
| **Max backoff** | 30 seconds |

### Logging Strategy

- **First failure:** Logged at `warn` level
- **Subsequent failures:** Logged at `debug` level to reduce noise
- **Recovery:** Logged at `info` level with failure count

---

## Sync Cycle Detail

Each sync cycle consists of sequential push and pull phases. The Admin pushes products, users, settings, and transactions; the Cashier pushes only transactions. Both pull everything.

### Push Phase (Upstream)

All entities with `sync_status = 'pending'` are pushed in **batches of 50** per cycle:

```mermaid
sequenceDiagram
    participant Local as Local SQLite
    participant Sync as Sync Engine
    participant Cloud as Supabase

    Sync->>Local: SELECT * FROM transactions<br/>WHERE sync_status = 'pending' LIMIT 50
    Local-->>Sync: pending_txns[]
    
    loop For each pending transaction
        Sync->>Local: SELECT * FROM transaction_items<br/>WHERE transaction_id = ?
        Local-->>Sync: items[]
        Sync->>Cloud: POST /rest/v1/transactions<br/>(upsert, merge-duplicates)
        Sync->>Cloud: POST /rest/v1/transaction_items<br/>(upsert, merge-duplicates)
        Cloud-->>Sync: 201 Created
        Sync->>Local: UPDATE sync_status = 'synced'<br/>WHERE id = ?
    end
```

The Admin also pushes products, users, and settings with `sync_status = 'pending'` — these are changes made locally (e.g., adding a product, creating a user).

> **LAN-aware optimization:** When a Cashier has an active LAN connection to the Admin, it **skips pushing transactions to the cloud**. The Admin receives them via WebSocket and pushes them to the cloud on the Cashier's behalf. This prevents race conditions where cloud sync marks transactions as `synced` before the LAN send task picks them up.

### Pull Phase (Downstream)

Data is pulled from Supabase using **paginated flat queries** (1,000 records per page) instead of embedded resources. This avoids the slow JSON aggregation that PostgREST performs for embedded resources.

```mermaid
sequenceDiagram
    participant Local as Local SQLite
    participant Sync as Sync Engine
    participant Cloud as Supabase

    Note over Sync,Cloud: Paginated pull (1,000 per page)
    Sync->>Cloud: GET /rest/v1/products?limit=1000&offset=0
    Cloud-->>Sync: products[] (page 1)
    Sync->>Cloud: GET /rest/v1/products?limit=1000&offset=1000
    Cloud-->>Sync: products[] (page 2, if any)
    
    Note over Sync,Local: Bulk upsert in single SQL transaction
    Sync->>Local: BEGIN TRANSACTION
    loop For each product
        Sync->>Local: INSERT OR REPLACE INTO products<br/>WHERE sync_status != 'pending'
    end
    Sync->>Local: COMMIT

    Note over Sync,Cloud: Same pattern for transactions, users, settings
```

**Smart conflict avoidance:** The `WHERE sync_status != 'pending'` clause ensures cloud data never overwrites local pending changes.

| Mode | Max Pages per Entity | Max Records |
|------|---------------------|-------------|
| Admin | 50 | 50,000 |
| Cashier | 5 | 5,000 |

---

## Sync Status State Machine

Every transaction carries a `sync_status` field that tracks its sync lifecycle:

```mermaid
stateDiagram-v2
    [*] --> pending : Transaction created
    pending --> synced : Successfully pushed to cloud/Admin
    pending --> failed : Push failed after retries
    failed --> pending : Retry triggered
    synced --> [*]
```

| Status | Meaning |
|--------|---------|
| `pending` | Created locally, waiting to be pushed |
| `synced` | Successfully stored in Supabase (or acknowledged by Admin) |
| `failed` | Push attempt failed — will be retried |

---

## Cloud Stock Deduction (Postgres Trigger)

When transactions are pushed to Supabase, a **Postgres trigger** automatically deducts stock from the products table:

```mermaid
flowchart TD
    Insert["INSERT INTO transaction_items"]
    Insert --> Trigger["AFTER INSERT trigger:<br/>deduct_stock()"]
    Trigger --> Update["UPDATE products<br/>SET stock = stock - NEW.quantity<br/>WHERE id = NEW.product_id"]
    Update --> Done["Stock updated atomically"]
```

This ensures that stock deduction happens server-side regardless of which terminal pushed the transaction. The trigger is atomic — concurrent transactions are handled safely by Postgres.

---

## Initial Sync (During Splash Screen)

On app startup, a **one-time initial sync** runs during the splash screen:

```mermaid
sequenceDiagram
    participant App as App Startup
    participant Splash as Splash Screen
    participant Cloud as Supabase

    App->>Splash: Show splash
    App->>Cloud: Pull products, users, settings
    
    alt Sync succeeds within 10s
        Cloud-->>App: Data received
        App->>App: Upsert locally
    else Sync times out or fails
        App->>App: Continue with local data
    end
    
    App->>Splash: Close splash (min 3s)
    App->>App: Show main window
```

- **Minimum splash duration:** 3 seconds (ensures smooth visual transition)
- **Maximum sync timeout:** 10 seconds (prevents blocking on slow connections)
- **On failure:** App continues with whatever local data is available

---

## Admin vs Cashier Sync Behavior

| Behavior | Admin | Cashier |
|----------|-------|---------|
| **Push transactions** | Yes (its own + LAN-received cashier transactions) | Yes (its own) |
| **Pull products** | Yes | Yes |
| **Pull users** | Yes | Yes |
| **Offline mode label** | "Local Network" (always serves cashiers) | "Offline" (unless LAN-connected) |
| **LAN-received transactions** | Stored as `pending`, pushed to cloud | N/A |

---

## Conflict Resolution

Since data flows are mostly **unidirectional** (transactions up, products down), conflicts are rare. The strategies for each case:

| Data Type | Strategy | Detail |
|-----------|----------|--------|
| **Transactions** | No conflict possible | Always created locally, pushed upstream, never modified |
| **Products** | Cloud wins | Cloud is the master for product data; local edits are pushed to cloud first |
| **Stock** | Atomic deduction | Both local (SQLite transaction) and cloud (Postgres trigger) deduct atomically |
| **Users** | Cloud wins | User data is managed in Supabase, pulled downstream |
| **Duplicate pushes** | Upsert with merge | Supabase `UPSERT` with `resolution=merge-duplicates` ignores duplicate inserts |

---

## Supabase Realtime

The Supabase Postgres tables `products` and `transactions` have **Realtime** enabled. This allows the Admin dashboard to receive live updates without polling:

```mermaid
flowchart LR
    Change["Product updated in Supabase"]
    Change --> Realtime["Supabase Realtime Engine"]
    Realtime --> Broadcast["WebSocket broadcast"]
    Broadcast --> AdminApp["Admin App receives update"]
    AdminApp --> Refresh["UI refreshes data"]
```

This is used for:
- Live product price/stock updates reflected in the Admin UI
- Real-time transaction notifications when other terminals sync
