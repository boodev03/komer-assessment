# Komer Dashboard — Codebase Assessment

**Scope:** high-level architecture review from reading the code only. We did **not** run the app, and the real backend logic (Supabase RLS policies, RPC/SQL function bodies, edge functions) is **not in this repo**, so anything depending on it is called out as unverified rather than asserted.

**Reviewed:** project structure and the 35 API routes; middleware and both Supabase client patterns; the feature layer (`src/features/*`) and its react-query hooks; the money-related routes (payout, withdrawals, refunds, driver ledgers); and the two committed SQL files. **Not reviewed:** Supabase RLS, RPC/edge-function source, production config, live traffic.

---

## 1. Overall architecture

Komer Dashboard is a **Next.js 15 (App Router, React 19) admin dashboard on Supabase**. There is no separate backend — the Next.js API routes plus Supabase (Postgres, Auth, Storage, Edge Functions) are the whole server. It serves admin and vendor users (rider is referenced but mostly not built), with English/French i18n, TanStack Query for data, Zustand for state, and Radix + Tailwind for UI. ~35k lines, 395 files, four contributors, last commit 2026-06-29.

The foundations are solid:

- **Feature-based structure.** `src/features/*` (order, product, vendor, payout, notification, reports…) keeps domains separated, each with its own `api/` hooks and `components/`.
- **Generated DB types** (`database.types.ts`) give type inference from the database through to the UI.
- **The money code was written with some care** — bcrypt PIN hashing with validation (`lib/pin-utils`), payout cooldowns, and refund SQL that uses row locks (`SELECT … FOR UPDATE`) and guards against double-releasing funds.

There are two distinct ways the app talks to Supabase, and the difference matters:

| Pattern | Key | RLS | Used by |
| --- | --- | --- | --- |
| User-scoped client (`supabaseServer()`) | anon + user cookie | **enforced** | vendor routes, server components |
| Admin client | `SUPABASE_KEY` (service_role) | **bypassed** | 13 `/api/admin/*` routes |

For a product at this stage, a Supabase-backed monolith is a reasonable choice. The concern is not the shape of the system; it's that it now moves real money (payouts, refunds, wallet transfers) while the parts that make money handling safe live outside the repo and outside version control.

---

## 2. Biggest risks

### R1 — The security and correctness that matters most isn't in the repo
Authorization for most routes relies on Supabase **RLS**, and the core money logic runs in **RPCs and edge functions** (`payout_tranzak`, `fn_transfer_funds`, refund/cancel functions). None of that is in the codebase, and there's no migrations directory, so it isn't version-controlled here either. We can't review it, and the team can't audit or roll it back through git. This is the single biggest unknown: the app can look correct while the real behavior lives somewhere we can't see.

### R2 — Some routes trust client-supplied data that should come from the session
The clearest case is `/api/admin/orders`: it reads `adminId` and `refundAmount` from the request body (`route.ts:105,138`) and passes them into the refund/cancel edge functions as the acting user. The actor's identity should be derived server-side from the session, not taken from whatever the client sends — as written, the caller decides which admin id gets recorded against a refund, and supplies the refund amount. The same edge function is called correctly from `/api/vendors/orders`, which uses `userId: user.id` from the session (`route.ts:205`). Since these admin routes run on the service_role key, there's no RLS backstop, so trusting the client here is higher-stakes.

**Audit of the other routes:** vendor routes consistently scope by `user.id` from the session (good). Other admin routes only take "which record" params (`role`, `id`, `profileId`) — acceptable, since an admin legitimately acts across tenants, not identity spoofing. The root cause is that these admin routes don't read the session user (no `getUser()`), so they can't derive the acting admin from it — which is exactly why `/api/admin/orders` reaches for `adminId` in the body. (The middleware admin-gate itself is fine and team-managed; it is not counted as a risk here.)

### R3 — Money operations aren't idempotent, and the client retries them
The payout, refund, and PIN paths do read-then-write without atomicity or idempotency keys. The withdrawal PIN attempt counter is a non-atomic read-modify-write (parallel wrong guesses defeat the lockout). The payout checks balance and then calls the edge function without a transactional guard. And react-query is configured with `mutations: { retry: 1 }`, so a transient failure can fire the same money mutation twice. Together these create double-spend / brute-force windows on the paths that touch cash.

### R4 — No real tests, and CI doesn't catch much
The only test is the default Playwright example (it navigates to playwright.dev). CI runs lint plus that placeholder test — no build, no type-check, nothing exercising the money paths. For a system handling payments there's effectively no automated safety net, and a green CI overstates confidence.

### R5 — Database schema and functions aren't version-controlled
There's no migrations folder. The two most important pieces of financial logic are loose `.sql` files in the repo root, and type generation points at a hardcoded project ref. The live database is the source of truth, which means schema/function/RLS changes are unreviewable and hard to reproduce or roll back.

---

## 3. Recommendations (highest impact first)

1. **Get the Supabase layer into version control.** Add a `supabase/` directory with migrations, RLS policies, and the RPC/edge-function source. This is the prerequisite for reviewing or testing anything else (R1, R5).
2. **Make money operations atomic and idempotent.** Do the PIN counter increment and lock decision in one DB statement; wrap balance-check + payout in a transaction with an idempotency key; reconsider `mutations.retry` for money mutations (R3).
3. **Stop trusting client-supplied identity and amounts.** In `/api/admin/orders`, derive the acting admin from the session instead of the body's `adminId`, and recompute/validate `refundAmount` server-side (R2).
4. **Test the money and auth paths, and make CI mean something.** Add tests for PIN lockout, payout, refund, and role-gating; make CI run build + type-check + tests, not just lint (R4).
5. **Add a CI guardrail for client-supplied identity fields.** Fail the build if an API route reads `adminId` / `userId` / `actorId` from the request body or query — they must come from the session (R2).
6. **Remove PII from logs, and confirm Sentry replay masking.** Server code logs full profile objects, and Sentry session replay records a dashboard full of personal/financial data — both leak personal data into the observability pipeline.
7. **Bound external calls in the request path.** `products/route.ts` calls OpenAI with a plain `fetch` — no timeout, synchronous, inside the product-create path — so a slow response blocks the request. Add a timeout and move translation off the critical path.
8. **Consolidate Supabase access into a server-side `services/` layer.** Move the DB logic that's currently hand-rolled and duplicated across API routes into server-only functions, consumed by **Server Components for reads** and **Server Actions (`'use server'`) for writes**. The goal isn't raw speed — the DB call is identical — it's centralizing auth and cutting boilerplate.
   - **Server Actions — pros:** no API endpoint to write; end-to-end type safety; work directly with forms and support progressive enhancement (function without JS); much less boilerplate. Only our own web/app can call them
   - **Keep API routes** for service_role/admin operations, cacheable GETs, and anything external services or webhooks must call.
9. **Housekeeping (lower priority).** Settle one file-naming convention and apply it — the same folder mixes styles today (`src/hooks/use-mobile.tsx` vs `useTranslation.ts`; components like `payment-dialogs.tsx` vs `MerchantProfile.tsx`), which a lint rule can enforce. Drop the `Database | any` cast on `supabaseServer()`; dial down Sentry `tracesSampleRate` for production and unify the hardcoded-vs-env DSN; dedupe the two `postcss.config.*` files; remove the hardcoded project ref in `next.config.ts` / `Makefile`; fix the README `.env` step and the phantom `type-check` script.

---

## 4. Open questions for the team

- Where do the RLS policies and RPC/edge-function definitions live, and are they tested? This decides how much of R1/R2 is already handled server-side.
- Does `payout_tranzak` (and the refund/transfer functions) enforce idempotency and re-validate amounts, or does it trust the caller?
- Is the service_role gate ever relied on for anything beyond the current `/api/admin/*` paths?
- What's the intended state of the **rider** role — partially built, or planned?

---

## 5. Fix now

- Stop acting on client-supplied `adminId` / `refundAmount` in `/api/admin/orders`; derive the actor from the session (R2).
- Make the PIN counter atomic and add idempotency to payout/refund before scaling volume (R3).
- Add the CI check that flags identity fields read from the request body/query (R2).
- Get migrations + RLS + function source under version control so the rest can be reviewed (R1/R5).

**Bottom line:** the app is well-organized and the money code shows intent, but it has grown into a payments system while the parts that keep payments safe — authorization, atomic money operations, and the database logic itself — either live outside the repo or aren't hardened yet. The priorities are to pull the Supabase layer into version control, make the money paths safe by construction, and stop trusting client-supplied identity and amounts on the routes that touch money.
