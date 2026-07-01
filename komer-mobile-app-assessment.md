# Komer Mobile App — Codebase Assessment

**Scope:** high-level architecture review from reading the code only. We did **not** run the app, and the real backend logic (Supabase RLS policies, RPC/SQL functions, edge functions) is **not in this repo**, so anything depending on it is called out as unverified rather than asserted.

**Reviewed:** project structure and the Expo Router screens; the `src/api/*` and provider layers; auth (phone-OTP / Google / Apple via Supabase); the order + payment flow (edge-function mediated); env config and token storage; and CI. **Not reviewed:** Supabase RLS, RPC/edge-function source, production config, live traffic.

---

## 1. Overall architecture

Komer Mobile is the **customer app**: a React Native / Expo (SDK 54, Expo Router, React 19) client for food delivery, ride-hailing, wallet, and kredits, built on the Obytes starter. It talks directly to Supabase (auth, ~14 RPCs, a few edge functions); the axios client against `API_URL` is unused. ~35k lines, 489 files, targeting Cameroon (phone-OTP, MTN MoMo). Version 1.3.2, build 36 — a shipped product, concentrated on one main author.

The setup is well put together:

- **Env config is validated and split correctly.** `env.js` uses Zod and separates client vs build-time vars, so `SECRET_KEY` stays out of the app bundle.
- **Token storage is done right.** `LargeSecureStore` (`src/lib/supabase.ts`) keeps an AES key in SecureStore and the encrypted session in AsyncStorage — the standard Supabase-RN pattern.
- **Payments are server-mediated.** Orders and payment intents go through Supabase edge functions (`payments/orders/create`, `payments/intents/create`), so no payment-provider credentials live in the client. (This is the right pattern, and notably better than the rider app, which embeds a MoMo key.)
- **Good CI and tooling.** CI runs lint, type-check, translation validation, and Jest with coverage; husky + commitlint + lint-staged enforce hygiene.

The overall approach — a Supabase-backed Expo client — fits the product. As with the rest of the platform, the caveat is that the authorization and business logic that actually keep it safe live in Supabase, outside this repo.

---

## 2. Biggest risks

These are the security risks substantiable from the client code. Performance is not a concern here — polling intervals are reasonable (10–30s), lists are mostly virtualized (`FlashList`), and there's no heavy client-side work.

### R1 — QA auth bypass wired into the client (conditional — must verify)
`auth.service.ts` has two test bypasses: skipping SMS, and accepting the magic code `888888` to call a `qa-auth/login-bypass` edge function that returns a real session. Both are gated on `APP_ENV !== 'production'`, so a correctly-built production binary won't call them. The real exposure is server-side: if `qa-auth/login-bypass` is deployed to the production Supabase project, anyone can call it directly (outside the app) with a phone number and mint a session — the client-side env gate is irrelevant to an attacker. We can't see the edge function, so this is conditional and needs verification — but the impact is high enough to list. **If it's live on prod, it's account takeover by phone number.**

### R2 — Order totals are computed and trusted from the client
`use-create-order.ts` builds the full price payload on the device — `unit_price`, `total_price`, `sub_total`, `total_amount` — and sends it to `payments/orders/create`. Sending price data from an untrusted client is the anti-pattern: if the edge function trusts those numbers instead of recomputing from the vendor's catalog, a modified client could pay an arbitrary amount. Exploitability depends on server-side re-validation we can't see.

---

## 3. Recommendations (highest impact first)

1. **Verify and contain the QA bypass.** Strip the `888888` path from release builds with a compile-time flag rather than relying on a runtime env check. **Confirm with team:** is `qa-auth/login-bypass` deployed to the production Supabase project, and does it refuse there? (R1)
2. **Don't trust client-sent order totals.** **Confirm with team:** that `payments/orders/create` and `payments/intents/create` recompute prices server-side and dedupe by transaction ref (R2).
3. **Scrub session/PII from logs.** Stop logging the session object (`auth.provider.tsx:189`) and phone numbers; they carry tokens/PII into device logs and Sentry.
4. **Get the Supabase layer into version control** — migrations, RLS policies, and RPC/edge-function source — so authorization can be reviewed and tested. The app is anonymous-by-default (`signInAnonymously`), so RLS must cleanly separate anonymous from authenticated users.
5. **Remove or guard the dev/test screens.** Gate `dev-test.tsx` / `test1.tsx` / `r.tsx` behind `__DEV__` or exclude them from production builds so privileged trip/payment functions aren't reachable in the shipped app.
6. **Add tests for auth, payment, and ride-hailing.** The harness and coverage gate already exist — point them at the business logic.
7. **Lock down the client Google keys.** **Confirm with team (Google Cloud):** that `GOOGLE_API_KEY` / `GOOGLE_PLACES_API_KEY` are app-restricted and quota-capped.
8. **Separate data access from the UI into one consistent layer (long-term maintainability).** Some screens/components/providers call Supabase directly — e.g. `active-trip-screen.tsx` runs `supabase.from('trips').select('status, rider_id, rider:profiles!rider_id(...)').eq('id', lastTripId).single()` inline, and the same happens in `wallet-view.tsx`, `cart-list-footer.tsx`, `gender-picker.tsx`, and the cart/location/auth providers (~9 files outside `src/api`). The app already has the right pattern — react-query hooks in `src/api/*` — so apply it everywhere: UI components consume hooks, and only the `api/` layer touches the Supabase client. This keeps queries testable, cacheable, and in one place, and stops data-shape and auth details from leaking into the view layer.
9. **Type the auth surface.** `auth.provider.tsx` uses `user/session/profile: any`; replace with Supabase's types and a defined profile type.
10. **Minor / team to handle (revisit later).** Remove the unused axios API client; drop the redundant payment polling (`use-payment-status-polling.ts` runs a realtime channel *and* a `setInterval`) and its leftover `console.log`; fix the `[...messing]` catch-all typo; confirm what `SECRET_KEY` (build-time) is used for. File naming is already consistent (all kebab-case) — no action. **Confirm with team** and update later.

---

## 4. Open questions for the team

- Is `qa-auth/login-bypass` deployed to the production Supabase project, and does it refuse there? This decides whether R1 is a non-issue or a critical backdoor.
- Does the backend enforce role/authorization on `accept_trip` / `complete_trip` / `create_trip_with_payment` / `user_cancel_trip`, so a customer-app caller (e.g. via the dev screens) can't drive them directly?
- Where do the RLS policies and RPC/edge-function definitions live, and are they tested?
- Do `payments/orders/create` and `payments/intents/create` recompute prices server-side and dedupe by transaction ref?
- Are the client Google keys application-restricted and quota-capped?
- Which parts of ride-hailing are actually live, and where does trip logic authoritatively run?
- What is `SECRET_KEY` (build-time) used for? It's validated but has no reference in `src/`.

---

## 5. Fix now

- Verify the QA bypass can't reach production, and strip it from release builds (R1).
- Confirm the payment edge functions re-validate order pricing (R2).
- Stop logging session tokens and phone numbers.
- Remove or gate the dev/test screens so privileged functions aren't reachable in the shipped app.

**Bottom line:** a well-scaffolded app with correct secure storage, a sensible env split, real CI, and payments handled the right way (server-mediated, unlike the rider app). Performance is fine. The two main risks are a QA auth bypass (account takeover if the edge function is live on prod) and order totals trusted from the client. Also worth addressing: session tokens written to logs, the dev/test screens that reach privileged functions, and an authorization model that lives entirely in unseen Supabase RLS (anonymous-by-default). Priorities: verify the bypass is inert in production, confirm the payment edge functions recompute prices, stop logging tokens, and get the Supabase layer under version control.
