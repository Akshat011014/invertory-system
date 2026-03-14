# 🗂️ CoreInventory — Project Structure & Implementation Blueprint v2.0

---

## Folder Structure

```
coreinventory/
├── README.md
├── .env.example
├── .gitignore
│
├── backend/
│   ├── package.json
│   ├── server.js                        # Express entry point
│   ├── db/
│   │   ├── database.js                  # SQLite connection + WAL mode
│   │   ├── schema.sql                   # Full DB schema
│   │   └── seed.js                      # Demo data seed script
│   ├── middleware/
│   │   ├── auth.js                      # JWT verify middleware
│   │   └── errorHandler.js              # Global error handler
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── products.routes.js
│   │   ├── categories.routes.js
│   │   ├── warehouses.routes.js
│   │   ├── locations.routes.js
│   │   ├── receipts.routes.js
│   │   ├── deliveries.routes.js
│   │   ├── transfers.routes.js
│   │   ├── adjustments.routes.js
│   │   ├── movements.routes.js          # Ledger / Move History
│   │   └── dashboard.routes.js
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── products.controller.js
│   │   ├── categories.controller.js
│   │   ├── warehouses.controller.js
│   │   ├── locations.controller.js
│   │   ├── receipts.controller.js       # validate() auto-increments stock
│   │   ├── deliveries.controller.js     # validate() auto-decrements stock
│   │   ├── transfers.controller.js      # validate() updates location stock
│   │   ├── adjustments.controller.js    # auto-logs to ledger
│   │   ├── movements.controller.js      # read-only ledger queries
│   │   └── dashboard.controller.js      # KPIs + alerts + recent ops
│   └── utils/
│       ├── jwt.js                       # sign / verify helpers
│       ├── otp.js                       # OTP generate + verify
│       ├── stock.js                     # shared stock mutation helpers
│       └── ledger.js                    # append to stock_movements
│
└── frontend/
    ├── package.json
    ├── vite.config.js
    ├── index.html
    ├── tailwind.config.js
    ├── postcss.config.js
    └── src/
        ├── main.jsx
        ├── App.jsx                      # Router + protected routes
        ├── index.css                    # CSS variables + global styles
        │
        ├── api/
        │   ├── axios.js                 # Axios instance + JWT interceptor
        │   ├── auth.api.js
        │   ├── products.api.js
        │   ├── categories.api.js
        │   ├── warehouses.api.js
        │   ├── receipts.api.js
        │   ├── deliveries.api.js
        │   ├── transfers.api.js
        │   ├── adjustments.api.js
        │   ├── movements.api.js
        │   └── dashboard.api.js
        │
        ├── store/
        │   ├── auth.store.js
        │   ├── ui.store.js              # sidebar, theme, modals
        │   └── alerts.store.js          # low stock alerts count
        │
        ├── pages/
        │   ├── auth/
        │   │   ├── Login.jsx
        │   │   ├── Signup.jsx
        │   │   ├── ForgotPassword.jsx
        │   │   └── ResetPassword.jsx
        │   ├── Dashboard.jsx
        │   ├── products/
        │   │   ├── ProductList.jsx
        │   │   ├── ProductDetail.jsx
        │   │   ├── ProductForm.jsx
        │   │   └── CategoryManager.jsx
        │   ├── operations/
        │   │   ├── receipts/
        │   │   │   ├── ReceiptList.jsx
        │   │   │   ├── ReceiptForm.jsx
        │   │   │   └── ReceiptDetail.jsx
        │   │   ├── deliveries/
        │   │   │   ├── DeliveryList.jsx
        │   │   │   ├── DeliveryForm.jsx
        │   │   │   └── DeliveryDetail.jsx
        │   │   ├── transfers/
        │   │   │   ├── TransferList.jsx
        │   │   │   ├── TransferForm.jsx
        │   │   │   └── TransferDetail.jsx
        │   │   ├── adjustments/
        │   │   │   ├── AdjustmentList.jsx
        │   │   │   └── AdjustmentForm.jsx
        │   │   └── MoveHistory.jsx      # Stock Ledger view
        │   ├── alerts/
        │   │   └── LowStockAlerts.jsx
        │   ├── settings/
        │   │   └── WarehouseManager.jsx
        │   └── profile/
        │       └── Profile.jsx
        │
        ├── components/
        │   ├── layout/
        │   │   ├── AppLayout.jsx        # Sidebar + Header wrapper
        │   │   ├── Sidebar.jsx          # Left nav with all sections
        │   │   └── Header.jsx           # Top bar + user menu
        │   ├── ui/
        │   │   ├── Button.jsx
        │   │   ├── Input.jsx
        │   │   ├── Select.jsx
        │   │   ├── Modal.jsx
        │   │   ├── Table.jsx
        │   │   ├── Badge.jsx            # Status badges
        │   │   ├── Card.jsx
        │   │   ├── Spinner.jsx
        │   │   ├── Toast.jsx
        │   │   ├── EmptyState.jsx
        │   │   └── ConfirmDialog.jsx
        │   ├── dashboard/
        │   │   ├── KPICard.jsx
        │   │   ├── RecentOps.jsx        # Last 10 operations feed
        │   │   ├── DashboardFilters.jsx # type / status / warehouse / category
        │   │   └── StockChart.jsx
        │   ├── operations/
        │   │   ├── OperationHeader.jsx  # Title + Status badge + Action buttons
        │   │   ├── ItemsTable.jsx       # Editable product rows in forms
        │   │   └── StatusStepper.jsx    # Draft → Waiting → Ready → Done
        │   └── shared/
        │       ├── SearchBar.jsx
        │       ├── DateRangePicker.jsx
        │       ├── StockStatusBadge.jsx
        │       └── LocationSelector.jsx
        │
        └── utils/
            ├── formatters.js            # date, number, currency
            ├── validators.js
            └── csvExport.js
```

---

## Database Schema (schema.sql)

```sql
PRAGMA journal_mode=WAL;

CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role TEXT DEFAULT 'warehouse_staff'
    CHECK(role IN ('inventory_manager','warehouse_staff')),
  otp TEXT,
  otp_expires_at DATETIME,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL UNIQUE,
  description TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE warehouses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  address TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE locations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  warehouse_id INTEGER NOT NULL REFERENCES warehouses(id),
  name TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  sku TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  category_id INTEGER REFERENCES categories(id),
  unit_of_measure TEXT DEFAULT 'pcs',
  min_stock_level INTEGER DEFAULT 10,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_stock (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id INTEGER NOT NULL REFERENCES products(id),
  location_id INTEGER NOT NULL REFERENCES locations(id),
  quantity INTEGER DEFAULT 0,
  UNIQUE(product_id, location_id)
);

-- RECEIPTS (Incoming Goods)
CREATE TABLE receipts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  supplier TEXT NOT NULL,
  status TEXT DEFAULT 'draft'
    CHECK(status IN ('draft','waiting','ready','done','canceled')),
  reference TEXT,
  notes TEXT,
  scheduled_date DATE,
  validated_at DATETIME,
  created_by INTEGER REFERENCES users(id),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE receipt_items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  receipt_id INTEGER NOT NULL REFERENCES receipts(id) ON DELETE CASCADE,
  product_id INTEGER NOT NULL REFERENCES products(id),
  location_id INTEGER NOT NULL REFERENCES locations(id),
  quantity INTEGER NOT NULL
);

-- DELIVERY ORDERS (Outgoing Goods)
CREATE TABLE deliveries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  customer TEXT NOT NULL,
  status TEXT DEFAULT 'draft'
    CHECK(status IN ('draft','waiting','ready','done','canceled')),
  reference TEXT,
  notes TEXT,
  scheduled_date DATE,
  validated_at DATETIME,
  created_by INTEGER REFERENCES users(id),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE delivery_items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  delivery_id INTEGER NOT NULL REFERENCES deliveries(id) ON DELETE CASCADE,
  product_id INTEGER NOT NULL REFERENCES products(id),
  location_id INTEGER NOT NULL REFERENCES locations(id),
  quantity INTEGER NOT NULL
);

-- INTERNAL TRANSFERS
CREATE TABLE transfers (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  from_location_id INTEGER NOT NULL REFERENCES locations(id),
  to_location_id INTEGER NOT NULL REFERENCES locations(id),
  status TEXT DEFAULT 'draft'
    CHECK(status IN ('draft','waiting','ready','done','canceled')),
  notes TEXT,
  scheduled_date DATE,
  validated_at DATETIME,
  created_by INTEGER REFERENCES users(id),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE transfer_items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  transfer_id INTEGER NOT NULL REFERENCES transfers(id) ON DELETE CASCADE,
  product_id INTEGER NOT NULL REFERENCES products(id),
  quantity INTEGER NOT NULL
);

-- STOCK ADJUSTMENTS
CREATE TABLE adjustments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id INTEGER NOT NULL REFERENCES products(id),
  location_id INTEGER NOT NULL REFERENCES locations(id),
  system_qty INTEGER NOT NULL,
  counted_qty INTEGER NOT NULL,
  difference INTEGER NOT NULL,        -- counted - system
  reason TEXT,
  created_by INTEGER REFERENCES users(id),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- STOCK MOVEMENT LEDGER (immutable audit log)
CREATE TABLE stock_movements (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id INTEGER NOT NULL REFERENCES products(id),
  type TEXT NOT NULL
    CHECK(type IN ('receipt','delivery','transfer','adjustment')),
  from_location_id INTEGER REFERENCES locations(id),
  to_location_id INTEGER REFERENCES locations(id),
  qty_change INTEGER NOT NULL,        -- positive = stock in, negative = stock out
  reference_type TEXT,                -- 'receipt' | 'delivery' | 'transfer' | 'adjustment'
  reference_id INTEGER,
  created_by INTEGER REFERENCES users(id),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## Environment Variables (.env.example)

```env
# Backend
PORT=5000
JWT_SECRET=your_super_secret_key_change_this
JWT_EXPIRES_IN=7d
DB_PATH=./db/coreinventory.db

# Frontend
VITE_API_URL=http://localhost:5000/api
```

---

## Backend package.json

```json
{
  "name": "coreinventory-backend",
  "version": "1.0.0",
  "type": "commonjs",
  "scripts": {
    "dev": "nodemon server.js",
    "start": "node server.js",
    "seed": "node db/seed.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "better-sqlite3": "^9.4.3",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "cors": "^2.8.5",
    "dotenv": "^16.4.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.3"
  }
}
```

---

## Frontend package.json

```json
{
  "name": "coreinventory-frontend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.22.0",
    "zustand": "^4.5.0",
    "axios": "^1.6.7",
    "recharts": "^2.12.0",
    "lucide-react": "^0.344.0",
    "react-hot-toast": "^2.4.1",
    "date-fns": "^3.3.1",
    "clsx": "^2.1.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.1",
    "vite": "^5.1.3",
    "tailwindcss": "^3.4.1",
    "postcss": "^8.4.35",
    "autoprefixer": "^10.4.18"
  }
}
```

---

## Design System

### Color Palette
```css
:root {
  --bg-primary:    #0a0d14;   /* Deep dark background */
  --bg-card:       #141720;   /* Card / panel */
  --bg-hover:      #1e2230;   /* Hover / selected row */
  --border:        #252836;   /* Borders */
  --accent:        #3b82f6;   /* Blue primary */
  --accent-light:  #60a5fa;
  --success:       #10b981;   /* Green — in stock / done */
  --warning:       #f59e0b;   /* Amber — low / waiting */
  --danger:        #ef4444;   /* Red — out / canceled */
  --info:          #6366f1;   /* Indigo — transfer / info */
  --text-primary:  #f0f4f8;
  --text-muted:    #5a6478;
}
```

### Status Badge Colors
| Status | Color |
|--------|-------|
| Draft | Muted gray |
| Waiting | Amber |
| Ready | Blue |
| Done | Green |
| Canceled | Red |

### Typography
- **Display:** `Syne` — geometric, strong headings
- **Body/UI:** `DM Sans` — clean, readable

---

## Build Order (Implementation Phases)

### Phase 1 — Backend Core (Day 1-2)
1. `server.js` + Express setup
2. `db/database.js` + schema migration
3. `db/seed.js` — 2 warehouses, 6 locations, 5 categories, 20 products, 5 receipts, 4 deliveries, 3 transfers, 10 adjustments, 2 users
4. `utils/stock.js` — shared stock mutation (increment/decrement/transfer)
5. `utils/ledger.js` — append stock_movements on every operation

### Phase 2 — Auth (Day 2)
1. `auth.controller.js` — signup, login, JWT issue
2. `otp.js` — 6-digit OTP, 15min expiry
3. Forgot password / verify OTP / reset password flow
4. `middleware/auth.js` — protect all routes

### Phase 3 — Master Data APIs (Day 2-3)
1. Categories CRUD
2. Warehouses CRUD
3. Locations CRUD (nested under warehouse)
4. Products CRUD + `product_stock` seeding on creation
5. `GET /products/:id/stock` — per-location breakdown

### Phase 4 — Operations APIs (Day 3-4)
Build each with: list, create, detail, update, validate, cancel
1. Receipts → validate: `product_stock += qty` + ledger entry
2. Deliveries → validate: `product_stock -= qty` + ledger entry
3. Transfers → validate: `from -= qty`, `to += qty` + ledger entry
4. Adjustments → auto-validate: update stock to counted_qty + ledger entry

### Phase 5 — Ledger + Dashboard APIs (Day 4)
1. `GET /movements` with filters
2. `GET /dashboard/kpis` — 5 KPI values
3. `GET /dashboard/alerts` — products where qty ≤ min_stock_level
4. `GET /dashboard/recent-ops` — last 10 across all types

### Phase 6 — Frontend Shell (Day 5)
1. Vite + React + Tailwind + Router setup
2. CSS variables + Google Fonts (Syne + DM Sans)
3. `AppLayout` — collapsible left sidebar + top header
4. `Sidebar` — grouped nav (Products / Operations / Settings)
5. Auth pages: Login, Signup, Forgot/Reset Password
6. Protected route wrapper + `auth.store.js`

### Phase 7 — Dashboard Page (Day 5)
1. 5 KPI cards with icons + trend
2. Dynamic filter bar (type / status / warehouse / category)
3. Recent operations feed
4. Low stock mini-list

### Phase 8 — Products Section (Day 6)
1. Product list with search + SKU filter + stock status badge
2. Product detail — stock per location table
3. Create / Edit product form
4. Category manager modal

### Phase 9 — All Operations Pages (Day 6-7)
For each of Receipts / Deliveries / Transfers / Adjustments:
1. List page — table with status badge + filter
2. Detail page — `StatusStepper` + items table + Validate / Cancel buttons
3. Create form — add supplier/customer, add product rows, set location
Reuse `OperationHeader`, `ItemsTable`, `StatusStepper` components.

### Phase 10 — Ledger, Alerts, Settings, Polish (Day 7-8)
1. Move History — full ledger table with all filters + CSV export
2. Low Stock Alerts — list with reorder suggestion
3. Warehouse Manager — add/edit warehouses + locations
4. Profile page
5. Toast notifications for all actions
6. Empty states for all pages
7. Loading skeletons
8. README + quick start docs

---

## Demo Seed Data Plan

```
Users (2):
  admin@coreinventory.com  / admin123  (inventory_manager)
  staff@coreinventory.com  / staff123  (warehouse_staff)

Warehouses (2):
  Main Warehouse, Plant B

Locations (6):
  Main Warehouse: Main Store, Rack A, Rack B
  Plant B: Production Floor, Rack C, Dispatch Bay

Categories (5):
  Raw Materials, Electronics, Packaging, Tools, Finished Goods

Products (20):
  Mix of in-stock, low-stock, out-of-stock across locations

Receipts (5):   Mix of Draft/Done statuses
Deliveries (4): Mix of Waiting/Done
Transfers (3):  Mix of Ready/Done
Adjustments (10): Various reasons
Stock Movements: Auto-generated from above operations
```

---

## Quick Start

```bash
# Backend
cd backend
npm install
cp ../.env.example .env
npm run seed       # Creates DB + seeds demo data
npm run dev        # http://localhost:5000

# Frontend (new terminal)
cd frontend
npm install
npm run dev        # http://localhost:5173

# Demo login
admin@coreinventory.com / admin123
```

---

*Blueprint v2.0 | CoreInventory | Aligned with PDF spec | 2026-03-14*
