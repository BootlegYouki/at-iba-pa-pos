# Database Schema Reference

This reference mirrors the SQLite schema created by the Rust startup logic.

## Tables

### 1. `products`

- `id` TEXT PRIMARY KEY
- `barcode` TEXT NOT NULL UNIQUE
- `name` TEXT NOT NULL
- `price` REAL NOT NULL
- `stock` INTEGER NOT NULL DEFAULT `0`
- `category` TEXT NOT NULL DEFAULT `'Uncategorized'`
- `sku` TEXT
- `low_stock_threshold` INTEGER NOT NULL DEFAULT `10`
- `created_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `updated_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'synced'`
- `is_active` INTEGER NOT NULL DEFAULT `1`

### 2. `product_deletions`

- `id` TEXT PRIMARY KEY
- `deleted_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'pending'`

### 3. `inventory_logs`

- `id` TEXT PRIMARY KEY
- `product_id` TEXT NOT NULL
- `change_amount` INTEGER NOT NULL
- `reason` TEXT NOT NULL
- `reference_id` TEXT
- `created_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'pending'`

### 4. `transactions`

- `id` TEXT PRIMARY KEY
- `total` REAL NOT NULL
- `cashier` TEXT NOT NULL
- `payment_method` TEXT NOT NULL
- `amount_paid` REAL NOT NULL
- `reference_number` TEXT
- `created_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'pending'`
- `lan_status` TEXT NOT NULL DEFAULT `'none'`

### 5. `transaction_items`

- `id` TEXT PRIMARY KEY
- `transaction_id` TEXT NOT NULL
- `product_id` TEXT NOT NULL
- `product_name` TEXT NOT NULL
- `category` TEXT NOT NULL DEFAULT `'Uncategorized'`
- `quantity` INTEGER NOT NULL
- `price_at_sale` REAL NOT NULL

Notes:

- `transaction_id` references `transactions(id)`.
- `product_id` is intentionally **not** enforced as a foreign key to `products`.

### 6. `users`

- `id` TEXT PRIMARY KEY
- `username` TEXT UNIQUE
- `password` TEXT
- `name` TEXT NOT NULL
- `initials` TEXT
- `pin` TEXT
- `role` TEXT NOT NULL
- `status` TEXT NOT NULL DEFAULT `'active'`
- `created_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `updated_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'synced'`

### 7. `user_deletions`

Tombstone table for cloud sync. Mirrors user hard-deletes.

- `id` TEXT PRIMARY KEY
- `deleted_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'pending'`

### 8. `settings`

- `key` TEXT PRIMARY KEY
- `value` TEXT NOT NULL
- `updated_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `sync_status` TEXT NOT NULL DEFAULT `'synced'`

### 9. `categories`

- `name` TEXT PRIMARY KEY

This table is local-only and used for category defaults plus inventory UI helpers.

### 10. `ai_conversations`

Admin-only local history table.

- `id` TEXT PRIMARY KEY
- `title` TEXT NOT NULL DEFAULT `'New Chat'`
- `created_at` TEXT NOT NULL DEFAULT `datetime('now')`
- `updated_at` TEXT NOT NULL DEFAULT `datetime('now')`

### 11. `ai_messages`

Admin-only local message table.

- `id` TEXT PRIMARY KEY
- `conversation_id` TEXT NOT NULL
- `role` TEXT NOT NULL
- `content` TEXT NOT NULL
- `is_error` INTEGER NOT NULL DEFAULT `0`
- `created_at` TEXT NOT NULL DEFAULT `datetime('now')`

## Seeded Defaults

On first initialization, the app seeds:

- A default category list
- `store_name = "My Store"`
- `store_subtitle = ""`
- Default Admin user if no Admin exists
- `system_instance_id` (UUID, admin DB only)

Default Admin credentials:

- Username: `admin`
- Password: `admin123`

## Important Notes

- AI config values are stored in `settings` as local-only rows (`sync_status = 'local'`) and are not intended for cloud overwrite.
- Cashier and Admin use the same logical schema, but only Admin creates the AI conversation tables.
- Product deletion sync uses `product_deletions` tombstones instead of only relying on hard deletes.
- User deletion sync uses `user_deletions` tombstones for the same reason.
- `transactions.lan_status` tracks the LAN push state independently from `sync_status`.
- The trigger `trg_apply_inventory_log_to_product_stock` updates `products.stock` on every `inventory_logs` INSERT where `sync_status != 'synced'`.
