<div align="center">

# 🎨 Hermex Frontend Showcase
### Visual UI/UX Architecture & Live Interface Gallery

[ **English** ] &nbsp;•&nbsp; [ [Українська](frontend-showcase.ua.md) ] &nbsp;•&nbsp; [ [System Overview](README.md) ] &nbsp;•&nbsp; [ [Screenshots Gallery](screenshots/README.md) ]

<p align="center">
  Next.js 14.2 &bull; Tailwind CSS &bull; 3D EMV Card &bull; Server-Sent Events (SSE)
</p>

</div>

This document provides a comprehensive visual and architectural breakdown of the two independent client applications in the Hermex ecosystem:
1. 🌐 **Hermex Storefront (`website`)** — Flagship consumer electronics & tech marketplace (Port `:3000`).
2. 💳 **HermexPay Hosted Checkout (`payment-portal`)** — Isolated secure payment gateway (Port `:3001`).

---

## 📸 High-Resolution UI Screenshots Gallery

All screenshots are captured directly from live running services and stored in [`screenshots/`](screenshots/):

| Screenshot | Description | Component / Route | Direct Link |
| :--- | :--- | :--- | :---: |
| `01_storefront_catalog.png` | Main storefront home, catalog grid, sticky blur navbar, and flagship hero banner | `website` (`/`) | [View Image](screenshots/01_storefront_catalog.png) |
| `02_faceted_filters.png` | Sidebar faceted filters with Staged Draft State and real-time count button | `website` (`/`) | [View Image](screenshots/02_faceted_filters.png) |
| `03_filtered_catalog.png` | Catalog dynamically filtered by Apple brand with active filter chips | `website` (`/`) | [View Image](screenshots/03_filtered_catalog.png) |
| `04_cart_drawer.png` | Slide-over cart drawer with real-time batch price validation & free shipping tier | `website` (`CartDrawer`) | [View Image](screenshots/04_cart_drawer.png) |
| `05_payment_portal.png` | Fintech Light Theme hosted checkout with 3D EMV card and collapsible Saga Test Lab | `payment-portal` (`/pay/[id]`) | [View Image](screenshots/05_payment_portal.png) |
| `06_live_order_tracker.png` | Real-time order tracking timeline powered by Server-Sent Events (SSE) | `website` (`/orders/[id]`) | [View Image](screenshots/06_live_order_tracker.png) |

---

## 🌐 1. Hermex Storefront (`website`)

- **Tech Stack:** Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS, Zustand, Lucide Icons
- **Service URL:** `http://localhost:3000`
- **GitHub Repository:** [🔗 HermexHub/website](https://github.com/HermexHub/website)

![Hermex Storefront Catalog](screenshots/01_storefront_catalog.png)

### Key Architectural & UI/UX Features:

### 1.1. Navigation & Header
- **Glassmorphism Sticky Navbar:** Frosted glass effect (`backdrop-blur-md`) with smooth stick-on-scroll behavior.
- **Bilingual Interface (i18n):** Instant runtime switching between Ukrainian (`UA`) and English (`EN`) with persistent locale preferences.
- **Session Management:** Instant **1-click Quick Login** (`test@gmail.com`), sanitized user initials, and token refresh interceptors.
- **Reactive Cart Indicator:** Animated badge with real-time item count and total sum in UAH (₴).

### 1.2. Hero Showcase Banner
- Seamless visual composition displaying flagship devices (Apple, Samsung, Asus, Sony).
- Technical specification chips, official 24-month manufacturer warranty, and 0% banking installment tags.

### 1.3. Intelligent Sidebar Faceted Filters
![Faceted Filters](screenshots/02_faceted_filters.png)

- **Isolated Scroll Container:** Independent scrolling prevents page jitter or vertical layout distortion.
- **Staged Draft State:** Users select and unselect checkboxes without triggering premature network re-fetches.
- **Dynamic Apply Counter:** The apply button dynamically calculates and displays matching products: `Apply (9 items found)`.
- **Faceted Attributes:** Price slider, brands (Apple, Asus, Sony, Samsung...), and hardware specs (CPU, RAM, SSD storage).

### 1.4. Filtered Results & Active Chips
![Filtered Catalog](screenshots/03_filtered_catalog.png)

- Instant faceted filtering based on PostgreSQL GIN-indexed JSONB specifications.
- Interactive filter dismissal chips (`Apple ✕`) allowing immediate resets.

### 1.5. Slide-Over Cart Drawer & Stale Data Healing
![Cart Drawer](screenshots/04_cart_drawer.png)

- **Client-Side First:** Zustand state with `localStorage` persistence and hydration mismatch guards.
- **Batch Verification (`POST /api/v1/cart/validate`):** Single gRPC query to `inventory-service` validating current inventory levels and unit prices on open.
- **Free Delivery Progress Bar:** Visual indicator unlocking free express courier delivery once reaching 2,000 ₴.

### 1.6. Checkout Flow & Anti-IDOR
- Address input with Nova Poshta and courier delivery options.
- Secure `X-Idempotency-Key` (UUIDv4) header generation preventing duplicate charges.
- Seamless redirection to the isolated hosted payment portal.

### 1.7. Live Order Tracker (Server-Sent Events)
![Live Order Tracker](screenshots/06_live_order_tracker.png)

- Real-time connection to API Gateway SSE stream (`GET /api/v1/orders/:id/live`).
- Reactive timeline status stepper:  
  `Order Placed (PENDING)` ➔ `Items Reserved` ➔ `Payment Verified` ➔ `Order Confirmed (CONFIRMED)`.
- Automatic exponential backoff reconnects upon connection loss.

---

## 💳 2. HermexPay Hosted Checkout (`payment-portal`)

- **Tech Stack:** Next.js 15, React 19, TypeScript, Tailwind CSS, Lucide Icons
- **Service URL:** `http://localhost:3001`
- **GitHub Repository:** [🔗 HermexHub/payment-portal](https://github.com/HermexHub/payment-portal)

![HermexPay Hosted Checkout](screenshots/05_payment_portal.png)

### Key Architectural & UI/UX Features:

### 2.1. Fintech Light Theme (Stripe Standard)
- Premium, airy aesthetic with clean neutral backgrounds (`bg-slate-50` / `bg-white`), refined borders, and high contrast typography.
- Eliminated security theater noise: replaced clunky dev badges with a clean, credible trust statement:  
  *“Secure checkout. Hermex never stores raw payment card data.”*

### 2.2. Session-Bound Contextual Route
- Root route `/` automatically and securely redirects to the main storefront `website`.
- Access is restricted exclusively to valid, session-bound `/pay/[id]` order routes.

### 2.3. Interactive 3D Payment Card (`PaymentCardVisualizer`)
- Photorealistic metallic EMV microchip with authentic gold circuit contacts.
- Natural 4-digit block formatting (`4242 4242 4242 4242`).
- Smooth 3D flip animation showing the card back when focusing on the CVV input field.

### 2.4. Non-Resetting 15-Minute Reserve Timer
- Target expiration derived from authoritative server-side `createdAt` timestamp (+15 minutes).
- Synchronized with wall-clock time and cached in `localStorage`.
- Pressing **F5**, refreshing the page, or opening in a new tab **does not reset** the timer to 15:00.
- Order details and pricing are persisted locally without flickering.

### 2.5. Collapsible Saga Test Lab (Developer Mode)
- Clean collapsible drawer: `[ 🧪 Saga Test Lab (Developer Mode) ]`.
- 1-click test simulation for all distributed transaction paths:
  - `[ ✅ Success ]` ➔ `4242...4242` (`payment.succeeded` ➔ order confirmed).
  - `[ ❌ Insufficient Funds ]` ➔ `4000...0002` (`payment.failed` ➔ inventory restocked ➔ order cancelled).
  - `[ 🚫 Card Expired ]` ➔ `4000...0003`.
  - `[ 🛑 Issuer Declined ]` ➔ `4000...0004`.
  - `[ ⏱️ Timeout ]` ➔ `4000...0005`.

### 2.6. Processing & 3D-Secure Modal
- Step-by-step processing animation (Session validation ➔ Issuer 3D-Secure ➔ Transaction capture).
- Automatic return redirect to the storefront Live Order Tracker.
