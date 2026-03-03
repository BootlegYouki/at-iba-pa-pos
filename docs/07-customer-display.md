# Customer Display & Digital Receipts

## Overview

After a cashier logs in, a **second Tauri window** opens on the customer-facing screen. This window displays:

1. A **welcome/idle** state with store branding
2. **Live cart updates** as the cashier scans items
3. **Payment status** during checkout
4. A **receipt summary + QR code** after payment — the customer scans the QR to view/download their receipt on their phone

---

## Multi-Window Architecture

```mermaid
graph LR
    subgraph MainWindow["Main POS Window (Cashier)"]
        POS["POS.tsx<br/>Cashier controls"]
    end

    subgraph CustomerWindow["Customer Display Window"]
        Display["CustomerDisplay.tsx<br/>Customer-facing screen"]
    end

    subgraph ReceiptSite["Receipt Website (Vercel)"]
        Receipt["Static HTML page<br/>Decodes QR data"]
    end

    POS -->|"cart-update"| Display
    POS -->|"payment-state-change"| Display
    POS -->|"transaction-complete"| Display
    Display -->|"QR Code"| Receipt
```

The two windows communicate via **Tauri's `emitTo` API** — a one-way event broadcasting system from the main POS window to the customer display.

---

## Event Flow

```mermaid
sequenceDiagram
    participant Cashier as Main POS Window
    participant Display as Customer Display

    Note over Cashier,Display: Cashier logs in
    Cashier->>Display: Window opens (invoke open_customer_display)
    Display->>Display: Show idle/welcome state

    Note over Cashier,Display: Scanning items
    Cashier->>Display: cart-update { items, subtotal }
    Display->>Display: Show live cart list

    Cashier->>Display: cart-update { items, subtotal }
    Display->>Display: Update cart (animated)

    Note over Cashier,Display: Payment initiated
    Cashier->>Display: payment-state-change { state: "payment" }
    Display->>Display: Show "Processing Payment..."

    Note over Cashier,Display: Payment confirmed
    Cashier->>Display: transaction-complete { receipt data }
    Display->>Display: Show receipt + QR code

    Note over Display: After 20 seconds...
    Display->>Display: Auto-reset to idle
```

### Events

| Event | Payload | Direction | Trigger |
|-------|---------|-----------|---------|
| `cart-update` | `{ items: CartItem[], subtotal: number }` | Main → Display | Item added/removed/updated |
| `payment-state-change` | `{ state: "cart" \| "payment" \| "success" }` | Main → Display | Payment screen opened/closed |
| `transaction-complete` | Full receipt data (items, totals, cashier, etc.) | Main → Display | Payment confirmed |

---

## Display States

```mermaid
stateDiagram-v2
    [*] --> Idle : App opens / cart empty

    Idle --> Cart : First item scanned
    Cart --> Idle : Cart cleared
    
    Cart --> Processing : Payment initiated (F8)
    Processing --> Cart : Payment cancelled

    Processing --> Receipt : Payment confirmed
    Receipt --> Idle : Auto-reset after 20s
```

### State Details

| State | Display Content |
|-------|-----------------|
| **Idle** | Store name, logo, "Welcome" message, current date/time |
| **Cart** | Live item list with quantities, prices, running total (large, readable text) |
| **Processing** | "Processing Payment..." with animated dots, amount due |
| **Receipt** | Transaction summary, line items, totals, payment info, QR code |

---

## QR Code Digital Receipt

Instead of printing paper receipts, the system generates a **QR code** that the customer scans with their phone. The QR code encodes the entire receipt data into the URL hash.

```mermaid
flowchart TD
    Sale["Sale completed"]
    Sale --> Encode["Encode receipt data<br/>to compact JSON"]
    Encode --> Base64["Base64 encode"]
    Base64 --> URL["Construct URL:<br/>https://receipt-site.vercel.app/#&lt;data&gt;"]
    URL --> QR["Render QR code<br/>on customer display"]
    QR --> Scan["Customer scans<br/>with phone camera"]
    Scan --> Browser["Opens mobile browser"]
    Browser --> Decode["Decode hash → receipt data"]
    Decode --> Render["Render beautiful<br/>mobile receipt"]
    Render --> Download["Download as<br/>Image or PDF"]
```

### Data Encoding

Receipt data is encoded using **compact JSON keys** to minimize QR code size:

| Full Key | Compact Key | Type |
|----------|-------------|------|
| `storeName` | `s` | string |
| `items` | `i` | array |
| `items[].name` | `n` | string |
| `items[].quantity` | `q` | number |
| `items[].price` | `p` | number |
| `subtotal` | `st` | number |
| `paymentMethod` | `pm` | string |
| `amountPaid` | `ap` | number |
| `change` | `c` | number |
| `cashier` | `ca` | string |
| `transactionId` | `id` | string |
| `date` | `d` | ISO 8601 |
| `referenceNumber` | `r` | string or null |

### QR Data Budget

QR codes (Version 25, error correction L) can hold ~2,953 bytes of binary data.

| Content | Estimated Size |
|---------|---------------|
| Store name (20 chars) | ~25 bytes |
| 10 items (name + qty + price each) | ~500 bytes |
| Totals, payment, metadata | ~150 bytes |
| JSON overhead | ~100 bytes |
| **Raw JSON total** | **~775 bytes** |
| **After base64 (+33%)** | **~1,030 bytes** |
| **URL prefix** | ~50 bytes |
| **Grand total** | **~1,080 bytes** |

Even a receipt with 30 items (~2,400 bytes) fits comfortably within QR capacity.

---

## Receipt Website

The QR code points to a **standalone static website** hosted on Vercel. It requires no backend — all data is decoded from the URL hash.

```mermaid
flowchart LR
    subgraph Phone["Customer's Phone"]
        Camera["📷 Camera"]
        Browser["🌐 Mobile Browser"]
    end

    subgraph Vercel["Vercel (Static Host)"]
        HTML["index.html<br/>(single file)"]
    end

    Camera -->|"Scan QR"| Browser
    Browser -->|"GET /receipt#&lt;data&gt;"| HTML
    HTML -->|"Decode hash"| Browser
    Browser -->|"Render receipt"| Phone
```

### Receipt Website Features

- **Mobile-first design** with clean receipt layout
- Same font (Poppins) and color palette as the POS app
- **Download as Image** button (captures receipt as PNG)
- **Download as PDF** button (uses browser print dialog)
- **Error state** if hash is missing or invalid (shows store branding with "No receipt found")
- Completely static — no server-side logic, no database, no API calls

---

## Auto-Reopen Logic

If the customer display window is accidentally closed:

```mermaid
flowchart TD
    Scan["Cashier scans next item"]
    Scan --> Ensure["ensureCustomerDisplay()"]
    Ensure --> Check{"Window<br/>exists?"}
    Check -->|Yes| Emit["Emit cart-update"]
    Check -->|No| Reopen["invoke('open_customer_display')"]
    Reopen --> Emit
```

The `ensureCustomerDisplay()` function is called before every cart update emission. If the window was closed, it silently reopens it. The POS never crashes or errors because of a missing customer display window.

---

## Theme Enforcement

The customer display is **always in light mode** regardless of the cashier's theme preference. This is enforced at three levels:

| Level | Method |
|-------|--------|
| **Rust** | Window background color set to `#F2F5F8` (light) |
| **JavaScript injection** | `window.__DEVICE_THEME__ = 'light'` injected on window creation |
| **React** | `useEffect` removes `dark` class and adds `light` class on mount |

---

## Window Configuration

| Property | Value |
|----------|-------|
| **Window label** | `customer-display` |
| **Title** | "Customer Display" |
| **Default size** | 900 × 700 px |
| **Resizable** | Yes |
| **Decorations** | No (borderless — for kiosk-like display) |
| **Always on top** | No |
| **Route** | `#/customer-display` (outside protected route — no auth required) |
