# ⚙️ CoreInventory — Technical Requirements Document (TRD) v1.0
> Technical blueprint for engineers. Covers architecture, stack decisions, data layer, API contracts, business logic, security, and deployment.

---

## 1. Document Purpose

This TRD translates the CoreInventory PRD into precise technical specifications. It is the single source of truth for every engineering decision — what to build, how to build it, and why each choice was made.

**Audience:** Frontend Engineers, Backend Engineers, DevOps, Code Reviewers
**Companion Docs:** PRD v2.0, PROJECT_BLUEPRINT v2.0

---

## 2. System Architecture

### 2.1 Architecture Pattern
**Decoupled SPA + REST API**

```
┌─────────────────────────────────────────────────────────┐
│                      BROWSER                            │
│                                                         │
│   React SPA (Vite)                                      │
│   ├── React Router v6  (client-side routing)            │
│   ├── Zustand          (global state)                   │
│   ├── Axios            (HTTP client + interceptors)     │
│   └── Recharts         (data visualization)             │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTPS  (JSON REST)
                        │ Authorization: Bearer <JWT>
┌───────────────────────▼─────────────────────────────────┐
│                   NODE.JS SERVER                        │
│                                                         │
│   Express.js                                            │
│   ├── Middleware: CORS, JSON parser, Auth, ErrorHandler │
│   ├── Routes   → Controllers → Utils                    │
│   └── better-sqlite3  (synchronous SQLite driver)       │
│                                                         │
│   SQLite Database  (file: coreinventory.db)             │
│   └── WAL mode enabled for concurrent reads            │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Why This Architecture?
| Decision | Reason |
|----------|--------|
| Decoupled frontend/backend | Independent deployment, clear contracts |
| SQLite over PostgreSQL | Zero setup, file-based, sufficient for single-server V1 |
| REST over GraphQL | Simpler to build, debug, and document |
| Zustand over Redux | Less boilerplate, same power for this app size |
| Vite over CRA | 10x faster dev server, native ESM |

### 2.3 Communication Contract
- All API requests/responses use **JSON**
- All protected routes require header: `Authorization: Bearer <token>`
- All responses follow a **standard envelope**:

```json
// Success
{
  "success": true,
  "data": { ... },
  "message": "Operation completed"
}

// Error
{
  "success": false,
  "error": "Error message here",
  "code": "VALIDATION_ERROR"
}

// Paginated list
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 84,
    "totalPages": 5
  }
}
```

---

## 3. Tech Stack — Detailed Specifications

### 3.1 Backend

| Package | Version | Purpose |
|---------|---------|---------|
| Node.js | >= 18.x | Runtime |
| Express | ^4.18.2 | HTTP framework |
| better-sqlite3 | ^9.4.3 | SQLite driver (sync, fast) |
| bcryptjs | ^2.4.3 | Password hashing (salt rounds: 12) |
| jsonwebtoken | ^9.0.2 | JWT sign / verify |
| cors | ^2.8.5 | Cross-origin headers |
| dotenv | ^16.4.1 | Environment variables |
| nodemon | ^3.0.3 | Dev auto-restart |

### 3.2 Frontend

| Package | Version | Purpose |
|---------|---------|---------|
| React | ^18.2.0 | UI framework |
| React DOM | ^18.2.0 | DOM renderer |
| React Router | ^6.22.0 | SPA routing |
| Zustand | ^4.5.0 | Global state management |
| Axios | ^1.6.7 | HTTP client |
| Recharts | ^2.12.0 | Charts & graphs |
| Lucide React | ^0.344.0 | Icon library |
| react-hot-toast | ^2.4.1 | Toast notifications |
| date-fns | ^3.3.1 | Date formatting & manipulation |
| clsx | ^2.1.0 | Conditional className utility |
| Tailwind CSS | ^3.4.1 | Utility-first CSS |
| Vite | ^5.1.3 | Build tool & dev server |

---

## 4. Database Design

### 4.1 SQLite Configuration

```javascript
// db/database.js
const Database = require('better-sqlite3');
const path = require('path');

const db = new Database(process.env.DB_PATH, {
  verbose: process.env.NODE_ENV === 'development' ? console.log : null
});

// Performance pragmas
db.pragma('journal_mode = WAL');     // Concurrent reads
db.pragma('foreign_keys = ON');      // Enforce FK constraints
db.pragma('synchronous = NORMAL');   // Balance safety/speed

module.exports = db;
```

### 4.2 Full Schema with Constraints

```sql
-- ─────────────────────────────────────────
-- USERS
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS users (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  name            TEXT    NOT NULL,
  email           TEXT    NOT NULL UNIQUE,
  password_hash   TEXT    NOT NULL,
  role            TEXT    NOT NULL DEFAULT 'warehouse_staff'
                          CHECK(role IN ('inventory_manager','warehouse_staff')),
  otp             TEXT,
  otp_expires_at  DATETIME,
  created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- CATEGORIES
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS categories (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  name        TEXT    NOT NULL UNIQUE,
  description TEXT,
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- WAREHOUSES & LOCATIONS
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS warehouses (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  name       TEXT    NOT NULL UNIQUE,
  address    TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS locations (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  warehouse_id INTEGER NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
  name         TEXT    NOT NULL,
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(warehouse_id, name)
);

-- ─────────────────────────────────────────
-- PRODUCTS
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS products (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  sku              TEXT    NOT NULL UNIQUE,
  name             TEXT    NOT NULL,
  category_id      INTEGER REFERENCES categories(id) ON DELETE SET NULL,
  unit_of_measure  TEXT    NOT NULL DEFAULT 'pcs',
  min_stock_level  INTEGER NOT NULL DEFAULT 10,
  created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at       DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Trigger: auto-update updated_at on product change
CREATE TRIGGER IF NOT EXISTS products_updated_at
AFTER UPDATE ON products
BEGIN
  UPDATE products SET updated_at = CURRENT_TIMESTAMP WHERE id = NEW.id;
END;

-- Per-location stock ledger (source of truth for quantity)
CREATE TABLE IF NOT EXISTS product_stock (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id  INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  location_id INTEGER NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
  quantity    INTEGER NOT NULL DEFAULT 0 CHECK(quantity >= 0),
  UNIQUE(product_id, location_id)
);

-- ─────────────────────────────────────────
-- RECEIPTS (Incoming Goods)
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS receipts (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  supplier       TEXT    NOT NULL,
  status         TEXT    NOT NULL DEFAULT 'draft'
                         CHECK(status IN ('draft','waiting','ready','done','canceled')),
  reference      TEXT,
  notes          TEXT,
  scheduled_date DATE,
  validated_at   DATETIME,
  created_by     INTEGER REFERENCES users(id),
  created_at     DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS receipt_items (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  receipt_id  INTEGER NOT NULL REFERENCES receipts(id) ON DELETE CASCADE,
  product_id  INTEGER NOT NULL REFERENCES products(id),
  location_id INTEGER NOT NULL REFERENCES locations(id),
  quantity    INTEGER NOT NULL CHECK(quantity > 0)
);

-- ─────────────────────────────────────────
-- DELIVERIES (Outgoing Goods)
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS deliveries (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  customer       TEXT    NOT NULL,
  status         TEXT    NOT NULL DEFAULT 'draft'
                         CHECK(status IN ('draft','waiting','ready','done','canceled')),
  reference      TEXT,
  notes          TEXT,
  scheduled_date DATE,
  validated_at   DATETIME,
  created_by     INTEGER REFERENCES users(id),
  created_at     DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS delivery_items (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  delivery_id  INTEGER NOT NULL REFERENCES deliveries(id) ON DELETE CASCADE,
  product_id   INTEGER NOT NULL REFERENCES products(id),
  location_id  INTEGER NOT NULL REFERENCES locations(id),
  quantity     INTEGER NOT NULL CHECK(quantity > 0)
);

-- ─────────────────────────────────────────
-- INTERNAL TRANSFERS
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS transfers (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  from_location_id INTEGER NOT NULL REFERENCES locations(id),
  to_location_id   INTEGER NOT NULL REFERENCES locations(id),
  status           TEXT    NOT NULL DEFAULT 'draft'
                           CHECK(status IN ('draft','waiting','ready','done','canceled')),
  notes            TEXT,
  scheduled_date   DATE,
  validated_at     DATETIME,
  created_by       INTEGER REFERENCES users(id),
  created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
  CHECK(from_location_id != to_location_id)
);

CREATE TABLE IF NOT EXISTS transfer_items (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  transfer_id INTEGER NOT NULL REFERENCES transfers(id) ON DELETE CASCADE,
  product_id  INTEGER NOT NULL REFERENCES products(id),
  quantity    INTEGER NOT NULL CHECK(quantity > 0)
);

-- ─────────────────────────────────────────
-- STOCK ADJUSTMENTS
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS adjustments (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id  INTEGER NOT NULL REFERENCES products(id),
  location_id INTEGER NOT NULL REFERENCES locations(id),
  system_qty  INTEGER NOT NULL,     -- qty before adjustment
  counted_qty INTEGER NOT NULL,     -- physically counted qty
  difference  INTEGER NOT NULL,     -- counted_qty - system_qty
  reason      TEXT    NOT NULL,     -- damage | theft | correction | other
  created_by  INTEGER REFERENCES users(id),
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ─────────────────────────────────────────
-- STOCK MOVEMENT LEDGER (immutable audit log)
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS stock_movements (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id       INTEGER NOT NULL REFERENCES products(id),
  type             TEXT    NOT NULL
                           CHECK(type IN ('receipt','delivery','transfer','adjustment')),
  from_location_id INTEGER REFERENCES locations(id),
  to_location_id   INTEGER REFERENCES locations(id),
  qty_change       INTEGER NOT NULL,  -- positive = stock in, negative = stock out
  reference_type   TEXT,              -- 'receipt' | 'delivery' | 'transfer' | 'adjustment'
  reference_id     INTEGER,           -- FK to the originating document
  created_by       INTEGER REFERENCES users(id),
  created_at       DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX IF NOT EXISTS idx_product_stock_product ON product_stock(product_id);
CREATE INDEX IF NOT EXISTS idx_product_stock_location ON product_stock(location_id);
CREATE INDEX IF NOT EXISTS idx_movements_product ON stock_movements(product_id);
CREATE INDEX IF NOT EXISTS idx_movements_type ON stock_movements(type);
CREATE INDEX IF NOT EXISTS idx_movements_created ON stock_movements(created_at);
CREATE INDEX IF NOT EXISTS idx_receipts_status ON receipts(status);
CREATE INDEX IF NOT EXISTS idx_deliveries_status ON deliveries(status);
CREATE INDEX IF NOT EXISTS idx_transfers_status ON transfers(status);
```

### 4.3 Stock Quantity Rules

| Rule | Enforcement |
|------|-------------|
| Quantity never goes below 0 | `CHECK(quantity >= 0)` on `product_stock` |
| Item quantities must be > 0 | `CHECK(quantity > 0)` on all item tables |
| Transfer source ≠ destination | `CHECK(from_location_id != to_location_id)` |
| Only Done status mutates stock | Enforced in controller logic, not DB trigger |
| Ledger is append-only | No UPDATE/DELETE routes on `stock_movements` |

---

## 5. Backend Architecture

### 5.1 Server Entry Point (server.js)

```javascript
const express = require('express');
const cors = require('cors');
require('dotenv').config();

const app = express();

// Middleware
app.use(cors({ origin: process.env.CLIENT_URL || 'http://localhost:5173' }));
app.use(express.json());

// Routes
app.use('/api/auth',        require('./routes/auth.routes'));
app.use('/api/products',    require('./routes/products.routes'));
app.use('/api/categories',  require('./routes/categories.routes'));
app.use('/api/warehouses',  require('./routes/warehouses.routes'));
app.use('/api/locations',   require('./routes/locations.routes'));
app.use('/api/receipts',    require('./routes/receipts.routes'));
app.use('/api/deliveries',  require('./routes/deliveries.routes'));
app.use('/api/transfers',   require('./routes/transfers.routes'));
app.use('/api/adjustments', require('./routes/adjustments.routes'));
app.use('/api/movements',   require('./routes/movements.routes'));
app.use('/api/dashboard',   require('./routes/dashboard.routes'));

// Global error handler (must be last)
app.use(require('./middleware/errorHandler'));

app.listen(process.env.PORT || 5000, () =>
  console.log(`CoreInventory API running on port ${process.env.PORT || 5000}`)
);
```

### 5.2 Shared Stock Utility (utils/stock.js)

All stock mutations go through this file — never directly from controllers.

```javascript
// utils/stock.js
const db = require('../db/database');

/**
 * Increase stock for a product at a location.
 * Creates product_stock row if it doesn't exist.
 */
function stockIn(productId, locationId, qty) {
  db.prepare(`
    INSERT INTO product_stock (product_id, location_id, quantity)
    VALUES (?, ?, ?)
    ON CONFLICT(product_id, location_id)
    DO UPDATE SET quantity = quantity + excluded.quantity
  `).run(productId, locationId, qty);
}

/**
 * Decrease stock. Throws if insufficient stock.
 */
function stockOut(productId, locationId, qty) {
  const row = db.prepare(
    `SELECT quantity FROM product_stock WHERE product_id = ? AND location_id = ?`
  ).get(productId, locationId);

  if (!row || row.quantity < qty) {
    throw new Error(`Insufficient stock for product ${productId} at location ${locationId}`);
  }

  db.prepare(
    `UPDATE product_stock SET quantity = quantity - ? WHERE product_id = ? AND location_id = ?`
  ).run(qty, productId, locationId);
}

/**
 * Set stock to an exact quantity (used in adjustments).
 */
function stockSet(productId, locationId, qty) {
  db.prepare(`
    INSERT INTO product_stock (product_id, location_id, quantity)
    VALUES (?, ?, ?)
    ON CONFLICT(product_id, location_id)
    DO UPDATE SET quantity = excluded.quantity
  `).run(productId, locationId, qty);
}

module.exports = { stockIn, stockOut, stockSet };
```

### 5.3 Ledger Utility (utils/ledger.js)

```javascript
// utils/ledger.js
const db = require('../db/database');

/**
 * Append an entry to the stock_movements ledger.
 * Called automatically after every stock mutation.
 */
function appendLedger({
  productId, type, fromLocationId = null, toLocationId = null,
  qtyChange, referenceType = null, referenceId = null, createdBy = null
}) {
  db.prepare(`
    INSERT INTO stock_movements
      (product_id, type, from_location_id, to_location_id,
       qty_change, reference_type, reference_id, created_by)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
  `).run(productId, type, fromLocationId, toLocationId,
         qtyChange, referenceType, referenceId, createdBy);
}

module.exports = { appendLedger };
```

### 5.4 Validate Operation — Core Business Logic

All four operation types share the same validate pattern. Example for Receipts:

```javascript
// controllers/receipts.controller.js — validate()
function validate(req, res) {
  const receipt = db.prepare(`SELECT * FROM receipts WHERE id = ?`).get(req.params.id);

  // Guards
  if (!receipt) return res.status(404).json({ success: false, error: 'Receipt not found' });
  if (receipt.status === 'done') return res.status(400).json({ success: false, error: 'Already validated' });
  if (receipt.status === 'canceled') return res.status(400).json({ success: false, error: 'Cannot validate a canceled receipt' });

  const items = db.prepare(`SELECT * FROM receipt_items WHERE receipt_id = ?`).all(receipt.id);
  if (items.length === 0) return res.status(400).json({ success: false, error: 'Receipt has no items' });

  // Atomic transaction — all or nothing
  const doValidate = db.transaction(() => {
    for (const item of items) {
      stockIn(item.product_id, item.location_id, item.quantity);
      appendLedger({
        productId:      item.product_id,
        type:           'receipt',
        toLocationId:   item.location_id,
        qtyChange:      item.quantity,
        referenceType:  'receipt',
        referenceId:    receipt.id,
        createdBy:      req.user.id
      });
    }
    db.prepare(`
      UPDATE receipts SET status = 'done', validated_at = CURRENT_TIMESTAMP WHERE id = ?
    `).run(receipt.id);
  });

  doValidate();
  res.json({ success: true, message: 'Receipt validated. Stock updated.' });
}
```

**All four validate flows:**

| Operation | Stock Effect | Ledger qty_change |
|-----------|-------------|-------------------|
| Receipt validate | `stockIn(product, toLocation, qty)` | `+qty` |
| Delivery validate | `stockOut(product, fromLocation, qty)` | `-qty` |
| Transfer validate | `stockOut(from)` + `stockIn(to)` | `-qty` at from, `+qty` at to (2 ledger entries) |
| Adjustment create | `stockSet(product, location, counted_qty)` | `±difference` |

---

## 6. API Contract — Full Specification

### 6.1 Authentication

#### POST /api/auth/signup
```json
// Request
{ "name": "John Doe", "email": "john@co.com", "password": "min8chars", "role": "inventory_manager" }

// Response 201
{ "success": true, "data": { "token": "<jwt>", "user": { "id": 1, "name": "John Doe", "role": "inventory_manager" } } }
```

#### POST /api/auth/login
```json
// Request
{ "email": "john@co.com", "password": "mypassword" }

// Response 200
{ "success": true, "data": { "token": "<jwt>", "user": { "id": 1, "name": "John Doe", "role": "inventory_manager" } } }
```

#### POST /api/auth/forgot-password
```json
// Request
{ "email": "john@co.com" }
// Generates 6-digit OTP, stores hash + expiry in users table
// Response 200
{ "success": true, "message": "OTP sent", "otp": "123456" }
// Note: In production, send via email. In V1, return in response for testing.
```

#### POST /api/auth/verify-otp
```json
// Request
{ "email": "john@co.com", "otp": "123456" }
// Response 200
{ "success": true, "data": { "reset_token": "<short-lived-jwt>" } }
```

#### POST /api/auth/reset-password
```json
// Request (Authorization: Bearer <reset_token>)
{ "password": "newpassword123" }
// Response 200
{ "success": true, "message": "Password updated" }
```

---

### 6.2 Products

#### GET /api/products
Query params: `?search=steel&category_id=2&status=low_stock&page=1&limit=20`

Stock status computed field:
- `in_stock` → total qty > min_stock_level
- `low_stock` → 0 < total qty ≤ min_stock_level
- `out_of_stock` → total qty = 0

```json
// Response 200
{
  "success": true,
  "data": [
    {
      "id": 1, "sku": "STL-001", "name": "Steel Rod",
      "category": { "id": 2, "name": "Raw Materials" },
      "unit_of_measure": "kg",
      "min_stock_level": 50,
      "total_quantity": 77,
      "stock_status": "in_stock"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 20 }
}
```

#### GET /api/products/:id/stock
```json
// Response 200
{
  "success": true,
  "data": {
    "product_id": 1,
    "total_quantity": 77,
    "by_location": [
      { "location_id": 1, "location_name": "Main Store", "warehouse": "Main Warehouse", "quantity": 50 },
      { "location_id": 3, "location_name": "Production Rack", "warehouse": "Plant B", "quantity": 27 }
    ]
  }
}
```

---

### 6.3 Operations — Common Pattern

All operations (receipts, deliveries, transfers) share this structure:

#### POST /api/receipts
```json
// Request
{
  "supplier": "Vendor A",
  "reference": "PO-2026-001",
  "scheduled_date": "2026-03-15",
  "notes": "Urgent order",
  "items": [
    { "product_id": 1, "location_id": 2, "quantity": 50 },
    { "product_id": 2, "location_id": 2, "quantity": 100 }
  ]
}

// Response 201
{
  "success": true,
  "data": { "id": 7, "status": "draft", "supplier": "Vendor A", "items": [...] }
}
```

#### POST /api/receipts/:id/validate
```json
// Request: no body needed
// Response 200
{
  "success": true,
  "message": "Receipt validated. Stock updated for 2 products."
}
```

#### POST /api/receipts/:id/cancel
```json
// Response 200
{ "success": true, "message": "Receipt canceled." }
// Note: Cannot cancel a 'done' receipt.
```

---

### 6.4 Dashboard KPIs

#### GET /api/dashboard/kpis
```json
{
  "success": true,
  "data": {
    "total_products": 20,
    "low_stock_count": 4,
    "out_of_stock_count": 1,
    "pending_receipts": 3,
    "pending_deliveries": 2,
    "scheduled_transfers": 1
  }
}
```

#### GET /api/dashboard/recent-ops
```json
{
  "success": true,
  "data": [
    {
      "type": "receipt",
      "id": 7,
      "description": "Receipt from Vendor A",
      "status": "done",
      "created_at": "2026-03-14T08:30:00Z"
    }
  ]
}
```

---

### 6.5 Move History (Ledger)

#### GET /api/movements
Query: `?product_id=1&type=receipt&from=2026-03-01&to=2026-03-14&location_id=2&page=1&limit=50`

```json
{
  "success": true,
  "data": [
    {
      "id": 12,
      "product": { "id": 1, "sku": "STL-001", "name": "Steel Rod" },
      "type": "receipt",
      "from_location": null,
      "to_location": { "id": 2, "name": "Main Store", "warehouse": "Main Warehouse" },
      "qty_change": 50,
      "reference_type": "receipt",
      "reference_id": 7,
      "created_by": { "id": 1, "name": "Admin User" },
      "created_at": "2026-03-14T08:30:00Z"
    }
  ]
}
```

---

## 7. Authentication & Security

### 7.1 JWT Configuration
```javascript
// utils/jwt.js
const jwt = require('jsonwebtoken');

const sign = (payload) =>
  jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: process.env.JWT_EXPIRES_IN || '7d' });

const verify = (token) =>
  jwt.verify(token, process.env.JWT_SECRET);

module.exports = { sign, verify };
```

### 7.2 Auth Middleware
```javascript
// middleware/auth.js
const { verify } = require('../utils/jwt');

module.exports = (req, res, next) => {
  const header = req.headers.authorization;
  if (!header || !header.startsWith('Bearer ')) {
    return res.status(401).json({ success: false, error: 'Unauthorized' });
  }
  try {
    req.user = verify(header.split(' ')[1]);
    next();
  } catch {
    res.status(401).json({ success: false, error: 'Invalid or expired token' });
  }
};
```

### 7.3 OTP Flow
```
1. User requests forgot-password with email
2. Server generates: otp = randomInt(100000, 999999).toString()
3. Server stores: otp (plaintext for V1) + otp_expires_at = now + 15 minutes
4. Server returns OTP in response (V1 — no email service)
5. User submits email + OTP to /verify-otp
6. Server checks: otp matches AND otp_expires_at > now
7. Server issues short-lived reset_token (JWT, 15min expiry)
8. User submits new password with reset_token as Bearer
9. Server hashes new password, clears otp fields, saves
```

### 7.4 Security Measures
| Threat | Mitigation |
|--------|-----------|
| Brute force login | Rate limit: 10 req/min per IP (V2 — use express-rate-limit) |
| Weak passwords | Minimum 8 characters enforced in validation |
| SQL injection | Parameterized queries via better-sqlite3 prepared statements |
| Password exposure | bcrypt with salt rounds = 12 |
| Token theft | Short JWT expiry (7d), no refresh token in V1 |
| CORS | Whitelist only frontend origin |
| XSS | React escapes by default; no dangerouslySetInnerHTML |

---

## 8. Frontend Architecture

### 8.1 Axios Instance & Interceptors

```javascript
// api/axios.js
import axios from 'axios';
import { useAuthStore } from '../store/auth.store';
import toast from 'react-hot-toast';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
});

// Attach JWT to every request
api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Global error handling
api.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      useAuthStore.getState().logout();
      window.location.href = '/login';
    } else {
      toast.error(err.response?.data?.error || 'Something went wrong');
    }
    return Promise.reject(err);
  }
);

export default api;
```

### 8.2 Auth Store (Zustand)

```javascript
// store/auth.store.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useAuthStore = create(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,

      login: (user, token) => set({ user, token, isAuthenticated: true }),
      logout: () => set({ user: null, token: null, isAuthenticated: false }),
      updateUser: (user) => set({ user }),
    }),
    { name: 'auth-storage' }  // persists to localStorage
  )
);
```

### 8.3 Protected Route

```javascript
// components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuthStore } from '../store/auth.store';

export default function ProtectedRoute({ children }) {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated);
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}
```

### 8.4 Router Structure (App.jsx)

```jsx
<Routes>
  {/* Public */}
  <Route path="/login"            element={<Login />} />
  <Route path="/signup"           element={<Signup />} />
  <Route path="/forgot-password"  element={<ForgotPassword />} />
  <Route path="/reset-password"   element={<ResetPassword />} />

  {/* Protected — wrapped in AppLayout (sidebar + header) */}
  <Route path="/" element={<ProtectedRoute><AppLayout /></ProtectedRoute>}>
    <Route index                     element={<Navigate to="/dashboard" />} />
    <Route path="dashboard"          element={<Dashboard />} />
    <Route path="products"           element={<ProductList />} />
    <Route path="products/new"       element={<ProductForm />} />
    <Route path="products/:id"       element={<ProductDetail />} />
    <Route path="categories"         element={<CategoryManager />} />
    <Route path="receipts"           element={<ReceiptList />} />
    <Route path="receipts/new"       element={<ReceiptForm />} />
    <Route path="receipts/:id"       element={<ReceiptDetail />} />
    <Route path="deliveries"         element={<DeliveryList />} />
    <Route path="deliveries/new"     element={<DeliveryForm />} />
    <Route path="deliveries/:id"     element={<DeliveryDetail />} />
    <Route path="transfers"          element={<TransferList />} />
    <Route path="transfers/new"      element={<TransferForm />} />
    <Route path="transfers/:id"      element={<TransferDetail />} />
    <Route path="adjustments"        element={<AdjustmentList />} />
    <Route path="adjustments/new"    element={<AdjustmentForm />} />
    <Route path="move-history"       element={<MoveHistory />} />
    <Route path="alerts"             element={<LowStockAlerts />} />
    <Route path="settings/warehouses" element={<WarehouseManager />} />
    <Route path="profile"            element={<Profile />} />
  </Route>
</Routes>
```

### 8.5 UI Store (Zustand)

```javascript
// store/ui.store.js
import { create } from 'zustand';

export const useUIStore = create((set) => ({
  sidebarCollapsed: false,
  toggleSidebar: () => set((s) => ({ sidebarCollapsed: !s.sidebarCollapsed })),

  // Global confirm dialog
  confirmDialog: null,
  openConfirm: (config) => set({ confirmDialog: config }),
  closeConfirm: () => set({ confirmDialog: null }),
}));
```

---

## 9. Component Specifications

### 9.1 StatusStepper Component

Renders the Draft → Waiting → Ready → Done / Canceled pipeline visually.

```
Props:
  status: 'draft' | 'waiting' | 'ready' | 'done' | 'canceled'

Renders:
  [Draft] ──● [Waiting] ──● [Ready] ──● [Done]
                                        or
                           [Canceled] (red, if applicable)
```

### 9.2 ItemsTable Component (Operation forms)

Reusable editable table for adding product rows in Receipts / Deliveries / Transfers.

```
Props:
  items: Array<{ product_id, location_id, quantity }>
  onChange: (items) => void
  showLocation: boolean (hide for transfers — location is on the parent)
  readOnly: boolean (for detail views)

Features:
  - Add row button
  - Product dropdown (searchable)
  - Location dropdown (filtered by warehouse)
  - Quantity input (integer, min 1)
  - Remove row button
  - Running total display
```

### 9.3 OperationDetail Page Pattern

All detail pages (Receipt, Delivery, Transfer) follow this layout:

```
┌─────────────────────────────────────────────┐
│  [← Back]   Receipt #7 — Vendor A     [DONE]│  ← OperationHeader
├─────────────────────────────────────────────┤
│  [Draft] ──●─ [Waiting] ──●─ [Ready] ──●─ [Done]  │  ← StatusStepper
├─────────────────────────────────────────────┤
│  Details: Supplier, Reference, Date, Notes  │
├─────────────────────────────────────────────┤
│  Products Table (read-only in Done/Canceled) │  ← ItemsTable
├─────────────────────────────────────────────┤
│  [Cancel]                      [Validate ✓] │  ← Action buttons (hidden if done/canceled)
└─────────────────────────────────────────────┘
```

---

## 10. Design System — CSS Variables

```css
/* src/index.css */
@import url('https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=DM+Sans:wght@300;400;500;600&display=swap');

:root {
  /* Backgrounds */
  --bg-base:      #0a0d14;
  --bg-card:      #141720;
  --bg-hover:     #1e2230;
  --bg-input:     #1a1d27;

  /* Borders */
  --border:       #252836;
  --border-focus: #3b82f6;

  /* Brand */
  --accent:       #3b82f6;   /* Blue — primary CTA */
  --accent-hover: #2563eb;

  /* Status Colors */
  --status-draft:    #64748b;  /* Gray */
  --status-waiting:  #f59e0b;  /* Amber */
  --status-ready:    #3b82f6;  /* Blue */
  --status-done:     #10b981;  /* Green */
  --status-canceled: #ef4444;  /* Red */

  /* Stock Status */
  --stock-in:    #10b981;
  --stock-low:   #f59e0b;
  --stock-out:   #ef4444;

  /* Text */
  --text-primary:  #f0f4f8;
  --text-secondary: #94a3b8;
  --text-muted:    #5a6478;

  /* Shadows */
  --shadow-card: 0 4px 24px rgba(0,0,0,0.4);

  /* Typography */
  --font-display: 'Syne', sans-serif;
  --font-body:    'DM Sans', sans-serif;

  /* Spacing scale */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 16px;
}

* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: var(--font-body);
  background: var(--bg-base);
  color: var(--text-primary);
}
h1, h2, h3, h4 { font-family: var(--font-display); }
```

---

## 11. Error Handling

### 11.1 Backend Error Handler

```javascript
// middleware/errorHandler.js
module.exports = (err, req, res, next) => {
  console.error(`[${new Date().toISOString()}] ${err.message}`);

  // SQLite constraint errors
  if (err.message?.includes('UNIQUE constraint')) {
    return res.status(409).json({ success: false, error: 'Record already exists', code: 'DUPLICATE' });
  }
  if (err.message?.includes('FOREIGN KEY constraint')) {
    return res.status(400).json({ success: false, error: 'Referenced record not found', code: 'FK_ERROR' });
  }
  if (err.message?.includes('CHECK constraint')) {
    return res.status(400).json({ success: false, error: 'Invalid data — constraint violation', code: 'CONSTRAINT' });
  }
  if (err.message?.includes('Insufficient stock')) {
    return res.status(422).json({ success: false, error: err.message, code: 'INSUFFICIENT_STOCK' });
  }

  // Default
  res.status(500).json({ success: false, error: 'Internal server error', code: 'SERVER_ERROR' });
};
```

### 11.2 Frontend Error Strategy

| Scenario | Handling |
|----------|----------|
| 401 Unauthorized | Auto-logout + redirect to /login |
| 409 Duplicate | Toast: "Already exists" |
| 422 Insufficient stock | Toast: "Not enough stock at this location" |
| 500 Server error | Toast: "Something went wrong, try again" |
| Network offline | Toast: "Connection lost" |
| Form validation | Inline field errors, no API call |

---

## 12. Input Validation Rules

### Backend (controller level)
```javascript
// All validation done inline in controllers (no external library in V1)

// Product
name:             required, string, max 200 chars
sku:              required, string, unique, max 50 chars
unit_of_measure:  required, one of: ['kg','g','pcs','litre','ml','box','bag','m','cm']
min_stock_level:  required, integer >= 0

// Operation items
quantity:         required, integer >= 1
product_id:       required, must exist in products table
location_id:      required, must exist in locations table

// Transfer
from_location_id != to_location_id  (enforced in DB + controller)

// Adjustment
reason: required, one of: ['damage','theft','correction','other']
```

### Frontend (form level)
- All required fields highlighted on submit attempt
- Numeric inputs: reject non-integer, reject ≤ 0
- SKU: auto-uppercase on blur
- Quantity: cannot exceed available stock (for deliveries — checked client + server)

---

## 13. Performance Requirements

| Metric | Target | Method |
|--------|--------|--------|
| API response time | < 300ms p95 | SQLite indexes + sync queries |
| Page initial load | < 2s | Vite code splitting |
| Dashboard KPI load | < 500ms | Optimized COUNT queries |
| Ledger query (1000 rows) | < 200ms | Index on created_at, product_id |
| Frontend bundle size | < 500KB gzipped | Lazy-loaded routes |

### Code Splitting (React Router lazy loading)
```javascript
const Dashboard    = lazy(() => import('./pages/Dashboard'));
const ProductList  = lazy(() => import('./pages/products/ProductList'));
const MoveHistory  = lazy(() => import('./pages/operations/MoveHistory'));
// ... all pages lazy loaded
```

---

## 14. Environment Configuration

### .env (Backend)
```env
PORT=5000
JWT_SECRET=change_this_to_a_long_random_string_in_production
JWT_EXPIRES_IN=7d
DB_PATH=./db/coreinventory.db
CLIENT_URL=http://localhost:5173
NODE_ENV=development
```

### .env (Frontend — vite env)
```env
VITE_API_URL=http://localhost:5000/api
```

---

## 15. Seed Data Specification (db/seed.js)

```
Users (2):
  { name: 'Admin User',  email: 'admin@coreinventory.com', password: 'admin123', role: 'inventory_manager' }
  { name: 'Staff User',  email: 'staff@coreinventory.com', password: 'staff123', role: 'warehouse_staff' }

Warehouses (2):
  { name: 'Main Warehouse', address: '12 Industrial Ave' }
  { name: 'Plant B',        address: '45 Production Rd' }

Locations (6):
  Main Warehouse: ['Main Store', 'Rack A', 'Rack B']
  Plant B:        ['Production Floor', 'Rack C', 'Dispatch Bay']

Categories (5):
  Raw Materials | Electronics | Packaging | Tools | Finished Goods

Products (20):
  4 per category, varying min_stock_level values
  Initial stock distributed across locations:
    8 products  → in_stock  (qty well above min)
    7 products  → low_stock (qty ≤ min_stock_level)
    5 products  → out_of_stock (qty = 0)

Receipts (5):
  2x done (with matching stock movements), 2x waiting, 1x draft

Deliveries (4):
  2x done (with matching stock movements), 1x ready, 1x draft

Transfers (3):
  1x done (stock moved between locations), 1x waiting, 1x draft

Adjustments (8):
  Mix of: damage, theft, correction reasons
  All with matching stock movements

Stock Movements:
  Auto-generated from all validated operations above
```

---

## 16. File Naming Conventions

| Type | Convention | Example |
|------|-----------|---------|
| React components | PascalCase | `ProductList.jsx` |
| Hooks | camelCase prefixed `use` | `useProducts.js` |
| Stores | camelCase + `.store.js` | `auth.store.js` |
| API files | camelCase + `.api.js` | `products.api.js` |
| Utils | camelCase + `.js` | `formatters.js` |
| Backend routes | camelCase + `.routes.js` | `receipts.routes.js` |
| Backend controllers | camelCase + `.controller.js` | `receipts.controller.js` |
| CSS classes | kebab-case | `.status-badge--done` |

---

## 17. Git Branching Strategy

```
main          → production-ready, always deployable
dev           → integration branch
feature/*     → feature branches (merge to dev via PR)
fix/*         → bug fix branches

Examples:
  feature/receipt-validate-endpoint
  feature/dashboard-kpi-cards
  fix/stock-negative-quantity
```

---

## 18. Deployment (V1 — Single Server)

```
Server: Any Linux VPS (Ubuntu 22.04)
Process Manager: PM2

# Build frontend
cd frontend && npm run build     # outputs to dist/

# Serve frontend via Express static (or Nginx)
app.use(express.static('../frontend/dist'));

# PM2 process
pm2 start server.js --name coreinventory
pm2 save && pm2 startup

# SQLite backup (daily cron)
0 2 * * * cp /app/db/coreinventory.db /backups/coreinventory_$(date +%Y%m%d).db
```

---

*TRD v1.0 | CoreInventory | 2026-03-14*
