# Komer Mobile Rider App — Codebase Assessment

**Scope:** high-level architecture review from reading the code only. We did **not** run the app, and the real backend logic (Supabase RLS policies, RPC/SQL functions, edge functions, and the MoMo server setup) is **not in this repo**, so anything depending on it is called out as unverified rather than asserted.

**Reviewed:** project structure and the Expo Router screens; `src/api/*` and providers; the payment flow (`payment.service.ts`, `use-payment-status-polling.ts`); env config and `app.config.ts`; token storage (`LargeSecureStore`); the driver location provider; and CI. **Not reviewed:** Supabase RLS, RPC/edge-function source, the MoMo server-side setup, production config, live traffic.

---

## 1. Overall architecture

Komer Rider is the **driver-side React Native / Expo app** (SDK 54, Expo Router, React 19) for the platform — ride-hailing trip acceptance, order pickup/delivery (QR "scan-delivery-code"), driver location tracking, wallet, and kredits. Same Obytes starter as the customer app. ~20k lines, 354 files, two main authors, last commit 2026-06-29.

The foundation is solid, and in a couple of places better than the customer app:

- **Env config is validated and split** (`env.js`), so build-time secrets stay out of the client — with one big exception (R1).
- **Token storage is done right** — `LargeSecureStore` keeps an AES key in SecureStore and the encrypted session in AsyncStorage.
- **The driver location provider is well built** — background tracking via `expo-task-manager`, an Android foreground service, and an offline ping queue (cap 100) gated on `NetInfo`. That's the right resilience for a driver app, and the strongest single piece of code in the project.
- **Good CI** (lint, type-check, translations, Jest), and unlike the customer app, **no QA auth bypass**, **no anonymous sessions**, and **no session/token logging**.

The defining problem is the payment integration: the app talks to MTN MoMo's API directly from the device and ships a privileged MoMo credential in the binary (R1). As with the rest of the platform, authorization also lives in Supabase RLS/functions outside this repo, and tests cover only UI primitives.

---

## 2. Biggest risks

These are the security and performance risks substantiable from the client code. Both are concrete (not conditional on the unseen backend).

### R1 — MoMo payment credentials and the collection flow live in the shipped client (security / financial)
`payment.service.ts` calls the MTN MoMo API directly with `'Ocp-Apim-Subscription-Key': Env.MOMO_SUBSCRIPTION_KEY`. That key is declared in the **client** env schema (`env.js:121`) and shipped into the binary via `app.config.ts` (`extra: { ...ClientEnv }`), so it's extractable from any installed APK/IPA. What's verified from the code: the key is in the bundle, and the app's own methods call the MoMo provisioning/collection endpoints on-device (`_createMomoApiUser`, `_createMomoApiKey`, `_createMomoToken`, `checkPaymentStatus`), passing the per-user MoMo `userId`/`apiKey` around the client. The exposure of the credential is concrete, not conditional — it's in the bundle.
- **Potential impact:** with the exposed subscription key, an attacker **could potentially** create MoMo API users, mint collection tokens, and invoke privileged Collection API endpoints under your subscription — leading to fraud, quota abuse, or problems with the payment provider. The exact reach **depends on the permissions tied to that key**, which we have **not** verified against MoMo's documentation. The sandbox/prod target is also client-controlled (`MOMO_TARGET_ENV`).
- **Fix:** move all MoMo interaction into a Supabase edge function / backend; the client should only ever send an order id and receive a status. Rotate the key (treat it as compromised).

### R2 — Unbounded payment-status polling (performance)
`use-payment-status-polling.ts` runs `setInterval` every 10s (the comment says "30 seconds" — mismatch) and only clears on `SUCCESSFUL`/`FAILED`. On `PENDING` or any error it polls **forever**, hitting the MoMo API directly with no max-attempt cap, backoff, or timeout.
- **Impact:** battery and data drain on drivers' phones, and sustained direct load on MoMo from every client; a stuck-`PENDING` payment polls indefinitely.

---

## 3. Recommendations (highest impact first)

1. **Move all MoMo interaction server-side, and rotate the key.** Replace the on-device provisioning/collection flow with an edge function; the client sends an order id and gets a status. Remove `MOMO_SUBSCRIPTION_KEY` from the client env. **Confirm with team:** is the shipped key a production key — if so, treat it as compromised and rotate now (R1).
2. **Get the Supabase layer into version control** — migrations, RLS policies, RPC/edge-function source — so authorization is reviewable. This matters most for driver-side authz: a driver must only accept and read trips assigned to them, which today lives entirely in unseen RLS.
3. **Bound the payment polling** — max attempts / total timeout, backoff, stop on repeated errors, and fix the 10s-vs-"30 seconds" mismatch. Better: drive status from a server push / realtime channel (R2).
4. **Add tests for the money/safety paths** — trip acceptance, delivery-code scan, location start/stop + offline-queue drain, payment status. The harness and coverage gate already exist.
5. **Lock down the client Google key.** **Confirm with team (Google Cloud):** that `GOOGLE_API_KEY` is app-restricted and quota-capped; it ships in the bundle like the MoMo key.
6. **Type the payment layer** — replace `paymentClient.post<any>` and the `any` error params with the declared `Momo*Response` types.
7. **Minor / team to handle (revisit later).** Delete the unused axios `client` (`API_URL`, no call sites); remove or gate the `test-dev.tsx` dev screen (it's benign but shipped) and fix the `[...messing]` catch-all typo; move the two direct-Supabase UI calls (`gender-picker.tsx`, `location.provider.tsx`) into the `api/` layer; gate payment/error `console.log`s on `__DEV__`; confirm what `SECRET_KEY` is used for; document the Supabase backend contract to reduce bus-factor (two authors own trip/payment/location). File naming is already consistent (all kebab-case) — no action. **Confirm with team** and update later.

---

## 4. Open questions for the team

- Why does the rider app collect payments via MoMo client-side at all? Can R1 be fixed by moving it server-side without breaking the delivery/COD flow?
- Is `MOMO_SUBSCRIPTION_KEY` a production key, and has it been rotated/restricted? A prod key in a released binary is compromised now.
- What permissions/scope does `MOMO_SUBSCRIPTION_KEY` actually grant, per MoMo's API? This determines the real blast radius of R1 and should be verified against MoMo's documentation.
- Where do the RLS policies and RPC/edge-function bodies live, and are they tested?
- How is trip assignment authorized server-side, so a driver can't accept or read trips that aren't theirs by calling Supabase directly?
- Is the client `GOOGLE_API_KEY` app-restricted and quota-capped?
- What is `SECRET_KEY` (build-time) used for? It's validated but has no reference in `src/`.

---

## 5. Fix now

- Move MoMo off the device and rotate the subscription key — the one urgent, concrete item (R1).
- Bound the payment polling so it can't run forever on PENDING/error (R2).
- Confirm the Google key restrictions in Google Cloud.
- Quick wins: delete the unused axios client; replace `post<any>` with the MoMo types; gate error logs on `__DEV__`; remove the dev screen.

**Bottom line:** a competently built driver app with a strong location layer, correct secure storage, validated env, real CI, and — unlike the customer app — no auth bypass or anonymous sessions. Performance has one real problem (unbounded payment polling); security has one concrete, potentially serious one: MTN MoMo credentials and the full collection flow ship in the client (the real blast radius depends on the key's scope, which is worth verifying against MoMo's docs). Fix MoMo server-side and rotate the key first, then get the Supabase authorization layer into version control and point the test harness at trip, payment, and location logic.
