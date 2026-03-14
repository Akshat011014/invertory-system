# 📦 CoreInventory — Inventory Management System
## Product Requirements Document (PRD) v2.0
> Updated from CoreInventory Problem Statement PDF

---

## 1. Overview

**Product Name:** CoreInventory
**Version:** 1.0
**Type:** Web Application
**Target Users:**
- **Inventory Managers** — manage incoming & outgoing stock
- **Warehouse Staff** — perform transfers, picking, shelving, and counting

**Goal:** Replace manual registers, Excel sheets, and scattered tracking methods with a centralized, real-time, easy-to-use inventory management app.

---

## 2. Problem Statement

Businesses managing inventory manually face:
- No real-time stock visibility across locations
- No structured receipts / delivery order flow
- No internal transfer tracking between warehouses/racks
- Stock mismatches with no adjustment audit trail
- No low-stock alerting system

**CoreInventory** solves all of this with a modular, operation-driven IMS.

---

## 3. Tech Stack

| Layer       | Choice                        | Reason                              |
|-------------|-------------------------------|-------------------------------------|
| Frontend    | React + Vite                  | Fast, component-driven              |
| Styling     | Tailwind CSS + custom CSS     | Utility-first, rapid UI             |
| State       | Zustand                       | Lightweight global state            |
| Backend     | Node.js + Express             | Simple REST API                     |
| Database    | SQLite via better-sqlite3     | Zero-config, file-based, fast       |
| Charts      | Recharts                      | React-native charting               |
| Icons       | Lucide React                  | Clean, consistent icons             |
| Routing     | React Router v6               | Client-side routing                 |

---

## 4. Authentication

- User **Sign Up / Login** with email + password
- **OTP-based password reset** flow
- On successful login → redirect to **Inventory Dashboard**
- JWT-based session management
- Roles: `inventory_manager` | `warehouse_staff`

---

## 5. Dashboard

The landing page after login. Shows a live snapshot of all inventory operations.

### 5.1 KPI Cards
| KPI | Description |
|-----|-------------|
| Total Products in Stock | Count of all active products |
| Low Stock / Out of Stock Items | Products at or below reorder threshold |
| Pending Receipts | Receipts in Draft or Waiting status |
| Pending Deliveries | Delivery orders not yet completed |
| Internal Transfers Scheduled | Transfers in Waiting/Ready state |

### 5.2 Dynamic Filters on Dashboard
- **By document type:** Receipts / Delivery Orders / Internal Transfers / Adjustments
- **By status:** Draft → Waiting → Ready → Done → Canceled
- **By warehouse or location**
- **By product category**

---

## 6. Navigation Structure

### Left Sidebar
```
📦 CoreInventory (logo)
─────────────────────────
🏠 Dashboard
📦 Products
   └─ Product List
   └─ Categories
   └─ Reordering Rules
⚙️  Operations
   └─ Receipts
   └─ Delivery Orders
   └─ Internal Transfers
   └─ Stock Adjustments
   └─ Move History
🏭 Warehouse (Settings)
─────────────────────────
👤 My Profile
🚪 Logout
```

---

## 7. Core Features

---

### 7.1 Product Management

Create and manage the product catalog.

**Product Fields:**
| Field | Type | Notes |
|-------|------|-------|
| Name | Text | Required |
| SKU / Code | Text | Unique identifier |
| Category | Dropdown | Links to Category table |
| Unit of Measure | Dropdown | kg, pcs, litre, box, etc. |
| Initial Stock | Number | Optional, sets opening balance |
| Min Stock Level | Number | Triggers low-stock alert |

**Features:**
- Create / Edit / Delete products
- View **stock availability per location** (per warehouse/rack)
- Manage **Product Categories** (CRUD)
- Set **Reordering Rules** per product (min qty → trigger alert)
- SKU search + smart filters

---

### 7.2 Receipts (Incoming Goods)

Used when items arrive from vendors/suppliers.

**Flow:**
```
Create Receipt → Add Supplier + Products → Input Quantities → Validate
                                                                   ↓
                                                          Stock increases automatically
```

**Receipt Statuses:** `Draft` → `Waiting` → `Ready` → `Done` | `Canceled`

**Fields:**
- Supplier name
- Product(s) + quantity received
- Reference / PO number (optional)
- Notes
- Date received

**On Validate:** stock quantity auto-increments per product per location.

**Example:** Receive 50 units "Steel Rods" → Stock +50

---

### 7.3 Delivery Orders (Outgoing Goods)

Used when stock leaves the warehouse for customer shipment.

**Flow:**
```
Create Delivery → Add Products + Quantities → Pick → Pack → Validate
                                                                ↓
                                                     Stock decreases automatically
```

**Delivery Statuses:** `Draft` → `Waiting` → `Ready` → `Done` | `Canceled`

**Fields:**
- Customer / destination
- Product(s) + quantity to deliver
- Reference number (optional)
- Notes
- Scheduled date

**On Validate:** stock quantity auto-decrements per product per location.

**Example:** Sales order for 10 chairs → Delivery reduces chairs by 10.

---

### 7.4 Internal Transfers

Move stock between locations inside the company.

**Use Cases:**
- Main Warehouse → Production Floor
- Rack A → Rack B
- Warehouse 1 → Warehouse 2

**Flow:**
```
Create Transfer → Select Source Location → Select Destination → Add Products + Qty → Validate
                                                                                        ↓
                                                                   Total stock unchanged,
                                                                   location updated in ledger
```

**Transfer Statuses:** `Draft` → `Waiting` → `Ready` → `Done` | `Canceled`

**Fields:**
- Source location (warehouse/rack)
- Destination location
- Product(s) + quantity
- Notes
- Scheduled date

Every movement is **logged in the Stock Ledger**.

---

### 7.5 Stock Adjustments

Fix mismatches between recorded stock and physical count.

**Flow:**
```
Select Product + Location → Enter Physically Counted Quantity
                                          ↓
                        System calculates difference → auto-updates stock
                                          ↓
                              Adjustment logged in ledger
```

**Fields:**
- Product
- Location
- Counted quantity (system shows current recorded qty)
- Reason (damage / theft / correction / other)

---

### 7.6 Move History (Stock Ledger)

A unified, read-only audit log of all stock movements.

**Columns:**
| Column | Description |
|--------|-------------|
| Date | Timestamp |
| Product | Name + SKU |
| Type | Receipt / Delivery / Transfer / Adjustment |
| From | Source location |
| To | Destination location |
| Qty Change | +/– quantity |
| Reference | Linked document |
| Done By | User who validated |

**Filters:** by product, type, date range, warehouse, user

---

### 7.7 Warehouse / Location Management (Settings)

- Create and manage **Warehouses**
- Create **Locations within warehouses** (Rack A, Production Floor, etc.)
- Assign products to default locations

---

## 8. Additional Features

- 🔔 **Low Stock Alerts** — qty ≤ min stock level, flagged on dashboard
- 🏭 **Multi-Warehouse Support** — stock tracked per warehouse + location
- 🔍 **SKU Search & Smart Filters** — across all operation pages
- 📊 **Dashboard Charts** — stock movement trends
- 📥 **CSV Export** — move history and low stock report

---

## 9. Inventory Flow Example (End-to-End)

```
Step 1 — Receive Goods
  Receipt: 100 kg Steel from Vendor A
  → Stock: +100 kg @ Main Store

Step 2 — Internal Transfer
  Transfer: Main Store → Production Rack
  → Total stock unchanged, location updated

Step 3 — Deliver Finished Goods
  Delivery Order: 20 kg Steel to Customer
  → Stock: –20 kg

Step 4 — Stock Adjustment
  3 kg Steel found damaged
  → Stock: –3 kg (Reason: Damage)

Final Stock: 77 kg ✅ — Everything logged in the Stock Ledger
```

---

## 10. Data Models

### User
```
id, name, email, password_hash, role, otp, otp_expires_at, created_at
```

### Category
```
id, name, description, created_at
```

### Warehouse
```
id, name, address, created_at
```

### Location
```
id, warehouse_id, name, created_at
```

### Product
```
id, sku, name, category_id, unit_of_measure,
min_stock_level, created_at, updated_at
```

### Product Stock (per location)
```
id, product_id, location_id, quantity
```

### Receipt
```
id, supplier, status, reference, notes,
scheduled_date, validated_at, created_by, created_at
```

### Receipt Item
```
id, receipt_id, product_id, location_id, quantity
```

### Delivery Order
```
id, customer, status, reference, notes,
scheduled_date, validated_at, created_by, created_at
```

### Delivery Item
```
id, delivery_id, product_id, location_id, quantity
```

### Internal Transfer
```
id, from_location_id, to_location_id, status, notes,
scheduled_date, validated_at, created_by, created_at
```

### Transfer Item
```
id, transfer_id, product_id, quantity
```

### Stock Adjustment
```
id, product_id, location_id, counted_qty, system_qty,
difference, reason, created_by, created_at
```

### Stock Movement Ledger
```
id, product_id, type (receipt/delivery/transfer/adjustment),
from_location_id, to_location_id, qty_change,
reference_type, reference_id, created_by, created_at
```

---

## 11. API Endpoints

### Auth
```
POST   /api/auth/signup
POST   /api/auth/login
POST   /api/auth/forgot-password
POST   /api/auth/verify-otp
POST   /api/auth/reset-password
GET    /api/auth/me
```

### Products
```
GET    /api/products
POST   /api/products
GET    /api/products/:id
PUT    /api/products/:id
DELETE /api/products/:id
GET    /api/products/:id/stock        (per location breakdown)
```

### Categories
```
GET/POST        /api/categories
PUT/DELETE      /api/categories/:id
```

### Warehouses & Locations
```
GET/POST        /api/warehouses
GET             /api/warehouses/:id/locations
GET/POST        /api/locations
PUT/DELETE      /api/locations/:id
```

### Receipts
```
GET    /api/receipts
POST   /api/receipts
GET    /api/receipts/:id
PUT    /api/receipts/:id
POST   /api/receipts/:id/validate
POST   /api/receipts/:id/cancel
```

### Delivery Orders
```
GET    /api/deliveries
POST   /api/deliveries
GET    /api/deliveries/:id
PUT    /api/deliveries/:id
POST   /api/deliveries/:id/validate
POST   /api/deliveries/:id/cancel
```

### Internal Transfers
```
GET    /api/transfers
POST   /api/transfers
GET    /api/transfers/:id
PUT    /api/transfers/:id
POST   /api/transfers/:id/validate
POST   /api/transfers/:id/cancel
```

### Stock Adjustments
```
GET    /api/adjustments
POST   /api/adjustments
GET    /api/adjustments/:id
```

### Move History
```
GET    /api/movements    (filters: product, type, date, location, user)
```

### Dashboard
```
GET    /api/dashboard/kpis
GET    /api/dashboard/alerts
GET    /api/dashboard/recent-ops
```

---

## 12. UI Routes

```
/login                    → Login
/signup                   → Sign Up
/forgot-password          → OTP Request
/reset-password           → Reset Password

/dashboard                → Dashboard

/products                 → Product List
/products/new             → Create Product
/products/:id             → Product Detail + Edit
/categories               → Category Manager
/reorder-rules            → Reordering Rules

/receipts                 → Receipts List
/receipts/new             → Create Receipt
/receipts/:id             → Receipt Detail

/deliveries               → Delivery Orders List
/deliveries/new           → Create Delivery
/deliveries/:id           → Delivery Detail

/transfers                → Internal Transfers List
/transfers/new            → Create Transfer
/transfers/:id            → Transfer Detail

/adjustments              → Adjustments List
/adjustments/new          → New Adjustment

/move-history             → Stock Ledger

/alerts                   → Low Stock Alerts

/settings/warehouses      → Warehouse + Location Manager

/profile                  → My Profile
```

---

## 13. Operation Status Flow

```
[Draft] → [Waiting] → [Ready] → [Done]
              ↓           ↓
          [Canceled]  [Canceled]

✅ Only "Done" (Validate) triggers automatic stock updates.
❌ Canceled operations have zero stock impact.
```

---

## 14. Non-Functional Requirements

| Area | Requirement |
|------|-------------|
| Performance | Page load < 2s, API < 300ms |
| Responsiveness | Desktop + Tablet |
| Security | JWT, bcrypt, input validation |
| Audit | Every stock change logged immutably in ledger |
| Reliability | SQLite WAL mode |

---

## 15. Out of Scope (V2+)

- Barcode / QR scanning
- Email / SMS notifications
- Mobile app
- ERP / Shopify integrations
- FIFO / LIFO costing methods
- Purchase Order management (formal vendor PO flow)

---

## 16. Milestones

| Phase | Deliverable | Est. |
|-------|-------------|------|
| 1 | Project setup + DB schema + seed data | Day 1 |
| 2 | Auth: signup, login, OTP reset | Day 2 |
| 3 | Backend: Products, Categories, Warehouses | Day 2-3 |
| 4 | Backend: Receipts + Deliveries + Transfers | Day 3-4 |
| 5 | Backend: Adjustments + Ledger + Dashboard KPIs | Day 4 |
| 6 | Frontend: Shell + Sidebar + Auth pages | Day 5 |
| 7 | Frontend: Dashboard + Products UI | Day 5-6 |
| 8 | Frontend: All Operations pages | Day 6-7 |
| 9 | Frontend: Ledger + Alerts + Settings | Day 7 |
| 10 | Polish, seed data, testing, README | Day 8 |

---

*PRD v2.0 | CoreInventory | Aligned with Problem Statement PDF | 2026-03-14*
