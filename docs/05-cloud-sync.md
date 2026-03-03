# Cloud Sync Algorithm

## Overview

When internet connectivity is available, the system syncs data with **Supabase** — a cloud Postgres database that serves as the "Global Truth." This enables data backup, cross-device visibility, and cloud-powered features like AI analytics.

Cloud sync is a **background process** that runs alongside normal POS operations. It never blocks the UI or interrupts transactions.

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

Each sync cycle consists of three sequential phases:

### Phase 1: Push Transactions (Upstream)

```mermaid
sequenceDiagram
    participant Local as Local SQLite
    participant Sync as Sync Engine
    participant Cloud as Supabase

    Sync->>Local: SELECT * FROM transactions<br/>WHERE sync_status = 'pending'
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

### Phase 2: Pull Products (Downstream)

```mermaid
sequenceDiagram
    participant Local as Local SQLite
    participant Sync as Sync Engine
    participant Cloud as Supabase

    Sync->>Cloud: GET /rest/v1/products?select=*
    Cloud-->>Sync: products[]
    
    loop For each product
        Sync->>Local: UPSERT into products<br/>(INSERT OR REPLACE)
    end
    
    Sync->>Local: DELETE local products<br/>not present in cloud response
```

### Phase 3: Pull Users & Settings (Downstream)

```mermaid
sequenceDiagram
    participant Local as Local SQLite
    participant Sync as Sync Engine
    participant Cloud as Supabase

    Sync->>Cloud: GET /rest/v1/users?select=*
    Cloud-->>Sync: users[]
    Sync->>Local: UPSERT users locally
    
    Sync->>Cloud: GET /rest/v1/settings?select=*
    Cloud-->>Sync: settings[]
    Sync->>Local: UPSERT settings locally
```

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
