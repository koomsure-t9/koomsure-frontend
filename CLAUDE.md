# CLAUDE.md

Single-file React app: everything lives in `src/App.tsx` (~2400 lines, Thai comments throughout).
Don't split it into multiple files/components speculatively — it works, type-checks cleanly, and
the project's organization effort has gone into the backend (see `../koomsure-backend/AGENTS.md`).
Only refactor structure if actually asked to.

## Current status (as of 2026-09-13)

- Wired to a real backend (`../koomsure-backend`, FastAPI + Postgres) via `API_BASE =
  "http://localhost:8000"` (top of `App.tsx`). No more hardcoded `PRODUCTS` mock array or fake
  deal/admin-login data — catalog, price history, compare, both deal dashboards, and admin
  login/CRUD all hit real endpoints.
- Product images come back from the backend as relative paths (e.g. `/static/products/P001.jpg`).
  **Always** pass `.image` through `resolveImageUrl()` before using it as an `<img src>` — a raw
  relative path resolves against the frontend's own origin (:5173) instead of the backend (:8000)
  and silently 404s. This bit us once already; see the 4 existing call sites for the pattern.
- `npm run build`'s `tsc -b` step passes with **zero** errors (fixed ~140 pre-existing
  implicit-`any` errors this session via real interfaces — `Listing`, `Product`,
  `PriceHistoryEntry`, `CompareRow`, `Deal`, etc.). Keep it that way; don't reintroduce implicit
  `any` when editing. The one deliberate exception is `AdminEntityForm`/`AdminEntityTable`/
  `ADMIN_ENTITIES`, which use loose `Record<string, any>` row values since fields vary per
  entity — that's a scope boundary, not an oversight.
- Admin login is real now (JWT bearer token from the backend), not a hardcoded demo credential.

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
