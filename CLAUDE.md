# CLAUDE.md

Single-file React app: everything lives in `src/App.tsx` (~3800 lines, Thai comments throughout).
Don't split it into multiple files/components speculatively — it works, type-checks cleanly, and
the project's organization effort has gone into the backend (see `../koomsure-backend/AGENTS.md`).
Only refactor structure if actually asked to.

## AI Assistant (added 2026-10-08)

`AiAssistantCard` on the homepage (always visible, above `TopDealDashboard`) and
`AiAnalyzeProductButton` inside `ProductModal` both call
`POST /api/ai-assistant/analyze` (`fetchAiAnalysis`). The homepage card sends free text
(`{query}}`) and the backend fuzzy-matches it to a product; the in-modal button already knows the
exact product so it sends `{product_id}` directly — always a precise result, no matching step.
All the analysis numbers are computed backend-side (see backend `CLAUDE.md`) — this frontend only
renders whatever text comes back. `AiAnalyzeProductButton` is keyed by `product.product_id` so
its state resets when switching between related products inside the same modal instance.

## Current status (as of 2026-10-07 — "final version" pass)

- Wired to a real backend (`../koomsure-backend`, FastAPI + Postgres) via `API_BASE =
  "http://localhost:8000"` (top of `App.tsx`).
- Product images come back from the backend as relative paths (e.g. `/static/products/P001.jpg`).
  **Always** pass `.image` through `resolveImageUrl()` before using it as an `<img src>` — a raw
  relative path resolves against the frontend's own origin (:5173) instead of the backend (:8000)
  and silently 404s.
- `npm run build`'s `tsc -b` step passes with **zero** errors. Keep it that way; don't reintroduce
  implicit `any` when editing. The one deliberate exception is `AdminEntityForm`/
  `AdminEntityTable`/`ADMIN_ENTITIES`, which use loose `Record<string, any>` row values since
  fields vary per entity — that's a scope boundary, not an oversight.
- **Two completely separate login forms/token systems, deliberately not sharing anything.** This
  was unified once (one account, one `UserAuthForm`, a `role` field), then un-merged same project
  — confusing in practice, and a PDPA requirement to stop collecting customer email made "one
  system" even less natural. Don't re-merge without being asked.
  - **Customer-facing**: `UserAuthForm` (username + password), triggered by the visible
    "เข้าสู่ระบบ" button in `UserApp`'s navbar. Session (`userToken`/`userUsername`) persists in
    `localStorage` (`USER_TOKEN_STORAGE_KEY`/`USER_USERNAME_STORAGE_KEY`) — regular users expect
    "stay logged in." Exists solely to back persistent favorites: clicking the heart while logged
    out opens `UserAuthForm` instead of favoriting locally; while logged in, `toggleFavorite`
    optimistically updates local state and calls the real `addFavoriteRemote`/
    `removeFavoriteRemote`, rolling back on failure. No admin capability of any kind reachable
    from this account — there is no role, no shared table, nothing to escalate to.
  - **Admin**: `AdminLoginForm` (username + password, a distinct component/endpoint), reachable
    only by navigating to `/admin` directly (see `isAdminPath()` in `App()`) — no button/link
    anywhere in the customer UI, matching how real backoffice tools (`/wp-admin` etc.) work. Token
    held in the module-level `adminToken` variable, **not** persisted to `localStorage` —
    deliberately logs out on reload/tab-close, unlike the customer session. `apiRequest` (used by
    all admin CRUD + refresh-prices calls) attaches `adminToken`; nothing else does.
- **Coupons removed entirely** — no coupon column/field anywhere, no Coupons admin tab. Not
  coming back; deemed too complex for this project's scope.
- **Shipping is now a real, researched estimate, not mock data.** `CompareTable` has a province
  `<select>` (persisted to `localStorage` via `DELIVERY_PROVINCE_STORAGE_KEY`) and fetches
  `GET /api/shipping-policies` once per table mount; `estimateShipping()` combines the two to show
  "ฟรี" / "~฿X" / "ไม่ทราบค่าส่ง" per listing — never a fabricated number for the ~16 platforms with
  no verified policy. The old `net_price` field (which used to bake in mock coupon+shipping math)
  is just `price` now everywhere, matching the backend's rename.
- **Price history is the flagship feature now.** Every `ProductCard` shows a `MiniPriceSparkline`
  (compact, axis-less, colored by trend direction) when that product has ≥2 distinct price-history
  dates — silently renders nothing otherwise, no placeholder clutter. The full `PriceHistoryChart`
  (with legend/axis/tooltip, per-platform lines) stays in `ProductModal` as the detailed drill-in
  view. `TopDealDashboard` got a one-line caption reinforcing "real tracked prices, not estimates."
- **Admin products tab has a "refresh prices" button** (`RotateCcw` icon, only rendered when
  `entityKey === "products"` in `AdminEntityTable`) that calls the real
  `refreshProductPrices(productId)` helper → `POST /api/admin/products/{id}/refresh-prices`, which
  genuinely re-scrapes that product's retailer pages on the backend. **Takes 30s-2min** — the
  button disables itself and spins while in flight; a result summary row ("2/3 อัปเดตสำเร็จ...")
  appears under that product's row afterward, and the table reloads so updated prices show
  immediately. Scraping accuracy is inherently imperfect (see backend `CLAUDE.md`) — an occasional
  wrong-looking price after a refresh is a scraper limitation, not a frontend bug.
- **"สแกนราคาใหม่ทั้งหมด" (refresh all) button** next to "เพิ่มรายการใหม่" in the Products tab
  header — loops `refreshProductPrices` over every row **sequentially, one at a time** (not
  parallel — matches the scraper's own one-URL-at-a-time design, avoids hammering retailer sites
  harder than a single refresh already does). For the full seeded catalog (~28 products) this can
  realistically take tens of minutes; there's a progress banner (`bulkRefresh` state, "กำลังสแกน
  X/Y") and the button turns into a "หยุด" (stop) button that sets `bulkCancelRef.current = true`,
  checked between products (won't interrupt a scrape already in flight, just stops starting new
  ones). This state lives in `AdminDashboard`, not `AdminEntityTable`, specifically so it survives
  switching between the Products/Listings tabs mid-scan.

## Two known, deliberately-unfixed pre-existing bugs (flagged, not fixed)

- `T.blueSoft` is referenced (search for it) but doesn't exist on the `T` design-token object —
  a close button renders with an undefined background color.
- A related-product thumbnail price is missing a null-guard that the identical logic in
  `ProductCard` has (`minPrice()` can return `null`); would throw if a related product has zero
  in-stock listings.

## Known gaps / likely next asks

- No image upload UI — the admin "image" field is a plain URL/path text input.
- Only 8 of 24 platforms have a real shipping-policy estimate (see backend CLAUDE.md for which).
  `estimateShipping()` already handles the "unknown" case correctly — adding more platforms is a
  backend-side research task, not a frontend change.

## Running both together

Backend must be running for this to do anything useful:
```
cd ../koomsure-backend && uv run fastapi dev main.py   # :8000
```
Then here:
```
npm install   # if node_modules/.bin scripts get "Permission denied", chmod +x node_modules/.bin/*
npm run dev   # :5173, falls back to :5174+ if taken
```

Admin panel: open `http://localhost:5173/admin` directly (nothing in the UI links there). Vite's
dev server has SPA fallback built in so this just works; a real deploy target needs the
equivalent (rewrite-all-paths-to-index.html) configured.
