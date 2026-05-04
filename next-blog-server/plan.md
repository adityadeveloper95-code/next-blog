# Next Blog Server — Phased Build Plan

This plan implements [`hld.md`](../hld.md) (schema, API contract, status codes) using the stack and layering in [`backend_architecture.md`](./backend_architecture.md).

**Contract note:** API-facing errors should match the HLD (`message` + `errors[]` for validation). **Controllers** map validation and service/domain outcomes to HTTP status and response bodies. **Centralized** error middleware is only for **uncaught errors (500)** and **generic unmatched `/api/*` (404)**.

---

## Phase 0 — Repository and toolchain

- Initialize Node + TypeScript: `package.json`, `tsconfig`, optional ESLint/Prettier.
- Scripts: `dev`, `build`, `start`, `test`, `test:watch`, `db:migrate`, `db:generate` (Prisma).
- Test stack: **Vitest or Jest** + **Supertest** for HTTP integration tests.
- `.env.example`: `DATABASE_URL`, `REDIS_URL`, `SESSION_SECRET`, `PORT`, `NODE_ENV`, `CORS_ORIGIN` (Next.js origin), and cookie/session flags as needed (`COOKIE_SECURE`, etc.).

**Exit criteria:** `npm run build` and `npm test` run (tests may be minimal at first).

---

## Phase 1 — Config, layout, and dependency wiring

- Folder layout per architecture doc: `src/config/`, `routes/`, `controllers/`, `services/`, `db/` (Prisma), `middlewares/`, `utils/`, `types/`, `index.ts`.
- **Environment:** validate with **Zod** in `src/config/env.ts` (fail fast on bad/missing vars).
- **PostgreSQL:** Prisma schema matching HLD—`users`, `blogs`, `comments` with `created_at` / `updated_at`, FKs, indexes (`blogs` by `created_at DESC`, `comments` by `(blog_id, created_at DESC)`).
- **Redis:** client module in `src/config/redis.ts` for session store and health checks.
- **Express app:** middleware order—helmet → cors (`credentials: true`, explicit origin) → rate limit → hpp → `express.json` / `urlencoded` → **session (Redis store)** → pino-http → compression.
- **Errors:** **controllers** translate validation failures and service-layer results (or typed domain errors) into HLD status codes (`422`, `409`, etc.) and the **HLD JSON error shape**. **Centralized** middleware handles only **unexpected failures (500)** and a **catch-all 404** for unmatched `/api/*` routes.
- **Graceful shutdown:** disconnect Prisma, quit Redis.

**Tests:** app boots with test env; optional test that invalid env fails clearly.

---

## Phase 2 — Migrations and `GET /api/health`

- Initial Prisma migration; document local Postgres workflow.
- **`GET /api/health`:** Prisma `SELECT 1`, Redis `PING`; `200` with `{ status, database, redis }` when both succeed; `503` when either fails (per HLD).

**Tests:** integration tests (test DB + Redis or containers)—`200` vs `503` payloads; endpoint is public (no auth).

---

## Phase 3 — User signup: `POST /api/user`

- Hash passwords (bcrypt or argon2); enforce HLD limits and username `^[a-z0-9]+$`.
- **`POST /api/user`:** `201` response body per HLD; `409` duplicate username; `422` validation with field errors.

**Tests:** `201`; `409`; `422`; response never exposes `password_hash`.

---

## Phase 4 — Login: `POST /api/login`

- Zod-validate body; verify credentials.
- Create Redis session (`session:{sessionId}` per HLD) with `user_id`, `username`, `created_at`, `expires_at`; set HTTP-only, secure (in prod), SameSite session cookie.
- **`200`** with `message` + `user`; **`401`** invalid credentials; **`422`** validation.

**Tests:** `200` and `Set-Cookie`; `401`; `422`; preserve cookies for later phases.

---

## Phase 5 — Current user and logout

- **`requireAuth` middleware:** load session, attach user to request.
- **`GET /api/user`:** `200` id, name, username; **`401`** if unauthenticated.
- **`POST /api/logout`:** destroy current session only, clear cookie; **`200`**; **`401`** if no session.

**Tests:** `GET /api/user` after login vs without cookie; logout then `GET /api/user` → `401`; logout without session → `401`.

---

## Phase 6 — Create blog: `POST /api/blog`

- **`requireAuth`:** set `user_id` from session.
- Validate title, slug, content; enforce global slug uniqueness; slug immutable after create (enforced on create path).
- **`201`** with blog + nested `author`; **`409`** slug conflict; **`422`** validation; **`401`** if not logged in.

**Tests:** happy path; duplicate slug; unauthorized; validation errors.

---

## Phase 7 — Update and delete blog: `PUT /api/blog/:id`, `DELETE /api/blog/:id`

- Owner-only: **`403`** if not owner; **`404`** if missing.
- **`PUT`:** title + content only; response shape per HLD including `updated_at`.
- **`DELETE`:** **`204`** empty body.

**Tests:** owner vs non-owner; `404`; bad body `422`.

---

## Phase 8 — Public blog read: `GET /api/blog/:slug`

- Public; lookup by **slug**; include **all** comments with authors (no comment pagination).
- **`404`** if not found.
- Use Prisma `include` / efficient query to avoid N+1 where reasonable.

**Tests:** `200` with comments; `404`; author shapes match HLD (detail sample omits author `id` on blog—match spec).

---

## Phase 9 — Create comment: `POST /api/blog/:id/comment`

- **`requireAuth`**; blog must exist (`404`).
- **`201`** with comment, `blog_id`, author (`id`, name, username); **`422`** content rules.

**Tests:** happy path; missing blog; unauthenticated; validation.

---

## Phase 10 — Update and delete comment: `PUT` / `DELETE` `/api/blog/:id/comment/:commentId`

- Comment belongs to blog; only comment owner may mutate (`403` otherwise); `404` if blog or comment missing or mismatched.
- **`PUT`:** `200` id, content, `updated_at`.
- **`DELETE`:** hard delete, **`204`**.

**Tests:** owner vs other user; wrong ids; auth and validation.

---

## Phase 11 — Feed: `GET /api/feed?page=<int>`

- Public; fixed `page_size` (e.g. 20); `page` starts at 1; **`400`** invalid `page`.
- Sort blogs by `created_at` DESC; each item: title, slug, `comment_count`, author, timestamps; pagination: `total_items`, `total_pages`, `items`.

**Tests:** pagination; empty feed; invalid page `400`; totals with fixtures.

---

## Phase 12 — Hardening and ops (tentative plan)

- Stricter rate limits on `/api/login` and `/api/user`.
- Document CSRF / cookie strategy for Next.js ↔ API (SameSite, CORS, credentials).
- Request logging (request id, optional `user_id`).
- README: Docker Compose for Postgres + Redis, migrate, optional seed.

---

## Traceability to HLD

| Phase | HLD coverage |
|-------|----------------|
| 0–1 | API server foundation, Postgres + Redis, session middleware; controllers own HTTP error mapping; centralized 500 + API 404 |
| 2 | `GET /api/health` |
| 3–5 | `POST /api/user`, `POST /api/login`, `POST /api/logout`, `GET /api/user`, sessions in Redis |
| 6–8 | Blog create/update/delete, `GET /api/blog/:slug` with comments |
| 9–10 | Comment create/update/delete |
| 11 | `GET /api/feed` |
| 12 | Security and operability |

Each API phase includes **automated tests** covering success paths, auth, validation (`422`), and domain errors (`401` / `403` / `404` / `409` as specified).
