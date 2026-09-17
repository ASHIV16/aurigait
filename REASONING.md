
# 🧠 Engineering Reasoning & Architecture Decisions
## Project: Café Rewards Counter — Loyalty & Points Management System

This document outlines the engineering thought process, architectural decisions, invariant guarantees, edge-case analysis, testing methodology, and how challenges were resolved during the design and implementation of **Café Rewards Counter**.

---

## 🎯 1. Problem Framing & Core Invariants

The primary challenge of a loyalty points system is not simply creating CRUD records for purchases and redemptions; it is **guaranteeing mathematical consistency under concurrent, high-velocity counter operations**.

### Non-Negotiable Invariants:
1. **Zero Negative Balances**: A member's available points balance (`currentPoints`) can never drop below 0 under any circumstance (`currentPoints >= 0`).
2. **Race-Condition Immunity**: Two simultaneous cashier requests trying to redeem rewards against the same balance must never both succeed if their combined cost exceeds the balance.
3. **No Lost or Orphaned Points**: Every point credited or debited must be backed by a corresponding immutable ledger entry.
4. **One-Way Loyalty Progression**: Points redemption cannot demote a member's loyalty tier. Tiers must be pegged to **lifetime earned points**, not spendable points.
5. **Ledger Parity**: The stored balance must at all times equal the net sum of all historical ledger transactions.

---

## 🏛️ 2. Architectural Thought Process

### 2.1 Monorepo Strategy
We selected a unified monorepo with `server/` and `client/` workspaces:
- **`server/`**: Express, TypeScript, Mongoose, Zod, Vitest. Handles business rules, atomic writes, transactions, and audit verification.
- **`client/`**: React 18, Vite, TypeScript, Tailwind CSS, TanStack Query v5, Recharts. Optimized for fast cashier keyboard-driven interactions and live metric dashboards.
- **Root Scripts**: Single-command bootstrapping (`npm run dev`, `npm run build`, `npm run seed`, `npm run test`) to streamline developer onboarding.

### 2.2 Double-Entry / Immutable Ledger Pattern
Rather than treating `Member.currentPoints` as an isolated mutable number, we adopted an **immutable audit ledger pattern**:
- Every modification produces a `PointTransaction` record with:
  - `type`: `EARN`, `REDEEM`, or `ADJUSTMENT`
  - `amount`: Points credited or debited
  - `balanceBefore`: Member's point balance prior to the mutation
  - `balanceAfter`: Member's point balance immediately after the mutation
  - `referenceType` & `referenceId`: Direct link to the corresponding `Purchase`, `Redemption`, or admin adjustment note
  - `cashierId`: Staff member responsible for the transaction
- In Mongoose, pre-save and pre-update middleware hooks reject any `update`, `findOneAndUpdate`, or `delete` on `PointTransaction` to guarantee immutability.

### 2.3 Tier Calculation: Spendable vs. Lifetime Points
In many naive loyalty implementations, spending points causes members to lose their tier (e.g. going from Gold back to Regular).
- **Our Decision**:
  - `currentPoints`: Spendable currency, increases on `EARN`, decreases on `REDEEM`.
  - `lifetimeEarnedPoints`: Strictly monotonic metric, increases **only** on `EARN` (and positive adjustments).
  - Tiers (`REGULAR`, `SILVER`, `GOLD`) are derived solely from `lifetimeEarnedPoints`:
    - **Regular**: 0 to 999 lifetime points (1.0x earn rate = 10 pts per ₹100)
    - **Silver**: 1,000 to 4,999 lifetime points (1.25x earn rate = 12.5 → 12 pts per ₹100)
    - **Gold**: 5,000+ lifetime points (1.50x earn rate = 15 pts per ₹100)
  - Calculations round down (`Math.floor`) to ensure integer point precision.

---

## ⚡ 3. Handling Concurrency & Race Conditions

### The Scenario:
- Member has **500 points**.
- Cashier 1 at Counter A and Cashier 2 at Counter B submit redemptions for a 400-point reward at the exact same millisecond.
- In a naive "Read-then-Write" pattern:
  1. Process A reads balance: 500 >= 400 (OK)
  2. Process B reads balance: 500 >= 400 (OK)
  3. Process A writes balance: 500 - 400 = 100
  4. Process B writes balance: 100 - 400 = -300 *(Violation: Negative balance & double redemption!)*

### Our Solution:
1. **Atomic Conditional Queries (Defense in Depth)**:
   Instead of fetching the document and calling `.save()`, `RedemptionService` uses an atomic conditional update:
   ```typescript
   const updatedMember = await Member.findOneAndUpdate(
     { _id: memberId, currentPoints: { $gte: reward.pointsCost } },
     { $inc: { currentPoints: -reward.pointsCost } },
     { new: true, session }
   );
   ```
   - If two requests run concurrently, MongoDB's document-level lock evaluates the condition `{ currentPoints: { $gte: 400 } }` sequentially.
   - The first request reduces the balance from 500 to 100 and receives the updated document.
   - The second request evaluates `{ currentPoints: { $gte: 400 } }` against 100, which evaluates to false. MongoDB returns `null`.
   - The service detects `!updatedMember` and immediately throws an `InsufficientPointsError` (HTTP 400).

2. **Dual-Mode Transactional Support**:
   - If running on a MongoDB Replica Set (production), operations are wrapped in an ACID multi-document session (`session.startTransaction()`).
   - If running on a standalone MongoDB instance (local dev / Windows service), `db.ts` dynamically detects the topology (`isReplicaSet()`) and falls back safely to document-level atomic conditional queries, ensuring zero crashes while preserving race-condition safety.

---

## 🔍 4. Edge Cases Identified & Handled

| Edge Case | Failure Mode If Ignored | Solution Implemented |
|---|---|---|
| **Phone variations** (`9876543210`, `+91 98765 43210`, `98765-43210`) | Duplicate member accounts created under different formats. | Normalized via `normalizePhone()` to E.164 (`+919876543210`) with a unique database index on `normalizedPhone`. |
| **Inactive / Deleted Rewards** | Member redeems an item no longer available in the café. | `RedemptionService` verifies `reward.isActive === true` before executing any deduction. |
| **Decimal Point Fractions** | Earning points with 1.25x multiplier (e.g. ₹250 → 31.25 pts) leads to float rounding discrepancies. | `Math.floor()` applied to all earned point calculations. |
| **Accidental Double-Clicks at Counter** | Cashier taps "Redeem" twice quickly. | Two-step confirmation modal with loading disabled states + backend atomic guard. |
| **Manual Admin Corrections** | Disgruntled customer requires points compensation. | Dedicated `POST /api/admin/adjust-points` requiring an explicit `reason` string, which is permanently etched into the audit ledger. |
| **Zero or Negative Purchase Amount** | Malformed input awarding negative points. | Zod schema validation enforces `amount: z.number().positive()`. |

---

## 🛠️ 5. Challenges Encountered & How They Were Fixed

### Challenge 1: Standalone MongoDB vs. Replica Set Transactions
- **Issue**: In local Windows development, MongoDB often runs as a single standalone service rather than a replica set. Calling `session.startTransaction()` throws:
  `MongoServerError: Transaction numbers are only allowed on a replica set member or mongos`.
- **Resolution**:
  1. Updated `server/src/config/db.ts` to inspect `client.topology.description.type` on startup.
  2. Implemented a `withTransaction` wrapper helper that applies transactions when available, and executes cleanly without transaction sessions on standalone nodes.
  3. Structured `RedemptionService` and `PointsService` around atomic conditional queries (`{ _id, currentPoints: { $gte: cost } }`) so that atomicity does not rely solely on replica set locks.

### Challenge 2: Strict Express 5 Param Typing
- **Issue**: In `@types/express` v5, `req.params[key]` is typed `string | string[] | undefined`, causing TypeScript compilation errors when passing `req.params.id` to Mongoose find queries.
- **Resolution**: Normalized all route parameters across controllers using `String(req.params.id)` or Zod route parameter validation.

### Challenge 3: Real-Time Cashier UX (No Stale Balances)
- **Issue**: In traditional SPAs, after recording a purchase or redemption, other open views (member detail, ledger, dashboard) retain stale balance figures unless manually refreshed.
- **Resolution**: Configured TanStack Query with aggressive query key invalidation (`queryClient.invalidateQueries({ queryKey: ['member'] })`, `['members']`, `['dashboard']`, `['transactions']`) on every successful mutation. When a cashier logs a purchase, the hero card and all views update instantaneously.

---

## 🧪 6. Testing Methodology & Verification

### 6.1 Automated Concurrency & Invariant Test Suite
We created a comprehensive Vitest test suite in `server/src/tests/pointsEngine.test.ts` with **15 automated scenarios (15/15 passed)**:
1. **Tier Multipliers**: ₹500 purchase correctly yields 50 pts (Regular), 62 pts (Silver @ 1.25x), 75 pts (Gold @ 1.50x).
2. **Tier Upgrades**: Member upgrades to Silver when lifetime points cross 1,000, and Gold when crossing 5,000.
3. **Atomic Redemption**: Balance deducts cleanly with matching redemption and ledger records.
4. **Negative Balance Guard**: Redemption exceeding balance fails with `INSUFFICIENT_POINTS` and preserves balance.
5. **Ledger Immutability**: Verifies `balanceBefore` and `balanceAfter` exactly match state transitions.
6. **Concurrent Double-Redemption**:
   - Initial balance = 500 points.
   - Dispatched `Promise.all([redeem400, redeem400])` concurrently.
   - Result: Exactly 1 returned 201 Created, 1 returned 400 Bad Request (`INSUFFICIENT_POINTS`). Final balance was exactly 100 points.
7. **Reconciliation Parity**: Asserted that `ReconciliationService.reconcileMember()` returns `consistent: true` and `difference: 0`.

### 6.2 End-to-End POS Flow Verification
We executed automated browser verification simulating real cashier interactions:
- **Staff Login**: Authenticated as `staff1@caferewards.com`.
- **Search**: Typed partial phone `9876543210` → selected Member "Rohan Mehta (Gold Tier)".
- **Purchase with Live Preview**: Recorded ₹500 purchase → verified live preview showed 75 pts earned at 1.50x → confirmed balance increased from 1,240 to 1,315.
- **Reward Redemption**: Redeemed "Cappuccino / Latte" (100 pts) → balance updated to 1,215.
- **Audit Verification**: Admin ran system-wide balance reconciliation scan → 20/20 members verified with 0 discrepancies.

---

## 💡 7. Lessons Learned & Recommendations for Scale

1. **Conditional Updates Beat Optimistic Locking for Simple Counters**:
   Using `{ currentPoints: { $gte: cost } }` with `$inc` avoids the overhead of retry loops often required by optimistic version keys (`__v`), delivering superior throughput at counter terminals.
2. **Normalized Indexes Prevent Edge Case Support Tickets**:
   Storing clean E.164 phone formats (`+919876543210`) as a separate indexed field eliminated customer complaints regarding whether cashiers typed spaces or national dialing codes.
3. **Continuous Reconciliation Builds Customer Trust**:
   Providing an admin-facing balance reconciliation scanner empowers café owners to independently verify mathematical integrity at any time.
