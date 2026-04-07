# WebBill — Precision Hospitality Operating System

> **A high-performance, full-stack restaurant management SaaS platform** built with Next.js 14, MongoDB Atlas, and Prisma — designed for real-world food & beverage operations.

**Developed by [WebCultivation](https://github.com/Dhruvil1308)**

---

## 📋 Table of Contents

- [Overview](#overview)
- [Live Demo](#live-demo)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Data Models](#data-models)
- [Application Modules](#application-modules)
  - [1. Landing Page](#1-landing-page--home)
  - [2. Waiter Panel](#2-waiter-panel--waiter)
  - [3. Kitchen Display System (KDS)](#3-kitchen-display-system--kitchen)
  - [4. Billing Counter](#4-billing-counter--billing)
  - [5. Digital Receipt](#5-digital-receipt--receiptorderid)
- [API Reference](#api-reference)
- [State Management](#state-management)
- [Design System](#design-system)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Environment Variables](#environment-variables)
  - [Database Setup & Seeding](#database-setup--seeding)
  - [Running Locally](#running-locally)
- [Deployment (Vercel)](#deployment-vercel)
- [Configuration](#configuration)
- [GST & Billing Logic](#gst--billing-logic)
- [Seed Data](#seed-data)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

**WebBill** is a production-grade, cloud-native restaurant and hotel management system that replaces paper-based workflows with a unified digital ecosystem. It is purpose-built for the Indian F&B industry with **GST-compliant billing**, multi-payment-method support, and WhatsApp-based digital receipt sharing.

The platform operates as three distinct operational terminals that communicate in real time via a shared MongoDB Atlas database:

| Terminal | Role | Description |
|---|---|---|
| 🧑‍🍳 **Waiter Panel** | Order Taking | Waiter selects a table, browses the menu, and sends orders to the kitchen |
| 👨‍🍳 **Kitchen Display (KDS)** | Food Production | Kitchen staff sees live incoming orders in a Kanban board and advances their status |
| 🧾 **Billing Counter** | Settlement | Cashier views all active tables, settles bills, and generates thermal receipts or WhatsApp links |

All terminals operate on a **polling-based real-time sync** (2–10 second intervals), ensuring zero external WebSocket dependencies and maximum deployment compatibility.

---

## Live Demo

> Deployed on **Vercel** with **MongoDB Atlas** as the database backend.

```
https://webbill.vercel.app
```

*(Replace with your actual deployment URL)*

---

## Key Features

### 🏨 Hotel / Multi-Tenant Ready
- Each property is identified by a **Hotel** record with a unique `slug`
- All data (tables, menus, orders, transactions) is scoped per `hotelId`
- Configurable via environment variables (`NEXT_PUBLIC_HOTEL_ID`, `NEXT_PUBLIC_HOTEL_NAME`)
- Architecture is prepared for full multi-tenancy expansion

### 📋 Waiter Panel
- **Mobile-first** UI optimized for smartphones and tablets
- **Live Table Status Grid** — color-coded (Free / Active / Waiting Bill) with real-time sync every 3 seconds
- **Category-filtered Menu Browser** — browses all menu items organized by food category
- **Persistent Cart** — backed by Zustand + `localStorage`, survives page refreshes
- **Optimistic Order Submission** — cart clears instantly for a responsive UX, rolls back on failure
- **Live Order Volume & Performance Score** analytics on the dashboard

### 🍳 Kitchen Display System (KDS)
- **Dual-column Kanban Board** — `Pending` and `In Preparation` columns
- **Polling every 2 seconds** for near-real-time order visibility
- **Optimistic Status Updates** — kitchen UI updates instantly while the API call runs in background
- **Delayed Order Detection** — orders open for more than 15 minutes are flagged with a pulsing "Delayed" badge
- Full rollback on failed API calls

### 🧾 Billing Counter
- **Revenue Dashboard** — today's revenue, active tables, pending bills, order completion rate
- **Payment Breakdown** — Cash / UPI / Card revenue split for the day
- **Recent Transactions Table** — last 10 settlements with table, order ID, method, and amount
- **Settlement Bottom Sheet** — slides up with full itemized bill, GST breakdown, and payment selection
- **Thermal PDF Generation** using `jsPDF + jspdf-autotable` in 80mm thermal receipt format
- **WhatsApp Receipt Sharing** — sends a public receipt URL via WhatsApp deep link
- **Discount Support** — discounts are distributed proportionally across multiple orders per table

### 🧾 Digital Public Receipt
- **Shareable Receipt URL** — `/receipt/{orderId}` is a public page any customer can access
- Displays a pixel-perfect **thermal receipt mockup** with jagged edge CSS effects
- **Download as PDF** (E-Receipt) or **Direct Print** via browser
- Thermal-format PDF (80mm × 200mm) with line items, GST, and grand total
- Print CSS media query — hides all UI and shows only the receipt on print

### 💰 GST-Compliant Billing
- **5% GST** applied automatically on all transactions
- GST amount stored separately per transaction in the database
- Receipt shows `Subtotal`, `GST (5%)`, and `Grand Total` separately
- Revenue API aggregates amounts including GST

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Next.js 14 App Router                    │
│                      (Deployed on Vercel)                       │
├──────────────┬──────────────────┬──────────────────────────────┤
│  /waiter     │  /kitchen        │  /billing  │  /receipt/[id]   │
│  Waiter UI   │  KDS Kanban      │  Cashier   │  Public Receipt  │
│              │  Board           │  Terminal  │                   │
└──────────────┴──────────────────┴──────────────────────────────┤
│                     Next.js API Routes (/api/*)                 │
│  orders/  |  menu/  |  tables/  |  billing/settle  | revenue   │
└─────────────────────────────────────────────────────────────────┤
│         Prisma ORM (prisma-client-js)                           │
└─────────────────────────────────────────────────────────────────┤
│              MongoDB Atlas (Cloud Database)                      │
│   Collections: hotels | users | tables | menuCategories |       │
│   menuItems | orders | orderItems | transactions                 │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Waiter selects table → browses menu → adds to cart (Zustand)
    ↓ [Pass Order]
POST /api/orders → Prisma → MongoDB (status: PENDING)
    ↓ [Kitchen polls every 2s]
GET /api/orders?hotelId=... → KDS shows order in PENDING column
    ↓ [Chef clicks "Start Prep"]
PATCH /api/orders/{id} → status: PREPARING
    ↓ [Chef clicks "Mark Done"]
PATCH /api/orders/{id} → status: READY
    ↓ [Billing polls every 10s]
GET /api/orders → Billing shows table as "WAITING BILL"
    ↓ [Cashier clicks "Settle Bill"]
POST /api/billing/settle → creates Transaction, marks orders COMPLETED
    ↓ [Optional]
WhatsApp deep link → customer opens /receipt/{orderId} → downloads PDF
```

---

## Tech Stack

| Category | Technology | Version |
|---|---|---|
| **Framework** | Next.js | 14.1.4 |
| **Language** | TypeScript | ^5 |
| **Database** | MongoDB Atlas | Cloud |
| **ORM** | Prisma | ^5.12.1 |
| **Authentication** | NextAuth.js | ^4.24.13 |
| **State Management** | Zustand | ^4.5.2 |
| **Data Fetching** | TanStack React Query | ^5.96.2 |
| **Animations** | Framer Motion | ^11.0.24 |
| **PDF Generation** | jsPDF + jspdf-autotable | ^4.2.1 / ^5.0.7 |
| **Styling** | Tailwind CSS | ^3.4.1 |
| **Forms** | React Hook Form | ^7.51.2 |
| **Validation** | Zod | ^3.22.4 |
| **Icons** | Lucide React | ^0.363.0 |
| **Date Utilities** | date-fns | ^4.1.0 |
| **HTTP Client** | Axios | ^1.14.0 |
| **UI Components** | Custom (no external UI lib) | — |
| **Fonts** | Inter + Outfit (Google Fonts) | — |
| **Deployment** | Vercel | Serverless |

---

## Data Models

All models are defined in `prisma/schema.prisma` and backed by MongoDB Atlas.

### Hotel
The root entity. All other data is scoped to a `Hotel`.

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated MongoDB ObjectId |
| `name` | String | Display name (e.g., "Saffron Bay") |
| `slug` | String (unique) | URL-safe identifier (e.g., "saffron-bay") |
| `address` | String? | Optional property address |
| `phone` | String? | Contact number |
| `subscriptionExpiry` | DateTime | SaaS subscription expiry date |
| `isActive` | Boolean | Whether the hotel is active |

### User
Staff accounts for the hotel (Waiters, Managers, etc.).

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `name` | String | Full name |
| `email` | String (unique) | Login email |
| `password` | String | bcrypt-hashed password |
| `role` | String | Default: `"WAITER"` |
| `hotelId` | ObjectId | FK → Hotel |

### Table
Dining tables at the property.

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `number` | String | Display number (e.g., "T1", "T12") |
| `capacity` | Int | Seating capacity |
| `hotelId` | ObjectId | FK → Hotel |

### MenuCategory
Organizes menu items into categories (e.g., "TEA", "PASTA", "NAAN PIZZA").

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `name` | String | Category display name |
| `hotelId` | ObjectId | FK → Hotel |

### MenuItem
Individual food/drink items.

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `name` | String | Item name |
| `price` | Float | Price in INR |
| `description` | String? | Optional description |
| `image` | String? | Optional image URL |
| `isAvailable` | Boolean | Toggles item visibility in Waiter app |
| `categoryId` | ObjectId | FK → MenuCategory |
| `hotelId` | ObjectId | FK → Hotel |

### Order
An order placed by a waiter for a specific table.

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `status` | String | `PENDING` → `PREPARING` → `READY` → `COMPLETED` |
| `totalAmount` | Float | Computed from order items (pre-GST) |
| `tableId` | ObjectId | FK → Table |
| `hotelId` | ObjectId | FK → Hotel |

### OrderItem
Line items within an order.

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `orderId` | ObjectId | FK → Order |
| `itemId` | ObjectId | FK → MenuItem |
| `quantity` | Int | Number of units ordered |
| `price` | Float | Price at time of order (snapshot) |
| `notes` | String? | Customization notes |

### Transaction
Financial record created when an order is settled.

| Field | Type | Description |
|---|---|---|
| `id` | ObjectId | Auto-generated |
| `orderId` | ObjectId (unique) | FK → Order (one-to-one) |
| `amount` | Float | Total amount collected (subtotal + GST) |
| `status` | String | `UNPAID` or `PAID` |
| `paymentMethod` | String? | `Cash`, `UPI`, or `Card` |
| `gstAmount` | Float | GST portion (5%) |
| `discount` | Float | Discount applied |
| `hotelId` | ObjectId | FK → Hotel |

---

## Application Modules

### 1. Landing Page (`/`)

The hub that links all three operational panels.

**Features:**
- Animated hero section with Framer Motion entrance animations
- Three module cards linking to Waiter Panel, KDS, and Billing Counter
- Premium gradient background using absolute-positioned blurred orbs
- "Network Live" and version status indicators
- Responsive grid layout (stacked on mobile, side-by-side on desktop)
- Brand footer with GST compliance badge

**Key Design Choices:**
- Custom `webbill-*` Tailwind color tokens for brand consistency
- `framer-motion` for sequential card entrance animations with staggered delays
- The header shows the WebCultivation branding

---

### 2. Waiter Panel (`/waiter`)

The primary mobile interface for wait staff.

**Two-view SPA Pattern:**
The waiter app uses a single-page pattern with two internal views toggled by the `currentView` state:

```typescript
type View = 'DASHBOARD' | 'MENU';
```

**DASHBOARD View:**
- **Live analytics row** showing Today's Orders (live count from DB) and Performance Score
- **Table Status Grid** — a 2–3 column grid of all tables with color-coded status:
  - 🟢 **Free** — no active orders
  - 🟠 **Active** — has `PENDING` or `PREPARING` orders
  - 🔴 **Billed** — has at least one `READY` order awaiting settlement
- Tapping any table opens the **MENU View** for that table
- Data refreshes automatically every **3 seconds** (background interval)

**MENU View:**
- Back navigation to DASHBOARD
- Shows **Active Table Items** — existing orders for this table with live production status, so waiters don't re-order what's already been sent
- **Horizontal scrollable category tabs** filter menu items
- **Item cards** with image (or letter avatar fallback), name, description, price
- **Quantity stepper** — ADD → shows `−` / count / `+` controls
- **Bottom Order Bar** (Zomato-inspired) — slides up when cart has items, shows total and "Pass Order" button
- Submitting the order:
  1. Cart is cleared immediately (optimistic UI)
  2. API call is made to `POST /api/orders`
  3. On failure, cart items are restored (rollback)

**State Management:**
- Cart is managed by `useCartStore` (Zustand + localStorage persistence)
- Order submission is handled by the `useOrder` custom hook
- Table and order data is managed by local `useState` + `useEffect` polling

---

### 3. Kitchen Display System (`/kitchen`)

A full-screen Kanban board for kitchen production staff.

**Panel Design:**
- Fixed navigation bar at the top with live terminal indicators
- Horizontally scrollable Kanban board with two columns:
  - **PENDING** — newly received orders, colored with burgundy accent
  - **IN PREPARATION** — orders being cooked, colored with amber accent
- Fixed stats bar at the bottom showing Avg Prep Time and Load Factor
- Polls every **2 seconds** for maximum responsiveness

**Order Cards:**
- Show `Table Number`, `Order ID`, `Creation Time`, and a list of `Item × Quantity`
- Orders open for more than **15 minutes** are flagged with a rotating red "Delayed" badge and a pulsing amber ring
- Action buttons advance the order status:
  - PENDING → **"Start Prep"** button → moves to PREPARING
  - PREPARING → **"Mark Done"** button → moves to READY

**Optimistic Updates:**
```typescript
// Updates local state first for instant UI feedback
setOrders(prev => prev.map(o => o.id === id ? { ...o, status: nextStatus } : o));
// Then calls the API
await fetch(`/api/orders/${id}`, { method: 'PATCH', ... });
// Rollback if API fails
fetchOrders();
```

**AnimatePresence:** Cards animate in/out smoothly when they change columns, powered by Framer Motion's `layout` and `AnimatePresence` components.

---

### 4. Billing Counter (`/billing`)

The cashier's terminal for settlement and revenue tracking.

**Dashboard Stats (from `/api/billing/revenue`):**
| Metric | Description |
|---|---|
| Today's Revenue | Sum of all paid transactions today (inc. GST) |
| Active Tables | Count of unique tables with non-completed orders |
| Pending Bills | Count of `READY` status orders |
| Completion Rate | (`completedToday` / `totalToday`) × 100 |
| Growth % | Compared to yesterday's revenue |
| Payment Breakdown | Cash/UPI/Card amounts for the day |
| Recent Transactions | Last 10 settled transactions |

**Active Tables List:**
- All non-completed orders are **grouped by table number**
- Multiple orders for the same table are merged into a single table entry
- Item quantities are consolidated if the same item appears in multiple orders
- Tables with `READY` orders display as **"WAITING BILL"** with pulsing red indicator

**Settlement Bottom Sheet:**
Tapping a table slides up a spring-animated bottom sheet containing:
1. Full **itemized breakdown** with `quantity × name → line total`
2. **Subtotal**, **GST (5%)**, and **Grand Total**
3. **Customer Phone Number** input for WhatsApp receipt
4. **Payment Method selector** — Cash / UPI / Card
5. **Print Bill** — generates PDF via `jsPDF`
6. **Settle Bill** — calls `POST /api/billing/settle` with atomic Prisma transaction

**Settlement Logic:**
- Uses `prisma.$transaction()` to atomically:
  1. Find all target orders
  2. Create a `Transaction` record for each (or update if one exists)
  3. Calculate GST and actual totals from line items
  4. Mark all orders as `COMPLETED`
- After a successful settlement, if a phone number was provided, opens WhatsApp with a pre-filled message containing the receipt URL

---

### 5. Digital Receipt (`/receipt/[orderId]`)

A shareable, customer-facing receipt page accessed via a public URL.

**Design:**
- Pixel-perfect recreation of an 80mm thermal receipt on a grey page background
- Jagged top and bottom edges simulated with CSS `radial-gradient` background patterns
- Receipt content:
  - WebBill + Hotel Name header
  - Table number, Date, Time, Bill ID (last 6 chars of MongoDB ObjectId, uppercased)
  - Items table with `Item | Qty | Total`
  - Subtotal, GST (5%), Grand Total
  - Thank You footer
- Floating action bar with **"E-Receipt PDF"** and **"Direct Print"** buttons
- PDF generation uses the same 80×200mm thermal format as the billing counter
- **Print CSS**: `@media print` styles hide all UI and show only the receipt paper

---

## API Reference

All API routes are in `app/api/` and follow the Next.js App Router convention.

### Orders

#### `GET /api/orders?hotelId=<id>`
Returns all orders for a hotel, ordered by `createdAt` descending.

**Response:** Array of orders with populated `items.item` and `table`.

#### `POST /api/orders`
Creates a new order.

**Request Body:**
```json
{
  "tableId": "ObjectId",
  "hotelId": "ObjectId",
  "items": [
    { "id": "menuItemId", "name": "Cappuccino", "price": 180, "quantity": 2 }
  ]
}
```

**Response:** The created order (HTTP 201).

#### `GET /api/orders/[id]`
Returns a single order by ID with all related data.

#### `PATCH /api/orders/[id]`
Updates an order's status.

**Request Body:**
```json
{ "status": "PREPARING" }
```

### Menu

#### `GET /api/menu?hotelId=<id>`
Returns all menu categories with their available items.

**Response:**
```json
[
  {
    "id": "...",
    "name": "TEA",
    "items": [
      { "id": "...", "name": "Regular Tea", "price": 50, "isAvailable": true }
    ]
  }
]
```

### Tables

#### `GET /api/tables?hotelId=<id>`
Returns all tables for the hotel.

### Billing

#### `POST /api/billing/settle`
Settles one or more orders atomically.

**Request Body:**
```json
{
  "orderIds": ["id1", "id2"],
  "paymentMethod": "UPI",
  "hotelId": "ObjectId",
  "discount": 0
}
```

**Response:**
```json
{ "success": true, "transactions": [...] }
```

#### `GET /api/billing/revenue?hotelId=<id>`
Returns today's complete revenue analytics.

**Response:**
```json
{
  "todayRevenue": 4250.50,
  "yesterdayRevenue": 3800.00,
  "growth": 11.9,
  "activeTables": 4,
  "pendingBills": 2,
  "completionRate": 78,
  "breakdown": { "Cash": 2000, "UPI": 1750.50, "Card": 500 },
  "recentTransactions": [...]
}
```

---

## State Management

### Cart Store (`useCartStore`)

Powered by **Zustand** with **localStorage persistence** via `zustand/middleware`.

```typescript
interface CartState {
  items: CartItem[];       // Current cart items
  addItem(item);           // Adds or increments quantity
  removeItem(id);          // Removes item completely
  updateQuantity(id, delta); // +1 / -1 (removes if qty reaches 0)
  clearCart();             // Empties the cart
  getTotal();              // Computed total (sum of price × qty)
}
```

The cart persists across page navigations and browser refreshes, enabling a waiter to switch tables mid-order without losing their cart.

### Order Hook (`useOrder`)

A thin custom hook that wraps cart and API interaction:

```typescript
const { placeOrder, isSubmitting } = useOrder();
await placeOrder(tableId, hotelId);
```

**Optimistic Pattern:**
1. Takes a snapshot of current cart items
2. Clears the cart immediately (instant feedback)
3. Posts the order to the API
4. On failure, restores the snapshotted items to the cart

---

## Design System

WebBill uses a custom Tailwind-based design system defined in `tailwind.config.*` and `globals.css`.

### Color Palette

| Token | Hex | Usage |
|---|---|---|
| `webbill-burgundy` | `#5C2D27` | Primary brand, buttons, CTAs |
| `webbill-cream` | `#F8F5F2` | Base background |
| `webbill-tan` | `#D4A373` | Secondary accent, decorative |
| `webbill-dark` | `#1A1211` | Primary text, headings |
| `webbill-muted` | `#8B7E7A` | Secondary text, placeholders |

### Typography

- **Body**: `Inter` (Google Fonts) — clean, readable
- **Headings**: `Outfit` (Google Fonts) — modern, bold
- Both loaded via `next/font/google` for performance

### Component Classes

| Class | Description |
|---|---|
| `.premium-card` | White rounded card with soft shadow, lift on hover |
| `.btn-primary` | Burgundy filled button with rounded corners |
| `.shimmer` | Loading skeleton animation |
| `.no-scrollbar` | Hides scrollbar while preserving scroll behavior |
| `.animate-pulse-subtle` | Gentle scale + opacity pulse for delayed orders |

### CSS Custom Properties

```css
:root {
  --df-burgundy: #5C2D27;
  --df-cream: #F8F5F2;
  --df-tan: #D4A373;
  --df-dark: #1A1211;
  --df-muted: #8B7E7A;
  --df-depth-1: 0 4px 20px rgba(92, 45, 39, 0.04);
  --df-depth-2: 0 8px 30px rgba(92, 45, 39, 0.08);
}
```

---

## Project Structure

```
hotel-management-saas/
├── app/                          # Next.js 14 App Router
│   ├── layout.tsx                # Root layout with Google Fonts
│   ├── page.tsx                  # Landing page (/)
│   ├── globals.css               # Design system, Tailwind utilities
│   ├── api/                      # API Route Handlers
│   │   ├── menu/route.ts         # GET /api/menu
│   │   ├── orders/
│   │   │   ├── route.ts          # GET, POST /api/orders
│   │   │   └── [id]/route.ts     # GET, PATCH /api/orders/:id
│   │   ├── tables/route.ts       # GET /api/tables
│   │   └── billing/
│   │       ├── settle/route.ts   # POST /api/billing/settle
│   │       └── revenue/route.ts  # GET /api/billing/revenue
│   ├── waiter/page.tsx           # Waiter Panel (/waiter)
│   ├── kitchen/page.tsx          # Kitchen Display (/kitchen)
│   ├── billing/page.tsx          # Billing Counter (/billing)
│   └── receipt/
│       └── [orderId]/page.tsx    # Public Digital Receipt
│
├── src/                          # Shared application code
│   ├── lib/
│   │   ├── db.ts                 # Prisma singleton instance
│   │   ├── config.ts             # App config (hotelId, GST rate, etc.)
│   │   └── sync.ts               # Sync utilities
│   ├── hooks/
│   │   ├── useOrder.ts           # Order placement hook
│   │   └── store/
│   │       └── useCartStore.ts   # Zustand cart store
│   ├── components/
│   │   └── dashboards/
│   │       └── waiter/           # Waiter-specific components
│   └── services/                 # API service layer
│
├── prisma/
│   └── schema.prisma             # MongoDB data models
│
├── seed-essential.ts             # Seeds hotel, tables, starter menu
├── seed-menu.js                  # Seeds full restaurant menu
│
├── next.config.js                # Next.js config (standalone output)
├── tailwind.config.*             # Tailwind with custom color tokens
├── .env                          # Environment variables (not committed)
├── .gitignore
└── package.json
```

---

## Getting Started

### Prerequisites

- **Node.js** ≥ 18.x
- **npm** ≥ 9.x
- A **MongoDB Atlas** account (free tier works perfectly)

### Installation

```bash
# Clone the repository
git clone https://github.com/Dhruvil1308/hotel-management-saas.git
cd hotel-management-saas

# Install dependencies
npm install
```

This also runs `prisma generate` automatically via the `postinstall` script.

### Environment Variables

Create a `.env` file in the project root:

```env
# ─── DATABASE (MongoDB Atlas) ──────────────────────────────────────
DATABASE_URL="mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<dbname>?retryWrites=true&w=majority"

# ─── AUTHENTICATION ────────────────────────────────────────────────
NEXTAUTH_SECRET="<generate-with-openssl-rand-base64-32>"
NEXTAUTH_URL="http://localhost:3000"

# ─── APP CONFIG (Set AFTER running seed script) ───────────────────
NEXT_PUBLIC_HOTEL_ID="<hotel-objectid-from-seed>"
NEXT_PUBLIC_HOTEL_NAME="Saffron Bay"
```

> **Important:** `NEXT_PUBLIC_HOTEL_ID` must be set after running the seed script. The seed will print the Hotel ObjectId to the console.

### Database Setup & Seeding

**Step 1: Generate the Prisma client**
```bash
npx prisma generate
```

**Step 2: Seed essential data** (Hotel + 25 Tables + Starter Category)
```bash
npx ts-node seed-essential.ts
```

The output will show:
```
─── SEEDING ESSENTIAL DATA (MongoDB) ───
✓ Hotel "Saffron Bay" ready (ID: 69d4c34c555a28be78a1ccd2)
✓ 25 Tables synchronized.
✓ Category "Starters" ready.
✓ 8 Menu Items synchronized.
─── SEEDING COMPLETE ───

🏨 Hotel ID: 69d4c34c555a28be78a1ccd2
```

**Step 3: Copy the Hotel ID into your `.env`:**
```env
NEXT_PUBLIC_HOTEL_ID="69d4c34c555a28be78a1ccd2"
```

**Step 4: (Optional) Seed the full restaurant menu** (9 categories, 50+ items)
```bash
node seed-menu.js
```

This seeds a complete Indian café menu including:
- TEA (Regular, Ginger, Ginger Fudina)
- ICE TEA (Lemon, Peach, Strawberry, Green Apple)
- HOT COFFEE (Cappuccino, Latte, Mocha, Irish, Tiramisu, etc.)
- CLASSIC SANDWICHES
- SANDWICH (Mumbai Style, Paneer Tikka, American Club, etc.)
- GARLIC BREAD
- PASTA (Red Sauce, White Sauce, Mac & Cheese, Pink, Pesto)
- FRENCH FRIES
- NAAN PIZZA

### Running Locally

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Navigate to:
- `/waiter` — Take orders
- `/kitchen` — Monitor production
- `/billing` — Settle bills

---

## Deployment (Vercel)

This project is optimized for Vercel with `output: 'standalone'` in `next.config.js`.

**Steps:**

1. Push your code to a GitHub repository
2. Import the project in [Vercel Dashboard](https://vercel.com)
3. Add all `.env` variables under **Settings → Environment Variables**:
   - `DATABASE_URL`
   - `NEXTAUTH_SECRET`
   - `NEXTAUTH_URL` (your Vercel domain, e.g., `https://webbill.vercel.app`)
   - `NEXT_PUBLIC_HOTEL_ID`
   - `NEXT_PUBLIC_HOTEL_NAME`
4. Deploy — Vercel automatically runs `npm install` which triggers `prisma generate`

> **Note:** MongoDB Atlas must add Vercel's IPs to the Network Access whitelist, or use `0.0.0.0/0` (allow all) for production.

---

## Configuration

All runtime configuration is centralized in `src/lib/config.ts`:

```typescript
export const HOTEL_ID = process.env.NEXT_PUBLIC_HOTEL_ID || '';
export const HOTEL_NAME = process.env.NEXT_PUBLIC_HOTEL_NAME || 'Saffron Bay';

export const APP_CONFIG = {
  hotelId: HOTEL_ID,
  hotelName: HOTEL_NAME,
  gstRate: 0.05,        // 5% GST
  currency: '₹',
  pollInterval: 5000,   // 5 seconds for kitchen polling
} as const;
```

To change the GST rate, only update `gstRate` in this file — all billing logic reads from this constant.

---

## GST & Billing Logic

WebBill implements **India's GST (Goods and Services Tax)** natively:

| Component | Formula |
|---|---|
| Subtotal | Sum of (item.price × quantity) for all items |
| GST Amount | subtotal × 0.05 (5%) |
| Grand Total | subtotal + gstAmount |
| Settlement Amount | Grand Total (stored in `transaction.amount`) |

**Price Snapshotting:**
When an order is created, item prices are stored in `OrderItem.price` — a snapshot of the price at time of ordering. This ensures historical accuracy even if menu prices are updated later.

**Discount Handling:**
When settling multiple orders for the same table, any discount is divided proportionally across all orders:
```typescript
discount: discount / orders.length
```

---

## Seed Data

### seed-essential.ts
Seeds the minimum required data to operate:
- 1 Hotel (`Saffron Bay`, slug: `saffron-bay`)
- 25 Tables (`T1` through `T25`, 4-seat capacity each)
- 1 Menu Category (`Starters`)
- 8 Sample Menu Items

Uses `upsert` / `findFirst` patterns to be **idempotent** — safe to run multiple times.

### seed-menu.js
Seeds the complete restaurant menu:
- 9 Categories
- 50+ Menu Items with realistic Indian café pricing (₹50 – ₹300)
- Uses `findFirst` + `create`/`update` to be idempotent
- Works with the same hotel created by `seed-essential.ts`

---

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Commit with clear messages: `git commit -m "feat: add table reservation system"`
4. Push to your fork: `git push origin feature/your-feature-name`
5. Open a Pull Request

### Development Notes
- All new API routes must accept `hotelId` for multi-tenancy support
- UI components should use the `webbill-*` Tailwind color tokens
- All monetary calculations must use `parseFloat` / `toFixed(2)` — avoid floating point issues
- Add optimistic updates for any interactive state changes in the KDS or Waiter panels

---

## License

This project is licensed under the **MIT License**.

---

<div align="center">

**WebBill** — Built with ❤️ by [WebCultivation](https://github.com/Dhruvil1308)

*Zero compromise. Maximum hospitality.*

</div>
