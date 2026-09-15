# CLAUDE.md

Single-file React app: everything lives in `src/App.tsx` (~2400 lines, Thai comments throughout).
Don't split it into multiple files/components speculatively — it works, type-checks cleanly, and
the project's organization effort has gone into the backend (see `../koomsure-backend/AGENTS.md`).
Only refactor structure if actually asked to.

## Current status (as of 2026-09-15)

- Wired to a real backend (`../koomsure-backend`, FastAPI + Postgres) via `API_BASE =
  "http://localhost:8000"` (top of `App.tsx`). No more hardcoded `PRODUCTS` mock array or fake
  deal/admin-login data — catalog, price history, compare, both deal dashboards, admin login/CRUD,
  and customer login/favorites all hit real endpoints.
- Product images come back from the backend as relative paths (e.g. `/static/products/P001.jpg`).
  **Always** pass `.image` through `resolveImageUrl()` before using it as an `<img src>` — a raw
  relative path resolves against the frontend's own origin (:5173) instead of the backend (:8000)
  and silently 404s. This bit us once already; see the existing call sites for the pattern.
- `npm run build`'s `tsc -b` step passes with **zero** errors (fixed ~140 pre-existing
  implicit-`any` errors this session via real interfaces — `Listing`, `Product`,
  `PriceHistoryEntry`, `CompareRow`, `Deal`, etc.). Keep it that way; don't reintroduce implicit
  `any` when editing. The one deliberate exception is `AdminEntityForm`/`AdminEntityTable`/
  `ADMIN_ENTITIES`, which use loose `Record<string, any>` row values since fields vary per
  entity — that's a scope boundary, not an oversight.
- **One account system, one login form (`UserAuthForm`), used everywhere.** There used to be two
  separate systems (admin username+password vs. customer email+password) — merged same day after
  it turned out confusing to actually use. Now: real JWT, session (`token`/`email`/`role`)
  persisted in `localStorage` (`USER_TOKEN_STORAGE_KEY`/`USER_EMAIL_STORAGE_KEY`/
  `USER_ROLE_STORAGE_KEY`) so it survives reloads regardless of which surface logged you in. The
  module-level `authToken` variable (used by `apiRequest` for admin CRUD calls) is kept in sync
  with whichever of these logged in most recently.
  - **Customer-facing entry point**: visible "เข้าสู่ระบบ" button in `UserApp`'s navbar (logged-in
    shows the email + a logout icon, **plus a `ShieldCheck` "go to admin" icon iff `userRole ===
    "admin"`** — a regular customer never sees that icon, only an admin account that happens to be
    logged in through the normal customer flow; it just navigates to `/admin`, where the already-
    persisted session gets them straight into the dashboard, no second login). Deliberately
    discoverable, unlike the admin path below. Signing up here always gets `role: "user"`
    server-side, regardless of what's sent. Clicking the favorite heart while logged out opens
    `UserAuthForm` instead of favoriting locally; while logged in, `toggleFavorite` optimistically
    updates local state and calls the real `addFavoriteRemote`/`removeFavoriteRemote`, rolling
    back on failure.
  - **Admin entry point**: still just the `/admin` path (see `isAdminPath()` in `App()`) — no
    button/link anywhere in the customer UI, same reasoning as before (regular users shouldn't see
    it, real backoffices work this way). What changed: it's the *same* `UserAuthForm`, not a
    separate admin-only form. `App()` gates on the resolved session's `role`: no session → show
    login; session but `role !== "admin"` → a plain "you don't have admin access" screen (a
    regular customer account landing on `/admin` doesn't silently get in); `role === "admin"` →
    `AdminDashboard`. The only way an account gets `role: "admin"` is a direct database write (see
    `koomsure-backend`'s seed script) — never through the sign-up form.

## Two known, deliberately-unfixed pre-existing bugs (flagged, not fixed)

- `T.blueSoft` is referenced (search for it) but doesn't exist on the `T` design-token object —
  a close button renders with an undefined background color.
- A related-product thumbnail price is missing a null-guard that the identical logic in
  `ProductCard` has (`minPrice()` can return `null`); would throw if a related product has zero
  in-stock listings.

## Known gaps / likely next asks

- No image upload UI — the admin "image" field is a plain URL/path text input. Matches the
  backend, which also has no upload endpoint yet.

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

Admin panel: open `http://localhost:5173/admin` directly (nothing in the UI links there — see
"Current status" above). Vite's dev server has SPA fallback built in so this just works; a real
deploy target needs the equivalent (rewrite-all-paths-to-index.html) configured.
