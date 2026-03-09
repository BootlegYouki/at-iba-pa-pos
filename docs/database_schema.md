# Database Schema Documentation

This document outlines the database schema for the Point of Sale (POS) and Inventory Management System (IMS) application. The backend uses SQLite, managed via Tauri and Rust (`src-tauri/src/db.rs`).

## Tables Overview

### 1. `products`
Stores all inventory items.
- `id` (TEXT, PRIMARY KEY): Unique identifier.
- `barcode` (TEXT, UNIQUE NOT NULL): Product barcode.
- `name` (TEXT, NOT NULL): Product name.
- `price` (REAL, NOT NULL): Selling price.
- `stock` (INTEGER, NOT NULL, DEFAULT 0): Current stock level.
- `category` (TEXT, NOT NULL, DEFAULT 'Uncategorized'): Product category.
- `sku` (TEXT): Stock Keeping Unit identifier.
- `low_stock_threshold` (INTEGER, NOT NULL, DEFAULT 10): Alert threshold for low stock.
- `created_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Creation timestamp (ISO 8601).
- `updated_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Last update timestamp (ISO 8601).
- `sync_status` (TEXT, NOT NULL, DEFAULT 'synced'): Synchronization status (`pending`, `synced`, `failed`, `deleted`).
- `is_active` (INTEGER, NOT NULL, DEFAULT 1): Soft delete flag (1 for active, 0 for archived).

### 2. `transactions`
Records each completed sale.
- `id` (TEXT, PRIMARY KEY): Unique identifier.
- `total` (REAL, NOT NULL): Total transaction amount.
- `cashier` (TEXT, NOT NULL): Name or ID of the cashier who processed the transaction.
- `payment_method` (TEXT, NOT NULL): E.g., Cash, Card, etc.
- `amount_paid` (REAL, NOT NULL): The amount tendered by the customer.
- `reference_number` (TEXT): Optional reference number for the transaction.
- `created_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Transaction timestamp.
- `sync_status` (TEXT, NOT NULL, DEFAULT 'pending'): Cloud synchronization status.

### 3. `transaction_items`
Records individual items sold within a transaction.
- `id` (TEXT, PRIMARY KEY): Unique line item identifier.
- `transaction_id` (TEXT, NOT NULL): Foreign key linking to `transactions(id)`.
- `product_id` (TEXT, NOT NULL): The ID of the product sold.
- `product_name` (TEXT, NOT NULL): Denormalized product name at the time of sale.
- `category` (TEXT, NOT NULL, DEFAULT 'Uncategorized'): Denormalized product category.
- `quantity` (INTEGER, NOT NULL): Quantity purchased.
- `price_at_sale` (REAL, NOT NULL): Price per unit at the time of sale.

### 4. `users`
System users (Auth/Staff accounts).
- `id` (TEXT, PRIMARY KEY): Unique identifier.
- `username` (TEXT, UNIQUE): Login username.
- `password` (TEXT): Hashed password (bcrypt).
- `name` (TEXT, NOT NULL): Display name of the user.
- `initials` (TEXT): User initials.
- `pin` (TEXT): Optional PIN for quick login.
- `role` (TEXT, NOT NULL): User role (`admin` or `cashier`).
- `status` (TEXT, NOT NULL, DEFAULT 'active'): Account status.
- `created_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Creation timestamp.
- `updated_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Last update timestamp.
- `sync_status` (TEXT, NOT NULL, DEFAULT 'synced'): Synchronization status.

### 5. `settings`
Key-value pair settings for the application.
- `key` (TEXT, PRIMARY KEY): Setting key (e.g., `store_name`, `store_subtitle`).
- `value` (TEXT, NOT NULL): Setting value.
- `updated_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Last update timestamp.
- `sync_status` (TEXT, NOT NULL, DEFAULT 'synced'): Synchronization status.

### 6. `categories`
Master list of local product categories (local-only, not synced to cloud).
- `name` (TEXT, PRIMARY KEY): Category name (e.g., `Beverages`, `Medicine`, `Snacks`).

### 7. `ai_conversations`
Stores conversation sessions with the AI assistant (local-only, Admin).
- `id` (TEXT, PRIMARY KEY): Unique identifier.
- `title` (TEXT, NOT NULL, DEFAULT 'New Chat'): Conversation title.
- `created_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Creation timestamp.
- `updated_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Last update timestamp.

### 8. `ai_messages`
Stores individual messages within an AI conversation.
- `id` (TEXT, PRIMARY KEY): Unique message identifier.
- `conversation_id` (TEXT, NOT NULL): Foreign key linking to `ai_conversations(id)`.
- `role` (TEXT, NOT NULL): Message sender role (e.g., `user`, `assistant`).
- `content` (TEXT, NOT NULL): Message content/text.
- `is_error` (INTEGER, NOT NULL, DEFAULT 0): Flag indicating if the message represents an error (1 for true, 0 for false).
- `created_at` (TEXT, NOT NULL, DEFAULT `datetime('now')`): Message timestamp.

## Additional Notes
- **Migrations & Initialization:** The database structure and default seeds (like an initial admin user and default categories) are initialized in Rust via sqlx upon app startup (`src-tauri/src/db.rs`). 
- **Synchronization:** Tables with a `sync_status` column are designed to interface with a remote backend (Supabase) for cloud backups and multi-device data consistency.
