# ☕ Café Rewards Counter — Loyalty & Points Management System

A production-quality loyalty and rewards management application built for café staff. Customers earn loyalty points on every purchase and can redeem them for artisanal coffees, pastries, and treats. Members progress through loyalty tiers (**Regular**, **Silver**, and **Gold**) earning points at higher rates.

---

## 📑 Table of Contents
1. [Core Invariant: Absolute Points Consistency](#-core-invariant-absolute-points-consistency)
2. [Key Features](#-key-features)
3. [Architecture & Monorepo Structure](#-architecture--monorepo-structure)
4. [Tech Stack](#-tech-stack)
5. [Prerequisites & Environment Setup](#-prerequisites--environment-setup)
6. [Installation & Seeding](#-installation--seeding)
7. [Running the Application](#-running-the-application)
8. [Debugging Guide](#-debugging-guide)
9. [Automated Testing & Concurrency Validation](#-automated-testing--concurrency-validation)
10. [Complete API Endpoints Reference](#-complete-api-endpoints-reference)
11. [Troubleshooting & FAQs](#-troubleshooting--faqs)

---

## 🛡️ Core Invariant: Absolute Points Consistency

The points engine strictly enforces **zero-discrepancy ledger auditability** and **race-condition safety**:

- **No Negative Balances**: `currentPoints >= 0` is strictly enforced at database, schema, and transactional level.
- **Race-Condition Safety**: Double-redemption is impossible. Uses atomic conditional updates (`{ _id, currentPoints: { $gte: pointsCost } }`) combined with MongoDB ACID transactions.
  - *Example invariant test*: Initial balance = 500. Two concurrent requests attempt to redeem 400 points simultaneously. **Exactly ONE succeeds, ONE fails with `INSUFFICIENT_POINTS`**. Final balance is strictly **100**, never **-300**.
- **Immutable Ledger Audit Trail**: Every point change (`EARN`, `REDEEM`, `ADJUSTMENT`) writes an immutable `PointTransaction` record with `balanceAfter` and reference links.
- **Cryptographic Balance Reconciliation**: Built-in verification engine checks that for every member:
  $$\text{Stored Balance} = \sum (\text{EARN} + \text{positive ADJUSTMENT}) - \sum (\text{REDEEM} + \text{negative ADJUSTMENT})$$

---

## 🚀 Key Features

### ☕ High-Velocity POS Counter
- **Instant Phone Lookup**: Normalized phone search (+91 / standard 10-digit) with partial matching and auto-focus.
- **Selected Member Hero Card**: VIP tier badge, real-time available points, lifetime earned points, and tier progress bar.
- **Record Purchase Modal**:
  - Live estimation preview as the cashier types:
    - `Purchase: ₹500`
    - `Tier: Gold (1.50x multiplier)`
    - `Points Earned: +75 pts`
    - `Projected Balance: 1,240 → 1,315 pts`
  - Single-click confirmation with immediate TanStack Query cache invalidation (no page refresh required).
- **Redeem Reward Modal**:
  - Visual catalog of active rewards with points costs.
  - Disabled states if member balance is insufficient.
  - Live deduction preview: `Member balance: 1,240 | Cost: 300 | Remaining: 940`.
  - Two-step confirmation dialog to prevent accidental cashier clicks.

### 📊 Management Dashboard
- **KPI Metrics**: Total Members, Total Points Issued, Total Points Redeemed, Today's Purchases (₹ sales & transaction count), Today's Redemptions.
- **Visual Analytics with Recharts**:
  - Revenue & Purchases volume trend over 14 days (Area chart).
  - Points Earned vs. Redeemed comparison (Bar chart).
  - Member distribution by loyalty tier (Donut chart).
- **Live points activity stream**: Real-time ticker of latest ledger transactions.

### 👥 Member Directory & Profiles
- Searchable directory table with phone, tier filter, current points, lifetime points, and joined date.
- Quick Register Member dialog with real-time phone normalization preview.
- Member Detail View: Tier progress bar, lifetime stats, Points Ledger timeline, Purchase history, and Redemption history.

### 📜 Immutable Points Ledger
- Comprehensive audit trail table with `Before → After` balance progression chips.
- Filter by transaction type (`EARN`, `REDEEM`, `ADJUSTMENT`) and cashier attribution.

### ⚙️ Admin & Audit Control
- **Ledger Balance Reconciliation Scanner**: One-click system-wide audit report comparing stored balance against ledger sum for all members.
- **Manual Points Adjustment Modal**: Admin-only points credit/debit with mandatory audit reason logging.
- **Tier Configuration Editor**: Configure lifetime qualification points thresholds and multipliers for Regular, Silver, and Gold tiers.
- **Staff User Management**: View active cashier and barista accounts.

---

## 🏗️ Architecture & Monorepo Structure

```
aurigait/
├── client/                     # Frontend (React 18, Vite, TypeScript, Tailwind CSS, TanStack Query)
│   ├── src/
│   │   ├── api/client.ts       # Typed Axios API client with JWT Bearer interceptor
│   │   ├── components/         # Reusable UI (Modal, Badge, Toast, Layout Shell, Sidebar, Header)
│   │   ├── context/            # AuthContext (state, login, logout, roles)
│   │   ├── features/
│   │   │   ├── counter/        # Fast POS counter screen & action modals
│   │   │   ├── dashboard/      # KPI metrics & Recharts visualizations
│   │   │   ├── members/        # Directory table, detail profile, registration modal
│   │   │   ├── purchases/      # Purchases log table
│   │   │   ├── redemption/     # Counter rewards picker
│   │   │   ├── rewards/        # Rewards catalog & admin CRUD modal
│   │   │   ├── transactions/   # Immutable ledger viewer
│   │   │   └── admin/          # Balance reconciliation & manual adjustment
│   │   ├── types/              # TypeScript interfaces matching backend models
│   │   ├── App.tsx             # Root router & layout
│   │   ├── main.tsx
│   │   └── index.css           # Tailwind base + custom scrollbars & glow utilities
│   ├── package.json
│   ├── tailwind.config.js
│   └── vite.config.ts
├── server/                     # Backend (Node.js, Express, TypeScript, Mongoose, Zod, Vitest)
│   ├── src/
│   │   ├── config/             # DB connection, environment validation, default tier rules
│   │   ├── controllers/        # Express handlers (auth, members, purchases, rewards, etc.)
│   │   ├── middleware/         # Auth (JWT + roles), errorHandler, validate, rateLimiting
│   │   ├── models/             # Mongoose schemas with indexes, constraints, and immutability
│   │   ├── routes/             # REST API routers
│   │   ├── services/
│   │   │   ├── loyalty/        # Central TierService (qualifying thresholds, multiplier logic)
│   │   │   ├── points/         # Transactional PointsService (purchase earn & adjustments)
│   │   │   ├── redemption/     # Transactional RedemptionService (atomic concurrency guard)
│   │   │   └── audit/          # ReconciliationService (ledger parity audits)
│   │   ├── validators/         # Zod schemas for request bodies and queries
│   │   ├── utils/              # Phone normalizer (+91 / E.164), error classes, response formatters
│   │   ├── scripts/seed.ts     # Database seed script (admin, staff, 20 members, rewards, ledger)
│   │   ├── tests/              # Vitest + Supertest automated points engine & concurrency tests
│   │   ├── app.ts              # Express application configuration
│   │   └── server.ts           # Server bootstrap
│   ├── package.json
│   ├── tsconfig.json
│   └── vitest.config.ts
├── package.json                # Monorepo root with concurrent scripts
├── .env.example
├── .gitignore
└── README.md
```

---

## 🛠️ Tech Stack

| Layer | Technologies |
|---|---|
| **Frontend** | React 18, TypeScript, Vite, Tailwind CSS, TanStack Query v5, React Hook Form, Zod, Lucide React, Recharts, Axios |
| **Backend** | Node.js, Express, TypeScript, Mongoose, Zod, JWT (`jsonwebtoken`), `bcrypt`, `helmet`, `cors`, `cookie-parser`, `express-rate-limit` |
| **Database** | MongoDB 6.0+ (ACID transactions, atomic conditional queries, proper indexes) |
| **Testing** | Vitest, Supertest, Concurrency test suite |

---

## ⚙️ Prerequisites & Environment Setup

### 1. Prerequisites
- **Node.js**: v18.0.0 or higher (`node -v`)
- **npm**: v9.0.0 or higher (`npm -v`)
- **MongoDB**: v6.0+ running locally or a remote MongoDB Atlas connection string.

### 2. Configure Environment Variables
Copy `.env.example` to `.env` in the workspace root:

```bash
cp .env.example .env
```

Default `.env` configuration:
```env
# Backend Server Configuration
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/cafe_rewards?directConnection=true
JWT_SECRET=super_secret_cafe_jwt_key_development_2026_secure
CLIENT_URL=http://localhost:5173
NODE_ENV=development

# Frontend Client Configuration (Vite)
VITE_API_URL=http://localhost:5000/api
```

---

## 📦 Installation & Seeding

### 1. Install All Dependencies (Monorepo)
```bash
npm install
```

### 2. Seed Development Database
Seeds 1 Admin user, 2 Staff users, 20 Members across all tiers, 7 Rewards, historical Purchases, Redemptions, and matching Points Ledger records:
```bash
npm run seed
```

### 🔑 Pre-Configured Development Credentials
| Role | Email | Password | Permissions |
|---|---|---|---|
| **Admin** | `admin@caferewards.com` | `Admin@123` | Full access, rewards CRUD, tier config, manual adjustment, reconciliation audits |
| **Staff 1 (Barista)** | `staff1@caferewards.com` | `Staff@123` | Counter POS, search members, record purchases, redeem rewards, view ledger |
| **Staff 2 (Cashier)** | `staff2@caferewards.com` | `Staff@123` | Counter POS, search members, record purchases, redeem rewards, view ledger |

---

## 🚀 Running the Application

### Option A: Run Full-Stack Concurrently (Recommended)
```bash
npm run dev
```
Starts both the Express API server on `http://localhost:5000` and the Vite frontend on `http://localhost:5173`.

### Option B: Run Workspaces Individually
```bash
# Terminal 1: Start Backend
npm run dev --workspace=server

# Terminal 2: Start Frontend
npm run dev --workspace=client
```

### Option C: Production Build
```bash
# Build both server and client bundles
npm run build

# Start production server
npm run start --workspace=server
```

---

## 🐞 Debugging Guide

### 1. Debugging the Backend (Node / Express)

#### Using VS Code / IDE Debugger
Add this configuration to `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Server (tsx)",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["tsx", "src/server.ts"],
      "cwd": "${workspaceFolder}/server",
      "envFile": "${workspaceFolder}/.env",
      "console": "integratedTerminal"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Vitest Tests",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "test", "--workspace=server"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

#### Node Inspector Mode
Run the backend with inspector listening on port 9229:
```bash
cd server
npx tsx --inspect src/server.ts
```
Open `chrome://inspect` in Chrome and click **Inspect**.

#### Health Check
Verify API availability directly:
```bash
curl http://localhost:5000/api/health
```
Expected response:
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "service": "cafe-rewards-backend"
  }
}
```

### 2. Debugging the Frontend (React / Vite)
- **Vite Proxy**: In development, requests to `/api/*` are automatically proxied to `http://localhost:5000` via `client/vite.config.ts`. Inspect the **Network** tab in browser DevTools to verify headers and payloads.
- **Auth Token**: The JWT token is saved in `localStorage.getItem('token')` and in an HTTP-only cookie.
- **TanStack Query DevTools**: Open the React inspector or browser console to see queries like `['members-search', searchQuery]` and mutation triggers.

### 3. Debugging MongoDB Transactions & Concurrency
- In `server/src/config/db.ts`, the backend automatically detects if the connected MongoDB instance is a replica set or standalone:
  - If replica set: executes with multi-document ACID sessions (`session.startTransaction()`).
  - If standalone: executes atomic conditional queries (`{ _id, currentPoints: { $gte: cost } }`), ensuring safety without throwing transaction errors.
- Run the dedicated concurrency test directly to watch race-condition handling:
  ```bash
  npm run test --workspace=server -- -t "CONCURRENCY"
  ```

---

## 🧪 Automated Testing & Concurrency Validation

Run the complete test suite:
```bash
npm run test
```

### Validated Test Scenarios (15/15 Passed):
1. **Regular member earn rate**: ₹500 purchase earns 50 points (1.0x).
2. **Silver member earn rate**: ₹500 purchase earns 62 points (1.25x).
3. **Gold member earn rate**: ₹500 purchase earns 75 points (1.50x).
4. **Tier upgrade thresholds**: Upgrades to Silver at 1,000 lifetime points and Gold at 5,000 points.
5. **Successful redemption**: Deducts points and creates matching redemption + ledger records.
6. **Insufficient points guard**: Returns `INSUFFICIENT_POINTS` error (400) and aborts.
7. **Negative balance prevention**: Points balance can never become negative.
8. **Purchase ledger immutability**: Records `EARN` ledger entry with exact `balanceAfter`.
9. **Redemption ledger immutability**: Records `REDEEM` ledger entry with exact `balanceAfter`.
10. **Clean rollback**: Inactive rewards or errors abort transaction without state modification.
11. **Duplicate phone check**: Enforces unique normalized phone numbers (409 Conflict).
12. **🔥 CRITICAL CONCURRENCY TEST**:
    - Member starts with 500 points.
    - Two simultaneous calls attempt to redeem 400 points.
    - **Exactly 1 succeeds, 1 fails with `INSUFFICIENT_POINTS`**.
    - Final balance is **100**, never **-300**!
13. **Sequential progression**: Balance progression matches ledger values sequentially.
14. **Cumulative consistency**: Multiple purchases accumulate balance and lifetime points accurately.
15. **Reconciliation parity**: `ReconciliationService` verifies that ledger sum matches stored points 100%.

---

## 📖 Complete API Endpoints Reference

All requests and responses use standard JSON formatting:
- **Success**: `{ "success": true, "data": { ... } }`
- **Error**: `{ "success": false, "error": { "code": "...", "message": "..." } }`

### 1. Authentication
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/auth/login` | Public | Sign in with email & password. Sets cookie & returns JWT. |
| `POST` | `/api/auth/logout` | Public | Clears session cookie. |
| `GET` | `/api/auth/me` | Authenticated | Returns current authenticated staff profile. |

#### `POST /api/auth/login`
```json
// Request Body:
{
  "email": "admin@caferewards.com",
  "password": "Admin@123"
}

// Response (200 OK):
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6...",
    "user": {
      "id": "651234...",
      "name": "Aarav Sharma (Admin)",
      "email": "admin@caferewards.com",
      "role": "ADMIN"
    }
  }
}
```

---

### 2. Members Management
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/members` | Staff, Admin | Register a new member. |
| `GET` | `/api/members` | Staff, Admin | List members with search query (`?query=...&tier=...&page=1`). |
| `GET` | `/api/members/phone/:phone` | Staff, Admin | Quick lookup by normalized or partial phone number. |
| `GET` | `/api/members/:id` | Staff, Admin | Full member profile with tier progression stats. |
| `GET` | `/api/members/:id/transactions` | Staff, Admin | Paginated points ledger entries for a member. |
| `GET` | `/api/members/:id/purchases` | Staff, Admin | Historical purchases recorded for a member. |
| `GET` | `/api/members/:id/redemptions` | Staff, Admin | Historical redemptions claimed by a member. |

#### `POST /api/members`
```json
// Request Body:
{
  "name": "Ashiv Nagar",
  "phone": "+91 98765 43210",
  "email": "ashiv.nagar@example.com"
}

// Response (201 Created):
{
  "success": true,
  "data": {
    "id": "651234...",
    "name": "Ashiv Nagar",
    "phone": "+91 98765 43210",
    "normalizedPhone": "+919876543210",
    "tier": "REGULAR",
    "currentPoints": 0,
    "lifetimeEarnedPoints": 0,
    "progression": {
      "currentTier": "REGULAR",
      "nextTier": "SILVER",
      "pointsToNextTier": 1000,
      "progressPercent": 0
    }
  }
}
```

---

### 3. Purchases & Points Earning
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/purchases` | Staff, Admin | Record café purchase & award loyalty points. |
| `GET` | `/api/purchases` | Staff, Admin | List recorded purchases with cashier attribution. |
| `GET` | `/api/purchases/:id` | Staff, Admin | Single purchase details. |

#### `POST /api/purchases`
```json
// Request Body:
{
  "memberId": "651234...",
  "amount": 500
}

// Response (201 Created):
{
  "success": true,
  "data": {
    "message": "Purchase recorded successfully",
    "pointsEarned": 75,
    "previousBalance": 1240,
    "newBalance": 1315,
    "tierBefore": "GOLD",
    "tierAfter": "GOLD",
    "tierUpgraded": false,
    "calculation": {
      "amount": 500,
      "basePoints": 50,
      "multiplier": 1.5,
      "earnedPoints": 75
    }
  }
}
```

---

### 4. Rewards Catalog
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/rewards` | Staff, Admin | List active rewards (`?active=true` or `all`). |
| `GET` | `/api/rewards/:id` | Staff, Admin | Single reward details. |
| `POST` | `/api/rewards` | Admin Only | Create a new reward item. |
| `PATCH` | `/api/rewards/:id` | Admin Only | Update reward name, points cost, or category. |
| `DELETE` | `/api/rewards/:id` | Admin Only | Soft-deactivate a reward. |

#### `POST /api/rewards` (Admin Only)
```json
// Request Body:
{
  "name": "Belgian Chocolate Waffle",
  "description": "Warm Belgian waffle with melted dark chocolate and vanilla bean gelato.",
  "pointsCost": 300,
  "category": "Dessert"
}
```

---

### 5. Reward Redemptions
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/redemptions` | Staff, Admin | Redeem reward for member (atomic balance deduction). |
| `GET` | `/api/redemptions` | Staff, Admin | List past redemptions. |
| `GET` | `/api/redemptions/:id` | Staff, Admin | Single redemption details. |

#### `POST /api/redemptions`
```json
// Request Body:
{
  "memberId": "651234...",
  "rewardId": "659876..."
}

// Response (201 Created):
{
  "success": true,
  "data": {
    "message": "Reward redeemed successfully",
    "pointsSpent": 100,
    "previousBalance": 1315,
    "newBalance": 1215,
    "reward": {
      "name": "Cappuccino / Latte",
      "pointsCost": 100
    }
  }
}
```

---

### 6. Points Ledger (Audit Trail)
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/transactions` | Staff, Admin | Full ledger audit trail (`?type=EARN|REDEEM|ADJUSTMENT&page=1`). |
| `GET` | `/api/transactions/member/:memberId` | Staff, Admin | All ledger transactions for a single member. |

---

### 7. Loyalty Tier Configurations
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/tiers` | Staff, Admin | View configured tier thresholds and earning multipliers. |
| `PATCH` | `/api/tiers/:id` | Admin Only | Update tier multiplier or qualifying points threshold. |

---

### 8. Management Dashboard & Analytics
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `GET` | `/api/dashboard/stats` | Staff, Admin | Returns KPI cards, 14-day sales/points trends, and live activity feed. |

---

### 9. Admin & Balance Reconciliation
| Method | Endpoint | Access | Description |
|---|---|---|---|
| `POST` | `/api/admin/adjust-points` | Admin Only | Manual point adjustment with mandatory audit reason. |
| `GET` | `/api/admin/reconciliation/member/:id` | Admin Only | Verify member's balance parity against ledger sum. |
| `GET` | `/api/admin/reconciliation/all` | Admin Only | System-wide integrity scan across all members. |
| `GET` | `/api/admin/staff` | Admin Only | List authorized staff and cashier accounts. |

#### `POST /api/admin/adjust-points`
```json
// Request Body:
{
  "memberId": "651234...",
  "points": 50,
  "reason": "Customer service courtesy compensation for delayed latte"
}
```

#### `GET /api/admin/reconciliation/member/:id`
```json
// Response:
{
  "success": true,
  "data": {
    "storedBalance": 1215,
    "calculatedBalance": 1215,
    "difference": 0,
    "consistent": true,
    "ledgerEntriesCount": 6,
    "auditBreakdown": {
      "totalEarned": 1315,
      "totalRedeemed": 100,
      "totalAdjusted": 0
    }
  }
}
```

---

## ❓ Troubleshooting & FAQs

### Q: Why did a redemption fail with `INSUFFICIENT_POINTS`?
**A**: The member's balance is lower than the reward's points cost, or a competing concurrent request already spent those points. Verify member balance via `GET /api/members/:id`.

### Q: How do I resolve `E11000 duplicate key error collection: members index: normalizedPhone`?
**A**: A member with this phone number already exists. The system standardizes numbers to international format (e.g. `9876543210` becomes `+919876543210`). Use phone lookup to locate the existing member profile.

### Q: How does the application handle standalone MongoDB vs. Replica Sets?
**A**: If a MongoDB replica set is detected, multi-document ACID transactions (`session.startTransaction()`) are automatically used. If running standalone, document-level atomic conditional queries are used, guaranteeing race-condition safety without requiring configuration changes.

---

## 📄 License
MIT License. Created for the Café Rewards Counter project.

   
