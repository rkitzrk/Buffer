# Invest-Tracker — SDE Interview Prep (50 Questions)

**Context:** This is built from an actual read of your codebase (not a generic template). Invest-Tracker is a full-stack portfolio tracker:

- **Frontend:** React 18 + Vite, React Router v7, Tailwind CSS, Redux Toolkit (installed but effectively unused), Recharts + Tremor for charts, Framer Motion, Supabase JS client for auth.
- **Backend:** Node.js + Express, deployed as a Vercel serverless function (`api/index.js` → `backend/src/server.js`), Supabase (Postgres) as the database, accessed through a hand-rolled Active-Record-style wrapper (`models/Stock.js`, `models/Asset.js`).
- **External APIs:** Finnhub (stock quotes/search/candles), MetalPriceAPI (gold/silver), Marketaux (news).
- **Auth:** Supabase Auth issues a JWT; the frontend attaches it as a Bearer token; the backend verifies it with `supabase.auth.getUser(token)` in `middleware/auth.js`.
- **Deployment:** Two separate Vercel projects (frontend static site + backend serverless functions) from one monorepo.

Since you're a Codeforces Specialist, your DSA is already strong — these questions deliberately skew toward **system design, real-world engineering judgment, debugging, and "why," not algorithms**, because that's what SDE interviews use project-based questions to test. Several questions reference *actual bugs and design smells found in your repo* — knowing these cold is a strong signal in interviews ("tell me about a bug you found/fixed").

Format per question:
- **Polished answer** — what to say
- **TL;DR** — one-line version for rapid recall
- **Key mappings** — exact file(s)/function(s) to point to if asked "show me"

---

## TOP 10 — Must-know, will almost certainly come up

### Q1: Walk me through the full architecture of this project — every layer, end to end.

**Polished answer:** Invest-Tracker is a three-tier app deployed as two independent services. The **frontend** is a Vite-built React SPA using React Router for client-side routing and the Supabase JS SDK directly in the browser for authentication (sign up, sign in, session refresh). The **backend** is an Express app that is *not* run as a traditional long-lived server in production — it's wrapped as a single Vercel serverless function (`api/index.js` imports the Express `app` and exports it; Vercel's Node runtime calls it per-request). The backend talks to **Supabase Postgres**, but instead of an ORM like Prisma/Sequelize, it uses the Supabase JS client's query builder directly, wrapped in two small model files (`Stock.js`, `Asset.js`) that mimic Active Record (`findAll`, `findByPk`, `create`, plus `update`/`destroy` closures attached to fetched rows). For external data, the backend proxies three third-party APIs — Finnhub for stock quotes, MetalPriceAPI for precious metals, Marketaux for news — so API keys never reach the browser. Auth is JWT-based: Supabase issues a token to the browser, the frontend's axios interceptor attaches it as `Authorization: Bearer <token>` on every request, and an Express middleware re-verifies that token server-side before touching any data. This separation (auth provider handles identity, your backend handles authorization + business data) is a very standard modern SaaS pattern worth naming explicitly.

**TL;DR:** React SPA (Vite) → Express API (as a single Vercel serverless function) → Supabase Postgres, with Supabase Auth issuing JWTs verified per-request, and the backend proxying three external market-data APIs so keys stay server-side.

**Key mappings:** `frontend/src/App.jsx`, `backend/src/server.js`, `api/index.js`, `backend/src/models/*.js`, `backend/src/middleware/auth.js`

---

### Q2: Why deploy frontend and backend as two separate Vercel projects instead of one? What broke when this Express app was first deployed to serverless, and how was it fixed?

**Polished answer:** Vercel's frontend hosting (static build output) and its serverless Node functions have different build/runtime models, so splitting them into two projects — one with root directory `frontend/` (Vite preset) and one with root directory `backend/` (Node runtime) — gives each its own build pipeline, environment variables, and independent scaling/redeploys. The real engineering story here is the migration bug: the Express app originally called `app.listen(PORT)` unconditionally at the bottom of `server.js`. That's correct for a traditional always-on server, but a Vercel serverless function doesn't "run" persistently — the platform imports your exported handler (the Express `app` object) and invokes it per request; a function that also tries to bind and hold open a TCP port either crashes cold starts or leaks handles across invocations. The fix was to guard it: `if (process.env.NODE_ENV !== 'production' || process.env.VERCEL !== '1') { app.listen(...) }`, so `app.listen()` only runs when you run `node src/server.js` locally, while `export default app` is what Vercel actually uses. This is a great "impedance mismatch between frameworks and deployment models" story for interviews — it shows you understand that serverless functions are *stateless, per-invocation* execution, not long-running processes.

**TL;DR:** Split by build model (static vs Node runtime); the real bug was `app.listen()` firing inside a serverless function — fixed by gating it behind a `VERCEL` env check so only `export default app` is used in production.

**Key mappings:** `backend/src/server.js` (bottom), `api/index.js`, `walkthrough.md` bug #3

---

### Q3: Explain authentication end-to-end — what happens from "user clicks login" to "user sees their portfolio"?

**Polished answer:** The user submits email/password in the React app; `AuthContext.signIn()` calls `supabase.auth.signInWithPassword()`, which talks directly to Supabase's Auth service from the browser (the backend is not involved in login at all). Supabase returns a session containing an access JWT, which the Supabase JS SDK persists (it manages storage/refresh automatically since the client is configured with `persistSession: true, autoRefreshToken: true`). `AuthContext` also subscribes to `supabase.auth.onAuthStateChange` so any tab-level session change (login, logout, token refresh) updates React state reactively. From here, every API call goes through a shared axios instance (`services/api.js`) whose **request interceptor** calls `supabase.auth.getSession()` and attaches `Authorization: Bearer <access_token>` before the request leaves the browser. On the backend, `middleware/auth.js`'s `authenticateUser` pulls that header, strips the `Bearer ` prefix, and calls `supabase.auth.getUser(token)` — this re-validates the JWT's signature and expiry against Supabase's auth service (crucially, this is *not* just decoding the JWT client-side; it's a real server-side verification), then attaches the resolved `user` object to `req.user` for every downstream route handler to use as the source of truth for "who is making this request."

**TL;DR:** Browser logs in directly against Supabase Auth → JWT stored client-side → axios interceptor attaches it as a Bearer token → Express middleware re-verifies it server-side via `supabase.auth.getUser()` → `req.user` becomes the trusted identity for the rest of the request.

**Key mappings:** `frontend/src/context/AuthContext.jsx`, `frontend/src/services/api.js` (interceptor), `backend/src/middleware/auth.js`

---

### Q4: Walk through what happens, line by line, when a user adds a new stock to their portfolio.

**Polished answer:** In `Portfolio.jsx` (or `AddStockModal.jsx`), the form collects `name`, `ticker`, `shares`, `buy_price`, `target_price` and calls `api.post('/stocks', requestData)`. The axios interceptor attaches the JWT. On the backend, `POST /` in `stocks.js` runs behind `authenticateUser`, so `req.user.id` is trusted. It parses the numeric fields with `parseFloat`, then calls `stockPriceService.getStockQuote(ticker)`, which hits Finnhub's `/quote` endpoint with a 7-second timeout. Here's the interesting design decision: if Finnhub fails or returns a non-positive price (`quote?.c && quote.c > 0`), the code **falls back to the user's own `buy_price`** rather than failing the request — a deliberate resilience trade-off (better to save the stock with an approximate price than to block the user because a third-party API hiccupped). It then builds a `stockData` object, forcibly setting `user_id: req.user.id` from the *verified* token (never from the request body — this matters for security, see Q26), and calls `Stock.create()`, which does a Supabase `insert().select().single()` and returns the new row. The frontend then closes the modal and calls `fetchStocks()` again to refresh the full list (see Q45 for why that's a debatable choice).

**TL;DR:** Form → axios (with JWT) → `authenticateUser` middleware → fetch live price from Finnhub with a graceful fallback to `buy_price` on failure → insert row with server-trusted `user_id` → frontend refetches the whole list.

**Key mappings:** `frontend/src/components/modals/AddStockModal.jsx`, `frontend/src/pages/Portfolio.jsx`, `backend/src/routes/stocks.js` (`POST /`), `backend/src/services/stockPriceService.js`

---

### Q5: The models (`Stock.js`, `Asset.js`) look like a mini-ORM. Explain the pattern and why they built it instead of using a real ORM like Prisma/Sequelize/TypeORM.

**Polished answer:** This is a hand-rolled **Active Record pattern** on top of the Supabase JS query builder. `findAll(options)` translates a generic `{ where, order }` object into chained `.eq()`/`.gt()`/`.lt()`/`.order()` calls; `findByPk(id)` fetches a single row by primary key and — this is the clever bit — attaches `update()` and `destroy()` as closures on the returned object, so callers can write `stock.update({...})` as if it were a real ORM instance, even though under the hood it's just another Supabase call scoped by the captured `id`. The reason to build this instead of pulling in Prisma/Sequelize is almost certainly because Supabase already *is* a full Postgres-backed REST/RPC client with built-in auth and row-level querying — adding a second ORM on top would mean maintaining two different ways of talking to the same database (connection pooling, migrations, types), and in a serverless environment, ORMs with persistent connection pools (like Prisma's default) are notoriously painful (connection exhaustion across cold starts). The trade-off: you lose type safety, migrations-as-code, and relational eager-loading that a real ORM gives you, and you re-implement (partially) an ORM's API surface by hand, which is more code to maintain long-term. A good follow-up point: this hand-rolled wrapper is also incomplete — e.g., there's no `findOne`, no transactions, no relational joins beyond what Supabase's `.select()` embedding syntax could offer.

**TL;DR:** Active Record pattern manually built on Supabase's query builder — avoids running two DB clients/connection strategies in serverless, at the cost of type-safety, migrations, and relational features a real ORM gives you.

**Key mappings:** `backend/src/models/Stock.js`, `backend/src/models/Asset.js`

---

### Q6: Explain CORS in this project — what it is, why it's needed, and how it's configured here.

**Polished answer:** CORS (Cross-Origin Resource Sharing) is a browser-enforced security mechanism: JavaScript running on `frontend-domain.vercel.app` cannot call `backend-domain.vercel.app` by default, because they're different origins (different subdomain = different origin). The server must explicitly opt in by sending `Access-Control-Allow-Origin` headers naming which origins are allowed. In `server.js`, `allowedOrigins` is a hardcoded array of known frontend URLs (production + two localhost ports for dev), plus a dynamic entry pushed from `process.env.FRONTEND_URL` — this is the fix noted in the walkthrough (bug #6), because hardcoding meant redeploying backend code every time the frontend got a new domain. The `cors` middleware is applied globally, and `app.options('*', cors(corsOptions))` explicitly handles preflight `OPTIONS` requests (which browsers send automatically before any "non-simple" request — e.g. one with a custom `Authorization` header or `Content-Type: application/json` — to ask "are you allowed to do this?" before sending the real request). Good interview point: distinguish CORS (a *browser-side* restriction that protects users, not really the server) from server-side auth (which actually protects data) — a request from Postman/curl bypasses CORS entirely since CORS is enforced by browsers, not by the API itself.

**TL;DR:** CORS lets the browser allow the frontend origin to call the backend origin; configured with a whitelist array plus an env-var override, and `OPTIONS` preflight is handled explicitly — but CORS is a browser protection, not a substitute for real auth.

**Key mappings:** `backend/src/server.js` (`allowedOrigins`, `corsOptions`)

---

### Q7: This backend uses two different Supabase keys — an "anon" key on the frontend and a "service role" key on the backend. Why, and what would go wrong if you mixed them up?

**Polished answer:** Supabase issues two classes of API keys for a project: the **anon key**, meant to be public (shipped inside the frontend bundle, visible in devtools), which is safe *only* if Row-Level Security (RLS) policies are configured on the database to restrict what an anonymous/authenticated client can see or write; and the **service role key**, which bypasses RLS entirely and has full read/write access to every row in every table — this must never leave the server. In this project, the frontend's `config/supabase.js` uses `VITE_SUPABASE_ANON_KEY` (Vite only exposes env vars prefixed `VITE_` to the client bundle — anything without that prefix stays server-only, which is itself worth mentioning), while the backend's `SUPABASE_KEY` (service role) is used to run privileged queries. Notice the backend's `stocks.js` route manually filters `.eq('user_id', req.user.id)` on every query — that's *compensating* for the fact that the service-role key has no automatic per-user restriction; the backend code itself is the only thing enforcing "you can only see your own stocks." If you accidentally used the service role key in the frontend bundle, anyone could open devtools, extract it from the JS source, and query/modify *any* user's data directly against Supabase — a full data breach. If you used the anon key on the backend without RLS policies configured, backend writes could silently fail or be over-restricted.

**TL;DR:** Anon key (public, RLS-dependent) on the frontend vs service-role key (full access, secret) on the backend; the backend manually scopes every query by `user_id` because the service-role key itself grants no automatic per-user isolation.

**Key mappings:** `frontend/src/config/supabase.js`, `backend/src/config/supabase.js`, `backend/src/routes/stocks.js` (`.eq('user_id', req.user.id)`)

---

### Q8: There's a real bug in the `PUT /stocks/:id` handler. Find it and explain how you'd fix it — this is the kind of thing an interviewer will ask you to spot live.

**Polished answer:** Look at the route:
```js
router.put('/:id', authenticateUser, async (req, res) => {
  const stock = await Stock.findByPk(req.params.id);
  if (!stock || stock.user_id !== req.user.id) {
    return res.status(403).json({ error: 'Access denied' });
  }
  await stock.update({ ...req.body, last_updated: new Date() });
  res.json(stock);   // <-- BUG
});
```
`stock.update(...)` performs the Supabase update and *returns* the freshly updated row (see `Stock.js`: `update()` resolves to `updatedData`) — but that return value is discarded. `res.json(stock)` then sends back the **original, pre-update object** captured before the write, not the new state. The frontend receiving this response would render stale data (e.g., if `Portfolio.jsx` used the response directly instead of always refetching). The fix is trivial but important: `const updated = await stock.update({...}); res.json(updated);`. This is exactly the kind of subtle "the write succeeded but the response lies about it" bug that's easy to introduce and easy to miss in code review — a great one to bring up if asked "tell me about a bug you found by reading code," because it demonstrates you actually traced data flow rather than skimming.

**TL;DR:** `PUT /:id` calls `stock.update()` but discards its return value and responds with the stale pre-update object — fix by capturing and returning the updated row.

**Key mappings:** `backend/src/routes/stocks.js` (`router.put('/:id', ...)`), `backend/src/models/Stock.js` (`update` closure)

---

### Q9: `stockPriceService.js` has an `updateStockPrices()` function that loops through every stock and calls `getStockQuote()` with a 1-second sleep between each. Why is this a scalability problem, and how would you redesign it?

**Polished answer:** This function fetches every user's every stock/watchlist row, then, in a plain `for...of` loop, calls the Finnhub API sequentially with `await new Promise(r => setTimeout(r, 1000))` between calls — clearly there to respect Finnhub's rate limit (their free tier caps requests/minute). The scalability problem is that this is **O(n) wall-clock time with a hard floor of 1 second per stock** — with 500 tracked tickers across all users, that's over 8 minutes for a single pass, and it only gets worse as the user base grows. There's a deeper problem too: this function is only ever wired up via `startPeriodicUpdates()`, which calls `setInterval(updateStockPrices, 5 * 60 * 1000)` — but `setInterval` requires a long-lived process to keep firing, and **this function is never actually invoked anywhere in `server.js`**, and even if it were, a Vercel serverless function is spun up per-request and torn down; it doesn't sit around running a `setInterval` for you. So this "periodic background job" pattern is fundamentally incompatible with the serverless deployment model it's shipped in. A correct redesign: (1) **deduplicate tickers** — many users likely hold overlapping symbols (AAPL, TSLA), so you should fetch each unique ticker once and fan the price out to all rows referencing it, not once per row; (2) use Finnhub's **batch/webhook** capabilities if available, or a provider with true real-time push (websocket) pricing; (3) move the actual scheduling off serverless entirely — a **Vercel Cron Job** (which *can* invoke a serverless endpoint on a schedule) or an external worker (a small always-on Node process, or a queue-based system like a cron-triggered Lambda + SQS) that batches requests and writes results back to Supabase; (4) cache quotes with a short TTL (e.g., in Redis or even a `price_cache` table) so multiple users reading the same ticker within a few seconds don't each trigger a fresh external call.

**TL;DR:** Sequential per-row fetching with 1s sleeps doesn't scale, and `setInterval`-based scheduling is incompatible with serverless anyway (and isn't even invoked); redesign with ticker de-duplication, a real scheduler (Vercel Cron / external worker), and a short-TTL price cache.

**Key mappings:** `backend/src/services/stockPriceService.js` (`updateStockPrices`, `startPeriodicUpdates`), `backend/src/server.js` (note: `startPeriodicUpdates` is imported but never called)

---

### Q10: If you had to scale this app to millions of users, what would you change, in priority order?

**Polished answer:** I'd frame this as a layered answer, cheapest/highest-impact first. **(1) Caching hot reads:** stock quotes are read far more often than they change — introduce a cache (Redis, or even Supabase itself with a `last_updated`-aware TTL check) so identical tickers requested by different users within a short window hit cache, not Finnhub, directly reducing both latency and third-party rate-limit pressure. **(2) Deduplicate external calls:** as in Q9, fetch unique tickers once, not once per user-row. **(3) Move price refresh off the request path entirely:** background job (cron-triggered) updates a shared `prices` table; user-facing routes only ever read from Postgres, never block on a live third-party call. **(4) Database indexing:** add indexes on `stocks(user_id)`, `stocks(ticker)`, and a composite on `(user_id, is_in_watchlist, shares)` since that's the exact filter used in the main list query. **(5) Pagination:** `GET /stocks` currently returns the full unpaginated list per user — fine at small scale, but should support cursor/offset pagination for users with large portfolios. **(6) Connection/serverless considerations:** Supabase's REST layer (PostgREST) handles connection pooling for you, which is actually a point in favor of the current architecture at scale (vs. a raw Postgres driver + serverless, which would need something like PgBouncer). **(7) Real-time updates:** move from polling (`fetchStocks()` on mount / manual refresh) to Supabase's real-time subscriptions or WebSockets, so price updates push to clients instead of requiring refetch. **(8) Rate limiting your own API:** add per-user rate limiting on the backend (e.g., the `/search` autocomplete endpoint, which fires on every keystroke — see Q46 — could be abused or simply overload Finnhub).

**TL;DR:** Cache + dedupe external calls, move price refresh to a background job off the request path, add DB indexes and pagination, move to push-based (real-time) updates instead of polling, and rate-limit the API itself.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/services/stockPriceService.js`, `frontend/src/pages/Dashboard.jsx` / `Portfolio.jsx`

---

## TOP 11–25 — Very likely to come up, strong follow-ups to the above

### Q11: The project installs Redux Toolkit and configures a store, but `stockSlice.js` is empty. What does this tell you, and what would you say if asked about it?

**Polished answer:** `store/store.js` does `configureStore({ reducer: todoReducer })` importing from `./stockSlice`, but `stockSlice.js` is a completely empty file — so `todoReducer` is `undefined`, meaning the Redux store is either non-functional or would throw/behave unpredictably if actually used, and in practice **no component in the app dispatches or selects from this store at all**; all state (stocks, assets, theme, auth) is managed with local `useState` and two Context providers (`AuthContext`, `DarkModeContext`). This is dead scaffolding — likely added early with the intention of centralizing state, then abandoned once Context proved sufficient for this app's actual complexity. In an interview, the honest and *correct* engineering answer is: "Redux wasn't needed here — the state is either local to a page (form inputs, loading flags) or genuinely global-but-simple (auth session, theme), both of which Context/local state handle fine without Redux's boilerplate. I'd remove the unused dependency and the dead store file rather than leave partially-wired infrastructure in the codebase, since unused code creates confusion for the next engineer who assumes it's live." This is also a good moment to state your own mental model for *when* Redux earns its cost: many components need the same frequently-updated state, with complex update logic, and you want time-travel debugging/middleware (e.g., logging, undo) — none of which applies to a portfolio dashboard this size.

**TL;DR:** `stockSlice.js` is empty, so the configured Redux store is dead/unused code — the app actually runs on Context + local state; know when Redux is worth its overhead vs when Context is enough.

**Key mappings:** `frontend/src/store/store.js`, `frontend/src/store/stockSlice.js` (empty), `frontend/src/context/AuthContext.jsx`, `frontend/src/context/DarkModeContext.jsx`

---

### Q12: `Dashboard.jsx` has multiple `useEffect` hooks syncing theme via a prop, `localStorage`, a custom `themeChange` event, *and* the native `storage` event. Critique this design.

**Polished answer:** There are effectively four theme-sync mechanisms layered on top of each other: (1) `theme` starts from a `propTheme` passed down from `App.jsx`; (2) a `useEffect` re-syncs `theme` whenever `propTheme` changes; (3) a listener on a custom `window` event `themeChange` *and* a `document` event `themeChanged` (two differently-named events doing similar things — likely a naming drift bug from refactoring); (4) a listener on the native `storage` event, which only fires in **other tabs**, not the tab that made the change (a common `localStorage` gotcha), so it's paired with manually re-reading `localStorage.getItem('theme')` inside the handler. This is over-engineered for what it's solving: theme is really just one piece of shared state that changes rarely and needs to reach every component. The idiomatic React fix is to lean entirely on `DarkModeContext` (which already exists in the codebase but isn't fully used here) — put `theme`/`toggleDarkMode` in one Context, consume it via `useDarkMode()` wherever needed, and drop the custom-event/`storage`-event machinery completely; Context re-renders subscribers automatically without manual event wiring, same-tab or cross-tab (if cross-tab sync is truly needed, a lightweight library like `broadcast-channel` or a single well-named event, not two, is the answer). This is a good example to discuss "signs of organic growth without refactoring" — the *symptom* is redundant listeners, the *root cause* is not having a single owner for shared state.

**TL;DR:** Theme state is synced through four redundant mechanisms (prop, two differently-named custom events, and the native `storage` event) instead of one Context — a textbook "state has no single owner" smell; consolidate into `DarkModeContext`.

**Key mappings:** `frontend/src/pages/Dashboard.jsx` (multiple `useEffect` theme hooks), `frontend/src/App.jsx`, `frontend/src/context/DarkModeContext.jsx`

---

### Q13: `ProtectedRoute.jsx` redirects unauthenticated users away from `/dashboard`. Does this actually secure anything?

**Polished answer:** No — and this is an important distinction to be able to articulate clearly. `ProtectedRoute` only controls **client-side rendering**: if `user` is null, it renders `<Navigate to="/" />` instead of the page component. But the JavaScript bundle containing `Dashboard.jsx`, `Portfolio.jsx`, etc. is still fully downloaded to the browser regardless (it's a static SPA — there's no server deciding what to send based on auth), and nothing stops someone from calling the backend API directly with curl/Postman, bypassing the React app entirely. The **actual** security boundary is server-side: `authenticateUser` middleware on every sensitive Express route, which verifies the JWT and scopes every Supabase query with `.eq('user_id', req.user.id)`. `ProtectedRoute` exists purely for **UX** — don't show a logged-out user a confusing dashboard full of API errors, redirect them to login instead. The correct mental model to state in an interview: *client-side route guards are UX, server-side request auth is security; never rely on the former to protect data.*

**TL;DR:** `ProtectedRoute` is a UX nicety (avoids rendering broken pages to logged-out users); real security is the backend's `authenticateUser` middleware + per-request `user_id` scoping, since the SPA bundle and any direct API call bypass client-side guards entirely.

**Key mappings:** `frontend/src/components/auth/ProtectedRoute.jsx`, `backend/src/middleware/auth.js`

---

### Q14: Explain the axios interceptor pattern used in `services/api.js`, and why `window.location.href = '/'` on a 401 is a questionable choice in a React Router app.

**Polished answer:** Axios interceptors let you hook into every request/response globally instead of repeating logic at every call site. The **request interceptor** here awaits `supabase.auth.getSession()` on every outgoing call and attaches the fresh access token — this matters because tokens expire and Supabase's SDK auto-refreshes them, so re-reading the session per-request (rather than caching a token once) guarantees you're always sending a valid one. The **response interceptor** watches for `status === 401` and does a full `window.location.href = '/'`. In a React Router SPA, that's a hard browser navigation — it discards the entire in-memory React state and JS heap, forces a full page reload (new bundle fetch, new mount, everything re-initializes), whereas the idiomatic approach in a router-based app is to use the router's imperative `navigate('/')` (from a hook or a router instance held outside components) so the SPA stays "alive" and you can preserve things like a "redirect back here after login" query param, or show a toast ("session expired") before navigating. The hard-redirect approach isn't *wrong* exactly — it's simple and guarantees a totally clean state — but it's a heavier hammer than necessary and loses any nuance you might want (e.g., distinguishing "token expired, please log back in" from "you don't have access to this resource," which is also often a 401/403 conflation worth calling out).

**TL;DR:** Interceptors centralize auth-token-attachment and 401-handling; the 401 handler using `window.location.href` forces a full page reload instead of an in-SPA `navigate()`, losing app state and the ability to show a graceful "session expired" UX.

**Key mappings:** `frontend/src/services/api.js`

---

### Q15: `AddStockModal.jsx` creates its own separate axios instance instead of importing the shared one from `services/api.js`. Why is that a problem, and how would you fix it?

**Polished answer:** `services/api.js` exports a pre-configured axios instance with request/response interceptors (auth token attachment, 401 redirect). `AddStockModal.jsx` instead does `const api = axios.create({ baseURL: API_BASE_URL, timeout: 10000 })` locally — a second instance with **no interceptors at all**. That means calls made from this modal never get the `Authorization` header attached automatically; looking at the actual `handleSubmit`, it works anyway only because... it doesn't attach a token, and the backend route it calls requires `authenticateUser` — so this modal's direct API calls would actually fail with 401 in practice unless something else is compensating, which is exactly the kind of silent, hard-to-reproduce bug ("works from the Portfolio page, fails from this modal") that duplicated setup causes. The fix is straightforward: delete the local `axios.create(...)` block and `import api from '../../services/api'` instead, so there's a single source of truth for base URL, timeout, and auth wiring. This is a good general lesson to state explicitly: **shared cross-cutting concerns (auth headers, base URLs, error handling, timeouts) belong in exactly one place**; every duplicate risks drifting out of sync as the "real" one evolves and nobody remembers to update the copy.

**TL;DR:** A second, interceptor-less axios instance in `AddStockModal.jsx` duplicates (and silently loses) the auth-header logic that the shared instance provides — fix by importing the shared `api` instance everywhere instead of re-creating axios clients ad hoc.

**Key mappings:** `frontend/src/components/modals/AddStockModal.jsx` (local `axios.create`), `frontend/src/services/api.js`

---

### Q16: Find the divide-by-zero / NaN risks in `Dashboard.jsx`'s portfolio math, and explain how you'd guard against them.

**Polished answer:** Two spots stand out. First: `percentageGain = (totalGain / (totalValue - totalGain)) * 100` — `totalValue - totalGain` is effectively the total cost basis; if the user has no holdings (`totalValue` and `totalGain` both `0`), this is `0/0 = NaN`, which would render as "NaN%" in the UI rather than something sane like "0%" or a friendly empty state. Second: `Avg. Cost Basis: $... / stocks.reduce((sum, stock) => sum + stock.shares, 0)).toFixed(2)` divides total cost by total shares — if `stocks` is empty, that denominator is `0`, again `NaN`, and calling `.toFixed(2)` on `NaN` just prints the string `"NaN"`. Both bugs share a root cause: the code assumes a non-empty portfolio and never checks for the empty-state / zero-division case before doing arithmetic. The fix pattern is consistent: guard the denominator (`totalValue - totalGain > 0 ? ... : 0`), and more broadly, render an explicit "You don't have any holdings yet — add your first stock" empty state when `stocks.length === 0` rather than trying to compute stats over nothing — that's both a correctness fix and a better UX than a numeric edge case leaking into the UI.

**TL;DR:** `percentageGain` and the avg-cost-basis calculation both divide by values that can be `0` for a new/empty portfolio, producing `NaN` in the UI — guard with explicit zero-checks and an empty-state UI instead of computing over an empty array.

**Key mappings:** `frontend/src/pages/Dashboard.jsx` (`totalValue`, `totalGain`, `percentageGain`, "Avg. Cost Basis" JSX)

---

### Q17: Walk through how gold/silver pricing works, and why using floating-point numbers for financial values is risky in general.

**Polished answer:** `assets.js`'s `POST /` route maps `name === 'Gold'` to ticker `XAU` (else `XAG` for silver), calls `assetPriceService.getAssetQuote(ticker)` against MetalPriceAPI's `/latest` endpoint (which returns conversion rates relative to USD), and then does `quote.rates.USDXAU / 28.34` — dividing by 28.34952 grams per troy ounce (hardcoded, slightly imprecise — the real constant is ~31.1035g, so this is actually using the wrong conversion factor, likely confusing grams-per-troy-ounce with something else; worth flagging as a real correctness bug if it comes up) to get a price-per-gram. More generally: every price in this schema (`buy_price`, `current_price`, `target_price`) is a JS `number` (IEEE-754 double), and JS floating-point arithmetic is famously imprecise for base-10 decimals (`0.1 + 0.2 !== 0.3`) because binary floats can't exactly represent most decimal fractions. For a hobby portfolio tracker displaying approximate gains, this is a minor cosmetic risk (rounding to `.toFixed(2)` usually hides it). But the standard practice for anything handling real money at scale is either **integer cents** (store `109999` meaning $1099.99, do all arithmetic in integers, divide by 100 only at display time) or a proper **decimal/fixed-point type** (Postgres `numeric`, or libraries like `decimal.js`), because floating-point drift compounds across many transactions and can cause off-by-a-cent discrepancies that matter a lot more in, say, a real brokerage's ledger than in a display-only tracker.

**TL;DR:** Metals pricing converts a MetalPriceAPI rate to price-per-gram via a troy-ounce constant (worth double-checking — 28.34 looks like the wrong figure); more broadly, storing money as JS floating-point numbers risks precision drift — production financial systems use integer cents or a decimal type instead.

**Key mappings:** `backend/src/routes/assets.js` (`POST /`), `backend/src/services/assetPriceService.js`

---

### Q18: The `stocks` table is used for both "holdings" and "watchlist" via an `is_in_watchlist` boolean and a `shares > 0` filter. Critique this schema design.

**Polished answer:** Every stock-fetching query filters `gt('shares', 0).eq('is_in_watchlist', false)` to mean "actual holdings," implying watchlist items live in the *same table*, distinguished by a boolean flag (and presumably `shares = 0` for pure watchlist entries, though that's inferred, not enforced anywhere I can see — nothing stops a row with `shares > 0 AND is_in_watchlist = true` existing simultaneously, which is an ambiguous state the schema doesn't prevent). This is a classic "overloaded table" smell: two conceptually different entities (something you *own* vs something you're *watching*) sharing one table via a flag, which means every query has to remember to apply the right filter combination, and the shape of a "holding" row (needs `shares`, `buy_price`) doesn't cleanly match a "watchlist" row (doesn't need either). The more normalized design: separate `holdings` and `watchlist` tables (or a `watchlist` table referencing `ticker` + `user_id` only, since a watchlist item doesn't need cost-basis fields at all), possibly both referencing a shared `securities` lookup table (ticker → name/exchange metadata) to avoid storing the same `name`/`ticker` pair redundantly per user. This is a strong system-design talking point: recognizing when a boolean flag is secretly modeling "this row means two different things" is exactly the kind of schema smell interviewers probe for.

**TL;DR:** Holdings and watchlist entries share one `stocks` table distinguished by `is_in_watchlist` + `shares > 0`, an overloaded/ambiguous design; a normalized schema would split them into separate tables (watchlist doesn't even need cost-basis columns).

**Key mappings:** `backend/src/routes/stocks.js` (`.gt('shares', 0).eq('is_in_watchlist', false)` pattern repeated across routes)

---

### Q19: How does Supabase's query builder (`.eq()`, `.gt()`, `.order()`) map to actual SQL, and where would you add indexes?

**Polished answer:** Supabase's client is a thin wrapper over **PostgREST**, which translates these chained method calls into a single HTTP request that PostgREST turns into SQL — `.from('stocks').select('*').eq('user_id', x).gt('shares', 0).eq('is_in_watchlist', false).order('name')` becomes roughly `SELECT * FROM stocks WHERE user_id = x AND shares > 0 AND is_in_watchlist = false ORDER BY name ASC`. Every one of those `WHERE` clauses is a candidate for indexing once the table grows. I'd add: a plain B-tree index on `stocks(user_id)` since *every* query filters by it (this is the single highest-value index — without it, every request is a full table scan as the table grows); a composite index on `(user_id, is_in_watchlist, shares)` to cover the exact filter combination used repeatedly, letting Postgres satisfy the whole `WHERE` clause from the index without touching the heap; and an index on `ticker` if/when you add ticker-level aggregate queries (e.g., "how many users hold AAPL" for the price-dedup optimization from Q9/Q10). I'd explicitly mention using `EXPLAIN ANALYZE` in Postgres to verify an index is actually being used rather than assuming — a query planner won't always pick an index if the table is small enough that a sequential scan is cheaper, which is worth knowing so you don't "add indexes" blindly.

**TL;DR:** Supabase's chained filters compile to a single SQL query via PostgREST; the highest-value index here is `stocks(user_id)` (filtered on every request), followed by a composite covering `(user_id, is_in_watchlist, shares)` — verify with `EXPLAIN ANALYZE`, don't assume.

**Key mappings:** `backend/src/models/Stock.js` (`findAll`), `backend/src/routes/stocks.js` (repeated `.eq('user_id', ...)` filters)

---

### Q20: Explain the environment-variable split between `VITE_`-prefixed and plain variables, and why that prefix convention exists.

**Polished answer:** Vite only inlines environment variables into the client bundle if they're prefixed `VITE_` (e.g., `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_API_BASE_URL`) — this is a deliberate safety convention: build tools that bundle *all* `.env` variables into client code by default have historically leaked secrets (database passwords, private API keys) straight into public JS bundles. By requiring an explicit prefix, Vite makes "this is meant to be public" an opt-in decision rather than an accident. On the backend side, `dotenv.config()` loads plain-named variables (`SUPABASE_KEY`, `FINNHUB_API_KEY`, `NEWS_API_KEY`, `METAL_API_KEY`) that are never bundled anywhere — they only exist in the Node process's `process.env`, invisible to any browser. So the project correctly keeps the service-role Supabase key and every third-party API key server-side only, while the anon key and API base URL (both safe to expose) go through the `VITE_` channel. Worth mentioning as a general practice: even "safe to expose" values like the anon key should still be *scoped* correctly server-side via RLS (see Q7) — the prefix convention protects against *accidental* leaks, not against a key being inherently over-privileged.

**TL;DR:** Vite bundles only `VITE_`-prefixed env vars into client code (an intentional secret-leak guard); everything else stays server-side-only via plain `dotenv` — the project correctly keeps API keys and the service-role key off the client.

**Key mappings:** `frontend/.env.example`, `backend/.env.example`, `frontend/vite.config.js`, `frontend/src/config/supabase.js`

---

### Q21: Walk through the stock ticker autocomplete/search feature and its exchange-filtering fallback logic.

**Polished answer:** `GET /stocks/search?q=...` proxies Finnhub's `/search` endpoint, then applies a **three-tier fallback filter**: first it tries to keep only results whose `primaryExchange` is one of `NASDAQ`, `NYSE`, `BATS`, `ARCA` (i.e., prefer major US exchanges so autocomplete doesn't surface obscure foreign tickers for a US-focused query); if that filter empties the result set, it falls back to filtering by `type === 'Common Stock'` (looser — accepts any exchange as long as it's a plain equity, not e.g. a warrant or bond); if *that's* still empty, it falls back to the loosest filter (`symbol && description` present at all), then truncates to the top 10. This progressive-relaxation pattern is a solid UX call — better to show *some* plausible results than a blank dropdown, while still preferring the highest-quality matches when available. A good follow-up to raise proactively: this endpoint has no debounce on the frontend caller (see Q46), so if it's wired to an `onChange` handler, every keystroke fires a fresh Finnhub request — that's worth flagging even though the filtering logic itself is well thought out.

**TL;DR:** Search results are filtered in three progressively looser tiers (major exchanges → any common stock → anything with a symbol/description) so autocomplete degrades gracefully instead of showing nothing — but the frontend caller has no debounce.

**Key mappings:** `backend/src/routes/stocks.js` (`GET /search`)

---

### Q22: The routes file includes `test-finnhub/:ticker` and `test-search/:query` endpoints that echo raw third-party responses and API-key presence. Why is leaving these in production a problem?

**Polished answer:** These look like debug scaffolding left over from initial Finnhub integration — they call Finnhub directly and return the raw response, and on error they include `apiKey: process.env.FINNHUB_API_KEY ? 'Present' : 'Missing'` in the JSON response. Two concerns: first, these are **unauthenticated** (no `authenticateUser` middleware), so anyone who discovers the route can burn through your Finnhub rate limit for free, or use your server as a proxy to probe Finnhub's API on your dime; second, even though it only reports presence/absence rather than the key's value, leaking *any* information about server configuration to unauthenticated callers is unnecessary attack-surface — it tells a would-be attacker your integration details for free. The fix is simple: delete these routes before shipping to production, or at minimum gate them behind `if (process.env.NODE_ENV !== 'production')` so they only exist in local/dev environments, and never leak key-presence information in any response, authenticated or not. This is a good general principle to voice: **debug/test endpoints are a common and easy-to-overlook source of production leaks** — a checklist item before any deploy should be "did any `test-`, `debug-`, `_internal` routes make it into the build?"

**TL;DR:** Unauthenticated debug routes (`test-finnhub`, `test-search`) proxy Finnhub directly and leak API-key-presence info to anyone — remove them or gate behind a non-production check before shipping.

**Key mappings:** `backend/src/routes/stocks.js` (`GET /test-finnhub/:ticker`, `GET /test-search/:query`)

---

### Q23: The codebase logs extensively — including `console.log`ging a preview of the auth token. Why is that risky, and what's the right logging approach for production?

**Polished answer:** Both `middleware/auth.js` and `services/api.js` log partial JWT previews (`authHeader.substring(0, 30)`, `access_token.substring(0, 20)`) presumably for debugging auth issues during development. The risk: production logs are often shipped to third-party log aggregators, stored for extended periods, and accessible to a wider set of people (ops, support, sometimes external vendors) than the application's actual auth boundary — logging *any* prefix of a bearer token, even truncated, is bad practice because (a) tokens are meant to be treated as secrets in transit and at rest, full stop, and (b) depending on the JWT's structure, even a prefix could leak header/algorithm info useful for crafting attacks, though the bigger issue is just the general hygiene principle: **never log credentials, even partially, even for debugging** — use a request ID or a boolean ("token present: true/false") instead, exactly as the `apiKey: ... ? 'Present' : 'Missing'` pattern elsewhere in the codebase already does correctly. Beyond this specific issue, the volume of `console.log` calls throughout the codebase (dozens across routes, services, and components) is itself a production concern: unstructured `console.log` output is hard to filter/query at scale, has non-trivial performance cost in hot paths, and mixes debug noise with real signal. The professional fix: a structured logger (e.g., `pino` or `winston`) with log **levels** (`debug`/`info`/`warn`/`error`) so verbose debug logs can be turned off in production via config, and a habit of never interpolating secrets into any log line regardless of level.

**TL;DR:** Logging any prefix of a JWT/token is a credential-hygiene violation — even truncated; replace with boolean presence checks. More broadly, replace scattered `console.log` with a leveled structured logger so debug noise can be silenced in production.

**Key mappings:** `backend/src/middleware/auth.js`, `frontend/src/services/api.js`

---

### Q24: Some routes return `error.message` directly to the client, others return a generic message. What's the security concern, and what's the right policy?

**Polished answer:** Compare `POST /stocks` (`res.status(400).json({ error: error.message })` — leaks the raw exception message) with `DELETE /stocks/:id` (`res.status(500).json({ error: 'Failed to delete stock' })` — generic, no leak). Returning raw `error.message` to the client is risky because database/library error messages can accidentally reveal internal implementation details — table/column names, ORM/driver internals, sometimes even fragments of a query — which is exactly the kind of information reconnaissance an attacker probing your API wants, and it's inconsistent UX besides (some errors are actionable/human-readable, others might be a raw stack-trace-adjacent string). The right policy: **log the full error server-side** (with a structured logger, including stack trace) for debugging, but **return a curated, generic, user-safe message to the client** by default, only surfacing specific validation messages you've deliberately authored yourself (e.g., "Please fill in all required fields" — that's fine because *you* wrote that string, it's not a raw DB exception). A common production pattern is a centralized Express error-handling middleware that catches all thrown errors, logs them with full detail, and formats a consistent, safe JSON error shape (`{ error: { code, message } }`) for every response, rather than each route handler deciding ad hoc what to reveal.

**TL;DR:** Some routes leak raw `error.message` (potential internal-details disclosure), others return safe generic strings — inconsistent; the fix is centralized error-handling middleware that logs full detail server-side and returns only curated, safe messages to clients.

**Key mappings:** `backend/src/routes/stocks.js` (compare `POST /` vs `DELETE /:id` catch blocks), `backend/src/routes/assets.js`

---

### Q25: `Dashboard.jsx` merges `stocks` and `metals` into one `assets` array via `useEffect`, then accesses fields inconsistently (`shares || grams`, `ticker || name`). Critique this.

**Polished answer:** `useEffect(() => { setAssets([...stocks, ...metals]) }, [stocks, metals])` combines two conceptually different entities — equities (which have `shares`, `ticker`) and metals (which have `grams`, no `ticker`) — into a single array, and every downstream calculation has to defensively branch: `asset.current_price * (asset.shares || asset.grams)`, `asset.ticker || asset.name`. This works, but it's fragile: it relies on the *convention* that exactly one of `shares`/`grams` is ever populated per object, with no type discriminant enforcing that, and it silently produces wrong results if both happen to be present or both absent (e.g., `0 || asset.grams` — if `shares` is legitimately `0`, this falls through to `grams`, which for a stock row would be `undefined`, so the value becomes `NaN` without an obvious signal why). This is a classic case for either a **discriminated union pattern** — normalize both shapes at the API boundary (or in this `useEffect`) into a common interface, e.g. `{ id, displayName, quantity, unit, current_price, buy_price, kind: 'stock' | 'metal' }` — or, in a typed codebase, a TypeScript union type with a `kind` tag that lets you narrow safely instead of `||`-guessing which field is populated. The broader lesson: when you find yourself writing `a || b` to paper over "this could be one of two different shapes," that's usually a sign the two shapes should have been unified into one consistent interface earlier in the data flow, ideally at the API layer rather than in a UI component.

**TL;DR:** Merging stocks (`shares`/`ticker`) and metals (`grams`/no ticker) into one array and reading fields with `a || b` fallbacks is fragile (silently wrong if `shares === 0`, no discriminant field) — normalize to a common shape with an explicit `kind` tag instead.

**Key mappings:** `frontend/src/pages/Dashboard.jsx` (the `assets` merge `useEffect` and all `asset.shares || asset.grams` usages)

---

## TOP 26–50 — Deeper cuts: validation, patterns, testing, behavioral framing

### Q26: The `PUT /stocks/:id` route spreads `req.body` directly into the update (`stock.update({ ...req.body, last_updated: new Date() })`) with no field allow-list. What's the risk, and what's this vulnerability class called?

**Polished answer:** This is a **mass assignment vulnerability**. Because `req.body` is spread wholesale into the update payload with no whitelist of permitted fields, a malicious (or just buggy) client could send `{ shares: 10, user_id: 'someone-elses-uuid', current_price: 999999 }` in the PUT body — `shares` and any other column update is fine, but `user_id` and `current_price` almost certainly shouldn't be client-settable at all: `user_id` should never change after creation (ownership shouldn't be reassignable via a stock-edit form), and `current_price` is supposed to be a server-computed/Finnhub-sourced value, not something the user directly sets when "editing shares." The ownership check earlier in the route (`stock.user_id !== req.user.id`) prevents editing *someone else's* row, but does nothing to stop a user from overwriting fields on their *own* row that shouldn't be user-editable. The fix is an explicit allow-list: `const { name, ticker, shares, buy_price, target_price } = req.body; await stock.update({ name, ticker, shares, buy_price, target_price, last_updated: new Date() })` — only pass through the specific fields the form is actually meant to edit, exactly mirroring what the `POST /` handler already does correctly by destructuring named fields instead of spreading the whole body.

**TL;DR:** Spreading `req.body` directly into an update is a mass-assignment vulnerability — a client could set `user_id` or `current_price` via a PUT meant only to edit `shares`/`buy_price`; fix with an explicit field allow-list, as `POST /` already does correctly.

**Key mappings:** `backend/src/routes/stocks.js` (`router.put('/:id', ...)`)

---

### Q27: Compare this hand-rolled model layer to a "real" ORM (Prisma/TypeORM/Sequelize) — pros, cons, and when you'd recommend switching.

**Polished answer:** Pros of the current approach: zero extra dependency/learning curve beyond Supabase's own client, no migration-tooling mismatch (schema changes happen directly in Supabase's dashboard/SQL editor, no separate migration DSL to keep in sync), and it works fine in serverless since it rides on Supabase's REST layer (PostgREST) rather than needing its own connection pool. Cons: no compile-time type safety (Prisma generates types from your schema; here, a typo in a column name only surfaces at runtime as a Postgres/PostgREST error), no built-in relation loading (Prisma's `include`, Sequelize's `include` — here you'd hand-write multiple queries or use Supabase's embedded-resource select syntax, which is less ergonomic for complex joins), no transactions across multiple writes (if you need "insert a transaction row AND decrement shares atomically," this model layer has no primitive for that — you'd need Postgres RPC functions or accept non-atomic multi-step writes, a real risk area), and no migrations-as-code (schema drift between environments has to be managed manually). I'd recommend switching to a real ORM once: (a) the schema needs multi-table transactional writes with rollback guarantees, (b) the team is large enough that compile-time type safety on every query meaningfully reduces bugs, or (c) query complexity (deep joins, aggregations) outgrows what Supabase's query builder expresses cleanly. For a project this size, the current approach is a reasonable, pragmatic choice — I wouldn't over-engineer it prematurely.

**TL;DR:** Current approach: no extra deps, plays well with serverless, but no type safety, no relation loading, no transactions, no migrations-as-code. Switch to a real ORM once you need atomic multi-table writes, compile-time safety at scale, or complex joins.

**Key mappings:** `backend/src/models/Stock.js`, `backend/src/models/Asset.js`

---

### Q28: What happens if `POST /stocks` succeeds on the server but the client's network drops before receiving the response, and the user retries? How would you make this operation idempotent?

**Polished answer:** As written, a retry after a dropped response would call `POST /stocks` a second time with the same form data, and since there's no uniqueness constraint or idempotency check, the second call creates a **second, duplicate row** — the user now owns the same stock twice, each independently tracked, silently inflating their portfolio total. This is a common distributed-systems gotcha: "at-least-once delivery" (the client can't distinguish "request never reached the server" from "request succeeded but the response was lost") means naive retry logic can double-apply non-idempotent operations. Standard fixes: (1) **client-generated idempotency key** — the frontend generates a UUID per "add stock" attempt, sends it as a header (`Idempotency-Key`), and the backend checks a short-lived store (a dedicated table or Redis) for that key before processing; if seen before, return the original result instead of re-inserting. (2) A **natural uniqueness constraint** in the schema — e.g., a unique index on `(user_id, ticker)` if the product intends "one row per ticker per user" (though that conflicts with allowing multiple separate buy-lots of the same ticker at different prices, which is a real product decision to clarify first). (3) Simpler mitigation for this specific UI: disable the submit button immediately on click and only re-enable on error, reducing (but not eliminating, since network-level retries can still happen) the chance of a double-submit.

**TL;DR:** Without an idempotency mechanism, a lost response + client retry duplicates the stock row; fix with a client-supplied idempotency key checked server-side, or a natural DB uniqueness constraint if the product model allows it — plus disabling the submit button as a cheap first mitigation.

**Key mappings:** `backend/src/routes/stocks.js` (`POST /`), `frontend/src/components/modals/AddStockModal.jsx` / `frontend/src/pages/Portfolio.jsx` (`handleAddStock`)

---

### Q29: There are *two* different implementations related to stock price history — `getHistoricalData()` in the service, and the actual `/:ticker/history` route. Compare them and explain why this is a code-quality red flag.

**Polished answer:** `stockPriceService.getHistoricalData(ticker)` is a real implementation — it calls Finnhub's `/stock/candle` endpoint for genuine 30-day OHLC data and maps timestamps to prices. But the actual route the frontend calls, `GET /:ticker/history`, does **not** use this function at all; instead it fetches a single current quote, then generates 31 fake data points via `currentPrice = Math.max(currentPrice + (Math.random() - 0.5) * 5, 1)` — a random walk seeded from the real current price, not real historical data. This is a meaningful discrepancy to flag: there's a correct, presumably-tested implementation sitting unused right next to the route that actually serves users fake data. This likely happened because Finnhub's candle endpoint has stricter tier/rate limits than the quote endpoint, so a developer substituted a synthetic placeholder during development ("fake it until the real API access is sorted") and it never got swapped back. In an interview, this is a great example of "read the code, don't trust the function names" — `/history` *sounds* like it returns history, but doesn't. The fix, beyond just wiring up the real function, is to make the fakeness impossible to miss in code (a `// TODO: replace with real getHistoricalData() once rate limits allow` comment at minimum, or better, fail loudly / return a clear "data unavailable" state rather than silently fabricating numbers a user might make decisions from).

**TL;DR:** A real `getHistoricalData()` (uses Finnhub's candle endpoint) exists but is unused; the actual `/history` route instead fabricates a random walk from the current price — a "the function name lies" bug, likely a leftover placeholder from a rate-limit workaround.

**Key mappings:** `backend/src/services/stockPriceService.js` (`getHistoricalData`, unused), `backend/src/routes/stocks.js` (`GET /:ticker/history`, the actual implementation used)

---

### Q30: Why is a `for` loop with a hardcoded `setTimeout` delay a fragile way to respect a third-party rate limit? What's a more robust pattern?

**Polished answer:** `await new Promise(r => setTimeout(r, 1000))` between iterations assumes Finnhub's rate limit is exactly "1 request/second, forever" — but real rate limits are usually expressed as a rolling window (e.g., "60 requests/minute," which actually permits short bursts as long as the *average* stays under the cap), and third-party limits can also change over time or differ by API tier, silently making a hardcoded sleep either too conservative (slower than necessary) or too aggressive (still getting 429s) without any code change signaling the mismatch. It's also **not resilient to failure** — if one `getStockQuote()` call throws or hangs near its 7-second timeout, the fixed 1-second gap doesn't account for that, and there's no backoff on actual `429 Too Many Requests` responses (no retry-with-backoff at all, actually — a `429` would just be swallowed by the generic `catch` and treated as a failed quote). A more robust pattern: use an actual **token-bucket or leaky-bucket rate limiter** (libraries like `bottleneck` or `p-limit` combined with a rate window) that adapts to real request timing rather than a fixed sleep, **and** implement exponential backoff with jitter specifically on `429` responses (read `Retry-After` headers if Finnhub provides them), so the system self-corrects if it's rate-limited rather than blindly plowing ahead at a fixed cadence that might not actually match the provider's real constraints.

**TL;DR:** A fixed `setTimeout(1000)` between calls is a rough approximation of a rate limit, not an adaptive one — no backoff on actual 429s, no burst awareness; use a real rate-limiting library (token bucket) plus exponential backoff with jitter on rate-limit responses.

**Key mappings:** `backend/src/services/stockPriceService.js` (`updateStockPrices`)

---

### Q31: Why is `setInterval`-based background scheduling fundamentally broken on Vercel serverless, even if `startPeriodicUpdates()` *were* called somewhere?

**Polished answer:** Serverless functions are, by design, **stateless and ephemeral** — the platform spins up an instance to handle a request (or a batch of requests during a warm period), then may freeze or fully terminate that instance once traffic subsides, with no guarantee of how long any given instance stays "warm." `setInterval` schedules a callback to fire on a timer *within a single running process* — but there's no guarantee any process stays alive long enough for a 5-minute interval to ever fire even once, and even if it did fire on one invocation, a **new** cold-started invocation for the next incoming request would start an entirely separate `setInterval` from scratch (with no shared memory to know "we already have one running"), so under real traffic you could end up with **many concurrent, duplicate interval timers** across different function instances rather than one clean periodic job, or **zero** if the instance is torn down between the interval boundaries. The correct primitive for "run this on a schedule" in a serverless context is an **external scheduler that invokes your function via HTTP**, not a timer living inside the function itself — Vercel Cron Jobs (a `vercel.json` `crons` config hitting a specific API route on a schedule), or an external service (GitHub Actions scheduled workflow, AWS EventBridge, a dedicated cron service) that calls your `/api/update-prices` endpoint every N minutes. This distinction — "in-process timers don't survive serverless's stateless execution model, so scheduling has to be pushed to an external trigger" — is a core serverless concept worth being able to explain crisply.

**TL;DR:** `setInterval` lives inside a process instance; serverless instances are ephemeral and can be duplicated/torn down unpredictably, so an in-process timer either never fires reliably or spawns duplicate timers across concurrent cold starts — scheduling must come from an external trigger (Vercel Cron, EventBridge, etc.) hitting an HTTP endpoint instead.

**Key mappings:** `backend/src/services/stockPriceService.js` (`startPeriodicUpdates`), `backend/src/server.js` (note: never actually invoked)

---

### Q32: Explain the dark-mode implementation — Tailwind's `class` strategy, `localStorage` persistence, and cross-component sync via custom events.

**Polished answer:** Tailwind's dark mode here works via the `class` strategy: `document.documentElement.classList.add('dark')` toggles a `dark` class on `<html>`, and every component's Tailwind classes use `dark:` variants (e.g., `theme === 'dark' ? 'bg-[#171717]' : 'bg-white'` — though notice this project mostly does **manual** conditional classes based on a `theme` string rather than actually using Tailwind's `dark:` prefix variants, meaning it's not fully leaning on Tailwind's own dark-mode mechanism at all, just reimplementing the same idea by hand with ternaries everywhere). `DarkModeContext` persists the boolean to `localStorage` on every change and initializes from it on load (avoiding a flash-of-wrong-theme on refresh, assuming the initializer runs before first paint — though with client-side-only React and no SSR here, some flash is still likely since the HTML `class` isn't set until JS runs). As discussed in Q12, theme *also* gets synced through parallel custom-event and prop-drilling paths in `Dashboard.jsx`, which is redundant given Context already solves this. A cleaner implementation: rely purely on `DarkModeContext` + Tailwind's actual `dark:` variant classes (configured via `darkMode: 'class'` in `tailwind.config.js`) instead of manually branching `theme === 'dark' ? 'x' : 'y'` throughout every component — that removes both the redundant event-sync code and the repetitive ternary styling pattern in one move.

**TL;DR:** Dark mode toggles a `dark` class on `<html>`, persisted to `localStorage` via Context — but most components hand-roll `theme === 'dark' ? a : b` ternaries instead of using Tailwind's own `dark:` variant classes, and theme state is redundantly re-synced via custom events on top of Context.

**Key mappings:** `frontend/src/context/DarkModeContext.jsx`, `frontend/tailwind.config.js`, `frontend/src/pages/Dashboard.jsx` (ternary styling pattern)

---

### Q33: Explain the `DashboardLayout` composition pattern and why wrapping every route in `<ProtectedRoute><DashboardLayout><Page /></DashboardLayout></ProtectedRoute>` in `App.jsx` is repetitive — how would you reduce the duplication?

**Polished answer:** `DashboardLayout` composes `Sidebar` + `Navbar` around whatever `children` it's given — a standard **layout component** pattern in React, letting every authenticated page share consistent chrome without each page re-implementing navigation. The repetition problem is in `App.jsx`: every single protected route repeats the same three-level wrap (`ProtectedRoute` → `DashboardLayout` → the actual page), which is verbose and easy to get subtly wrong (forget the wrapper on a new route, or pass `theme` inconsistently). React Router v6+ (this project uses v7) supports **nested routes with a shared layout route**, which collapses this: define one parent `<Route element={<ProtectedRoute><DashboardLayout theme={theme}><Outlet /></DashboardLayout></ProtectedRoute>}>` and nest all nine dashboard pages as children rendered via `<Outlet />`, eliminating the repeated wrapping entirely and making "is this route protected + laid out consistently" a property of the route tree structure rather than something repeated at every leaf. This is a good React Router–specific optimization to mention if asked "how would you refactor this."

**TL;DR:** Every protected route repeats `ProtectedRoute` + `DashboardLayout` wrapping manually; React Router's nested-route + `<Outlet />` pattern collapses this into one shared parent route, removing the duplication.

**Key mappings:** `frontend/src/App.jsx`, `frontend/src/components/layout/DashboardLayout.jsx`

---

### Q34: `PortfolioAnalytics.jsx` renders a Recharts pie chart with a custom hover "active shape." What performance/accessibility concerns would you raise about this component?

**Polished answer:** **Performance:** `chartData` is recomputed via `.map().sort()` on every render from the `stocks` prop with no memoization (`useMemo`) — fine for a handful of holdings, but if this list grows large (dozens+ of positions) or the parent re-renders frequently (e.g., due to the 1-second `currentTime` timer ticking in `Dashboard.jsx`, which triggers a full component re-render every second — worth flagging on its own as an unnecessary re-render source for anything below it in the tree that doesn't need per-second updates), this recomputation happens needlessly often; wrapping it in `useMemo(() => ..., [stocks])` would avoid recalculating unless the underlying data actually changed. **Accessibility:** the pie chart's information (name, percentage, dollar value, gain/loss) is conveyed entirely through SVG shapes, color, and hover-triggered text — there's no `aria-label` or accessible text alternative for screen-reader users, and color is used to convey gain (green) vs loss (red) without a non-color indicator (like a `+`/`-` sign, which thankfully *is* present in the text, so that part's fine — but the color-only distinction elsewhere, like the pie slice colors themselves carrying no semantic meaning beyond "different category," is a more minor concern). A stronger accessible version would pair the visual chart with a genuinely readable data table (which, notably, `Dashboard.jsx`'s "Your Holdings" table already provides separately — so the *information* is accessible elsewhere in the page, just not through the chart itself, which is a reasonable trade-off to state rather than a hard failure).

**TL;DR:** Chart data is recomputed on every render without memoization (worsened by an unrelated 1-second timer causing frequent parent re-renders); accessibility-wise, the chart's info is redundant with an existing plain-text holdings table elsewhere on the page, which mitigates (but doesn't eliminate) the lack of chart-level ARIA labeling.

**Key mappings:** `frontend/src/components/dashboard/PortfolioAnalytics.jsx`, `frontend/src/pages/Dashboard.jsx` (`currentTime` 1s interval, "Your Holdings" table)

---

### Q35: Explain precisely what `supabase.auth.getUser(token)` does server-side, and why that's different (and necessary) compared to just decoding the JWT's payload without verification.

**Polished answer:** A JWT has three parts — header, payload, signature — and the payload (containing claims like user ID, expiry) is only **base64-encoded**, not encrypted; anyone can decode it and read the claims without any secret, using something like `jwt.decode()` (no verification) or even just pasting it into jwt.io. That means **trusting the payload without verifying the signature is trusting user-controllable input** — a malicious client could hand-craft a JWT-shaped string with whatever `user_id` claim they want, and a naive `jwt.decode()`-only check would happily accept it as if Supabase had issued it. `supabase.auth.getUser(token)` instead makes a real verification: it checks the signature against Supabase's known signing key (proving the token was actually issued by Supabase's auth server, not forged), and (depending on implementation) checks expiry and that the token hasn't been revoked — effectively asking Supabase's auth service "is this token real and currently valid," not just "what does this token's payload claim to say." This is the difference between **decoding** (reading claims, trusting them blindly) and **verifying** (cryptographically confirming those claims are legitimate) — and it's a foundational security concept worth being able to state precisely, since "I decoded the JWT and read the user ID" is a dangerously common mistake in take-home projects and even production code.

**TL;DR:** JWT payloads are base64-encoded, not encrypted — decoding without verifying the signature means trusting attacker-controllable input; `supabase.auth.getUser(token)` does real signature verification against Supabase's auth service, confirming the token is genuinely valid, not just well-formed.

**Key mappings:** `backend/src/middleware/auth.js`

---

### Q36: Why does `stockPriceService.js` set a 7-second `axios` timeout on external calls specifically, and how does that interact with serverless function execution limits?

**Polished answer:** Vercel serverless functions have a maximum execution duration (on the free/hobby tier, historically around 10 seconds; higher on paid tiers) — if a function runs longer than its allotted limit, the platform kills it and the client gets a timeout/error regardless of whether your own code was about to finish. A 7-second `axios` timeout on the Finnhub call is a deliberate **timeout budget**: it ensures that if Finnhub is slow or hanging, your own function fails fast (throwing/returning `null`, per the `catch` block) with enough headroom left in the *overall* function execution budget to still send a clean error response, rather than the entire function being hard-killed mid-request by the platform with no chance to respond gracefully at all. This is a good example of **defensive timeout budgeting in a constrained execution environment**: the external call's timeout should always be meaningfully shorter than your own function's total execution limit, with margin for the rest of your logic (JSON parsing, database writes) to still run afterward. Worth mentioning as a generalizable principle: whenever you're calling an external dependency from inside a hard execution-time budget (serverless functions, but also things like AWS Lambda, or even client-side code with a UX-driven "don't hang forever" requirement), you should always set an explicit timeout on that call, tuned to leave enough slack for the rest of your own logic.

**TL;DR:** A 7s timeout on external API calls "fails fast" so there's still time left in the serverless function's own execution budget to handle the failure gracefully, instead of letting the platform hard-kill the whole function if the third-party API hangs.

**Key mappings:** `backend/src/services/stockPriceService.js`, `backend/src/services/assetPriceService.js`

---

### Q37: The walkthrough claims `react-router-dom` was moved from `devDependencies` to `dependencies` to fix a prod build. Check the rest of `frontend/package.json` — is that fix actually complete?

**Polished answer:** This is a strong "trick question" to be ready for, because the honest answer is **no, the fix is incomplete**. `react-router-dom` was indeed moved to `dependencies`, but `frontend/package.json` still lists `@tremor/react` (imported directly in `Dashboard.jsx` for the `AreaChart`/`Card`/`Badge` components), `@heroicons/react` (imported in `AddStockModal.jsx`), `react-icons` (imported across `Dashboard.jsx`, `Portfolio.jsx`, and others for `Hi*` icons), and `framer-motion` (imported in `Portfolio.jsx` as `motion`) all still under `devDependencies` — every one of these is used in runtime UI code, not just build tooling or local dev. The distinction that matters: `devDependencies` are stripped out when a production install runs with `npm install --production` (or equivalents), which is exactly the class of build environment where "works locally, breaks in prod" bugs like the original `react-router-dom` issue occur — so unless the specific Vite/Vercel build pipeline here happens to install all deps regardless of the `dev`/`prod` split (some CI setups do), these libraries are latent copies of the exact same bug class the walkthrough claims to have fixed. The correct fix, generalized: audit every import across the codebase and confirm each package is declared in the dependency section matching how it's actually used — "is this imported by code that ships to the browser/server at runtime" (→ `dependencies`) vs "is this only used by build tools, linters, or local dev scripts" (→ `devDependencies`).

**TL;DR:** Only `react-router-dom` got moved to `dependencies`; `@tremor/react`, `@heroicons/react`, `react-icons`, and `framer-motion` are all still in `devDependencies` despite being imported in runtime UI components — the same bug class the walkthrough claims to have fixed is still present elsewhere.

**Key mappings:** `frontend/package.json` (`dependencies` vs `devDependencies`), `frontend/src/pages/Dashboard.jsx` (`@tremor/react`, `react-icons`), `frontend/src/components/modals/AddStockModal.jsx` (`@heroicons/react`), `frontend/src/pages/Portfolio.jsx` (`framer-motion`, `react-icons`)

---

### Q38: Why does the project commit `package-lock.json`, and what breaks if it's missing or out of sync?

**Polished answer:** `package.json` specifies version *ranges* (e.g., `^18.3.1`), not exact versions — `package-lock.json` records the **exact resolved version** of every direct and transitive dependency actually installed at some point in time, so that `npm ci` (which the lockfile enables, and which most CI/CD pipelines including Vercel's builds use or should use) produces a byte-identical `node_modules` tree on every machine and every build, rather than potentially resolving to a newer patch/minor version that happens to have a breaking change. Without a committed lockfile (or with one that's out of sync with `package.json`, e.g., a dependency was added to `package.json` by hand without running `npm install` to regenerate the lock), builds become **non-reproducible**: it might work on your machine today and break in CI tomorrow purely because a transitive dependency published a new version in between, with zero code changes on your end — exactly the kind of "works on my machine" bug that's maddening to debug because the diff between working and broken isn't in your commit history at all. The walkthrough's troubleshooting table explicitly calls this out ("Build fails — missing module → Run `npm install` locally and commit `package-lock.json`"), which is the right instinct: any time a build fails due to a missing/mismatched module, regenerating and committing the lockfile is the fix, and it should generally be treated as required, not optional, in any real deployment pipeline.

**TL;DR:** `package-lock.json` pins exact dependency versions for reproducible installs across machines/CI; without it (or if it drifts out of sync with `package.json`), builds can silently resolve different dependency versions over time and break with no corresponding code change.

**Key mappings:** `frontend/package-lock.json`, `backend/package-lock.json`, `walkthrough.md` (troubleshooting table)

---

### Q39: React Router uses client-side (`BrowserRouter`) routing. What has to be configured on Vercel's static hosting for a direct URL like `/portfolio` to work on a hard refresh, and does this project have it?

**Polished answer:** With `BrowserRouter`, routes like `/portfolio` are **fake paths as far as the actual file system/server is concerned** — React Router intercepts navigation client-side and swaps components without a real page load, but if a user hard-refreshes on `/portfolio`, or shares that URL directly, the *browser* makes a real HTTP `GET /portfolio` request to Vercel's static host, which has no literal `portfolio.html` file — without special configuration, that returns a 404 instead of loading the SPA. The fix required is a **rewrite rule**: any unmatched path should be rewritten to serve `index.html` (letting React Router then take over client-side and render the right page based on the URL). This project's `frontend/vercel.json` is explicitly called out in `implementation_plan.md` as "already correct — no changes needed," implying it already contains this rewrite. I'd confirm this is genuinely a SPA fallback rewrite (typically `{ "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }` in Vercel's config format) rather than assume — it's exactly the kind of "looks fine until someone bookmarks a deep link or refreshes" bug that doesn't show up in normal click-through testing during development (since dev-server navigation never triggers a real server request for those paths), which makes it a great thing to specifically test for before considering a SPA deploy "done."

**TL;DR:** `BrowserRouter` paths like `/portfolio` don't correspond to real files — a hard refresh sends a real HTTP request the static host must rewrite to `index.html`, or it 404s; this needs to be explicitly configured in `vercel.json`'s rewrites (the project claims this is already handled — worth verifying, not assuming).

**Key mappings:** `frontend/vercel.json`, `frontend/src/App.jsx` (`BrowserRouter`)

---

### Q40: There's no request-body validation library (Zod, Joi, express-validator) anywhere in the backend — just manual `parseFloat`/`isNaN` checks. What's the risk, and how would you add a validation layer?

**Polished answer:** Validation here is scattered and inconsistent: `assets.js`'s `POST /` manually checks `if (!name || !grams || !buy_price)` then separately validates `isNaN(parsedGrams) || parsedGrams <= 0`, while `stocks.js`'s `POST /` does *no* explicit validation at all beyond implicitly relying on `parseFloat` producing `NaN` for garbage input (which would then silently propagate into a database write as `NaN`, since nothing checks for it before calling `Stock.create()`). This inconsistency — some routes validate thoroughly, some barely at all — means the actual data-integrity guarantees of the API depend on which route you happen to be calling, which is fragile and hard to reason about as the API grows. A validation library like **Zod** (increasingly the default choice in the Node ecosystem for its TypeScript-first, composable schema API) would let you define one schema per route's expected body shape once (e.g., `z.object({ name: z.string().min(1), ticker: z.string().min(1), shares: z.number().positive(), buy_price: z.number().positive() })`), run it as Express middleware before the handler even executes, and get consistent, well-structured 400 error responses automatically for any malformed request — centralizing the "is this input even valid" concern instead of hand-rolling ad hoc `if` checks (or, worse, no checks) differently in every route file. This is a strong "what would you add to harden this API" answer.

**TL;DR:** Validation is inconsistent and manual (`isNaN` checks in some routes, none in others) — a schema-validation library like Zod, run as middleware before route handlers, would centralize and standardize input validation across every route instead of ad hoc per-route checks.

**Key mappings:** `backend/src/routes/assets.js` (`POST /` manual checks), `backend/src/routes/stocks.js` (`POST /`, no explicit validation)

---

### Q41: There are no test files anywhere in this repository. If asked "how would you test this," what would you prioritize, and why?

**Polished answer:** I'd frame this by risk and cost, not "test everything equally." **Highest priority — backend business logic with real money/data implications:** unit tests for `stockPriceService.getStockQuote()`'s fallback logic (does it correctly fall back to `buy_price` when Finnhub fails or returns `0`?), and for the portfolio math (`totalValue`/`totalGain`/`percentageGain` calculations, including the zero-division edge cases from Q16) — these are pure functions or near-pure, cheap to unit test, and bugs here directly show wrong numbers to users. **Second priority — API contract tests (integration tests) for the Express routes:** using something like `supertest` against a test Supabase project (or a mocked Supabase client), verifying auth is actually enforced (a request with no/invalid token gets 401), ownership checks work (`user_id` mismatch on PUT/DELETE actually returns 403, not just for the "happy path" test), and that the mass-assignment issue from Q26 doesn't let a client set `current_price` via PUT — this is exactly the class of bug that's invisible in manual testing but easy to write a regression test for once found. **Third — critical user flows end-to-end** (Playwright/Cypress): sign up → add a stock → see it in the portfolio list → edit it → delete it, as one flow, to catch integration breakage across the full stack. I would explicitly *not* prioritize exhaustive UI snapshot tests for every component early on — for a project this size, they're high-maintenance relative to the bugs they actually catch, whereas the auth/ownership/math test categories above catch the bugs that would actually hurt real users or leak data.

**TL;DR:** Prioritize (1) unit tests on portfolio math and price-fallback logic, (2) integration tests asserting auth/ownership are actually enforced on every route (the highest-value tests, since these are invisible bugs in manual testing), then (3) a handful of true end-to-end flows — deprioritize exhaustive UI snapshot testing for a project this size.

**Key mappings:** N/A (no test files present — `backend/`, `frontend/src/` both lack any `*.test.js`/`*.spec.js`)

---

### Q42: If you were designing this schema from scratch today, what would the normalized table structure look like?

**Polished answer:** I'd propose separating concerns that are currently conflated in a single `stocks` table: **`securities`** — a shared lookup table (`ticker` PK/unique, `name`, `exchange`) so ticker/name metadata isn't duplicated per user-row and can be looked up once; **`holdings`** — `user_id`, `security_id` (or `ticker`), `shares`, `buy_price`, `target_price`, `current_price` (or, better, current price lives in a separate `prices` cache table keyed by ticker, shared across all users holding that ticker, updated by the background job from Q9/Q31 — avoiding storing a redundant, independently-staling copy of the same market price once per user per holding); **`watchlist`** — `user_id`, `ticker`, `added_at` (no cost-basis fields at all, since watchlist items were never bought); **`transactions`** — an actual audit-log table (`user_id`, `ticker`, `type: 'buy'|'sell'`, `shares`, `price`, `timestamp`) recording every buy/sell event individually, which the current schema has *no equivalent of at all* (see Q43) despite a `TransactionHistory.jsx` component existing in the frontend; **`assets`** (metals) similarly split into a metals-specific table since they don't share `holdings`' `ticker`/exchange semantics. This design trades a bit more query complexity (joins instead of one flat table) for correctness (no more `shares || grams` guessing), no data duplication (shared price cache instead of N copies of the same market price), and an actual historical record (`transactions`) instead of only ever representing the *current* state.

**TL;DR:** Split the overloaded `stocks` table into `securities` (shared metadata), `holdings`, `watchlist`, a shared `prices` cache, and — critically missing today — an actual `transactions` audit-log table, rather than one flat table trying to represent current holdings, watchlist, and price all at once.

**Key mappings:** current: `backend/src/models/Stock.js`, `backend/src/models/Asset.js`; frontend hint at missing feature: `frontend/src/components/dashboard/TransactionHistory.jsx`

---

### Q43: `TransactionHistory.jsx` exists in the frontend, implying a buy/sell history feature. Does the backend actually support it? What does this gap tell you about verifying a feature "end to end"?

**Polished answer:** Searching the backend routes and models turns up **no `transactions` table, model, or route anywhere** — `stocks.js` only ever reads/writes the current state of a `stocks` row (`shares`, `buy_price`, `current_price`), with no event log of individual buy/sell actions over time. So `TransactionHistory.jsx` either renders from mock/hardcoded data, derives a pseudo-history from the single current snapshot (which can't reconstruct real historical events, only the current state), or is simply an unfinished/aspirational component not yet wired to a real backend endpoint. This is exactly the kind of gap that's invisible if you only skim file names and component lists ("oh, they have transaction history, nice") but obvious the moment you trace data flow end to end and ask "where does this component's data actually come from, and does the backend model support recording that history at all?" It's a great habit to describe explicitly in an interview: when asked to assess a codebase (yours or someone else's) or estimate work for a feature, **trace the full path from UI component to database schema** before concluding a feature "exists" — a component name or a page route existing is necessary but not sufficient evidence that the underlying capability is actually implemented.

**TL;DR:** `TransactionHistory.jsx` exists in the frontend, but there's no `transactions` table/route/model anywhere in the backend — the schema only stores current state, not a history of buy/sell events, so this feature is not actually backed end-to-end; a reminder to trace UI → API → schema before assuming a feature is real.

**Key mappings:** `frontend/src/components/dashboard/TransactionHistory.jsx`, absence in `backend/src/models/`, `backend/src/routes/`

---

### Q44: Beyond the metals troy-ounce conversion (Q17), where else in this codebase would floating-point/precision issues realistically bite, and how would a fix propagate through the stack?

**Polished answer:** Every computed financial value in the UI — `totalValue`, `totalGain`, `percentageGain`, per-stock `value`/`gain` in `Dashboard.jsx`'s table, the pie chart's percentage breakdown in `PortfolioAnalytics.jsx` — is derived via chained floating-point arithmetic on values that ultimately came from a Postgres column (likely `float`/`double precision` rather than `numeric`, though I'd confirm the actual column type before asserting), each display point independently calling `.toFixed(2)`, which rounds for *display* but doesn't fix underlying precision drift in further calculations built on unrounded intermediate values. A full fix would need to propagate through three layers: **(1) database** — change `buy_price`/`current_price`/`target_price`/`shares` columns to `numeric`/`decimal` types instead of `float`/`double precision`, so Postgres itself stores and computes exact decimal values; **(2) backend** — since JS numbers are still IEEE-754 doubles regardless of what Postgres sends, any arithmetic done in Node (like the portfolio summary in `GET /stocks/summary`) would ideally use a decimal library (`decimal.js`, `big.js`) rather than native `+`/`*`/`-` on floats, at least for money-relevant aggregation; **(3) frontend** — same principle for any client-side math (the `totalValue`/`totalGain` reduces in `Dashboard.jsx`). In practice, for a personal portfolio tracker (not a real brokerage clearing actual trades), this is a "nice to fully fix, but low real-world risk" issue — I'd flag it accurately as a known limitation rather than either dismissing it or over-stating it as a critical bug, which is itself a good calibration signal to show in an interview.

**TL;DR:** Every financial calculation across DB, backend, and frontend uses native floating-point rather than a decimal type — a full fix means `numeric` columns in Postgres plus a decimal library (`decimal.js`) anywhere money math happens in JS; low real-world risk for a personal tracker, but worth naming accurately rather than over- or under-stating.

**Key mappings:** `backend/src/routes/stocks.js` (`GET /summary`), `frontend/src/pages/Dashboard.jsx` (`totalValue`/`totalGain` reduces)

---

### Q45: `Portfolio.jsx` calls `fetchStocks()` (a full refetch of the entire list) after every add/edit/delete, instead of updating local state directly. Discuss optimistic vs. pessimistic UI updates and the trade-off here.

**Polished answer:** The current pattern is **pessimistic**: after a mutation (`POST`/`PUT`/`DELETE`), the UI waits for the server to confirm success, then re-fetches the *entire* list from scratch to reflect the new state — simple to reason about (the UI is always showing literally what the server says, no risk of client/server state drift) but with two costs: an extra network round-trip beyond the mutation itself (mutate, then separately re-GET everything), and a perceptible delay before the UI updates (button click → wait for POST → wait for GET → then finally see the new row appear), which feels sluggish compared to instant feedback. An **optimistic** alternative would update local state immediately upon the user's action (e.g., append the new stock to local state right when the form submits, using a temporary client-generated ID), then reconcile with the server's actual response once it arrives (replace the temp row with the real one, including the server-assigned ID and Finnhub-sourced `current_price`) — and **roll back** the optimistic change if the request ultimately fails (show an error, remove the optimistically-added row). This feels dramatically snappier but is more complex to implement correctly (rollback logic, temporary-ID reconciliation, handling out-of-order responses if multiple mutations fire close together). For a personal-finance app where correctness matters more than raw snappiness, and where mutation frequency is low (a user isn't adding stocks dozens of times a minute), I'd argue the current pessimistic-refetch approach is a **reasonable, low-risk trade-off** — I'd only push for optimistic updates if user feedback specifically flagged the current UX as sluggish, rather than optimizing this preemptively.

**TL;DR:** Current approach is pessimistic (mutate, then refetch everything) — simple and safe, at the cost of an extra round-trip and slower perceived responsiveness; optimistic updates would feel snappier but add real complexity (temp IDs, rollback) — for this app's low mutation frequency, the current trade-off is defensible, not obviously wrong.

**Key mappings:** `frontend/src/pages/Portfolio.jsx` (`handleAddStock`, `handleEditStock`, `handleDeleteStock`, all followed by `fetchStocks()`)

---

### Q46: The ticker search endpoint (`GET /stocks/search`) is almost certainly wired to an `onChange` handler with no debounce. What's the concrete fix, and how would you explain debouncing to an interviewer who asks you to whiteboard it?

**Polished answer:** Without a debounce, every keystroke in a search input fires a fresh network request — typing "AAPL" naively could fire 4 separate `GET /stocks/search?q=...` calls (`A`, `AA`, `AAP`, `AAPL`), each one hitting Finnhub, racing each other over the network, and potentially resolving **out of order** (a slower response for `A` arriving after a faster response for `AAPL` would overwrite the correct final results with stale ones) — both a wasted-cost problem (burning Finnhub rate-limit budget on intermediate keystrokes nobody needed results for) and a correctness problem (race condition on which response "wins"). The standard fix is **debouncing**: delay firing the request until the user has paused typing for some interval (e.g., 300ms) — implemented with a `useEffect` that sets a `setTimeout` on every `searchQuery` change, clearing the previous timeout if the query changes again before it fires:
```js
useEffect(() => {
  const handle = setTimeout(() => {
    if (searchQuery.length >= 2) searchStocks(searchQuery);
  }, 300);
  return () => clearTimeout(handle);
}, [searchQuery]);
```
The cleanup function returned from `useEffect` is what makes this work — React calls it before running the effect again on the next render, which cancels the pending timeout from the previous keystroke, so only the *last* pause in typing actually triggers a request. I'd also mention **request cancellation** (via `AbortController`, passed to axios's `signal` option) as a complementary fix for the out-of-order race specifically — even with debouncing, if the user pastes text and edits it quickly, an in-flight request can still be superseded; aborting stale in-flight requests when a new one starts is the more complete fix for the race condition itself, debouncing alone mainly reduces request *volume*.

**TL;DR:** No debounce means every keystroke fires a request, wasting rate-limit budget and risking out-of-order response races; fix with a `useEffect`-based debounce (`setTimeout` + cleanup-based cancellation) and, for full correctness against races, an `AbortController` to cancel stale in-flight requests.

**Key mappings:** `backend/src/routes/stocks.js` (`GET /search`), `frontend/src/pages/Portfolio.jsx` (`searchStocks`, `searchQuery` state)

---

### Q47: Is "authorization" (ownership checking) applied consistently across routes? Compare the approach in `GET /stocks` vs. `PUT /stocks/:id`.

**Polished answer:** There are two different (both valid, but worth distinguishing) authorization patterns used here. `GET /` **filters at the query level**: `.eq('user_id', req.user.id)` is baked directly into the Supabase query, so the database itself only ever returns rows belonging to the requesting user — there's no possibility of leaking another user's row because it's never fetched in the first place. `PUT /:id` and `DELETE /:id`, by contrast, **fetch first, then check**: `Stock.findByPk(req.params.id)` retrieves the row *regardless* of who owns it, and only afterward does `if (!stock || stock.user_id !== req.user.id) return res.status(403)` reject the request. Both patterns are legitimate and commonly used — the "filter in the query" approach is generally preferable when fetching *lists* (more efficient, and structurally impossible to leak by omission), while "fetch then check ownership" is often unavoidable for single-resource-by-ID operations where you need the row's data to even know who owns it (you can't filter a query you haven't run yet, since the `id` alone doesn't tell you the owner ahead of time) — though note you *could* combine both by adding `.eq('user_id', req.user.id)` directly to the `findByPk` query itself, turning "not found" and "not yours" into the same outcome (a `404`/`null`), which is arguably a **more secure default** than returning a `403` that confirms to an attacker "this ID exists, you just don't own it" (a minor information-leak difference between "doesn't exist" vs "exists but isn't yours" — some security-conscious APIs deliberately return `404` for both cases specifically to avoid that leak).

**TL;DR:** `GET /` filters ownership at the query level (structurally safe); `PUT`/`DELETE /:id` fetch first then check ownership in application code (works, but returns a distinguishable 403 vs 404, which is a minor info-leak — combining the ownership filter directly into the fetch query would collapse "not found" and "not yours" into one indistinguishable response).

**Key mappings:** `backend/src/routes/stocks.js` (`GET /` vs `PUT /:id` / `DELETE /:id`)

---

### Q48: Trace exactly what happens in the UI when a request returns 401 mid-session (e.g., token just expired) — is the experience graceful?

**Polished answer:** The axios response interceptor in `services/api.js` catches any `401`, logs it, and immediately does `window.location.href = '/'`. Concretely: if a user is deep in the `/portfolio` page, mid-form-fill, and their token happens to expire right as a background request fires, they'd be yanked to the homepage **with no explanation shown in the UI itself** (the explanation only exists as a `console.log`, invisible to a real user) and **with whatever they were doing lost** (unsaved form state, scroll position, current page context) — since, as discussed in Q14, this is a hard browser navigation, not an in-app message-then-redirect. A more graceful version: show a toast/banner ("Your session expired — please sign in again") *before* redirecting, use `navigate('/')` instead of a hard reload to at least avoid a full bundle re-download, and ideally capture the current path so `AuthContext` can support a "redirect back here after re-login" flow instead of always dumping the user at the homepage regardless of where they were. Also worth noting: Supabase's client already handles **silent token refresh** in the background (`autoRefreshToken: true`), so a "real" 401 here (as opposed to a legitimately revoked/invalid session) should actually be relatively rare in normal usage — this matters for prioritization: it's a real UX gap, but likely a low-frequency one, which is a reasonable thing to say explicitly rather than treating it as more urgent than it probably is in practice.

**TL;DR:** A 401 mid-session hard-redirects to `/` with zero user-facing explanation and full loss of in-progress UI state; a more graceful version shows a toast first and uses in-app navigation — though `autoRefreshToken: true` means this path is likely rare in normal usage, worth flagging accurately rather than over-prioritizing.

**Key mappings:** `frontend/src/services/api.js` (401 handler), `frontend/src/config/supabase.js` (`autoRefreshToken: true`)

---

### Q49: There are *three* separate places that call `createClient()` for Supabase on the backend (`config/supabase.js`, `config/database.js`, and again inline in `middleware/auth.js`). Why is this a problem, and how would you fix it?

**Polished answer:** `config/supabase.js` and `config/database.js` are near-identical files — both read the same two env vars and construct a Supabase client with the same fallback demo values, differing only in that `database.js` additionally runs a connection-test query (`.from('stocks').select('count', ...)`) on load; meanwhile `middleware/auth.js` constructs **yet another separate client** inline, purely for `auth.getUser()` calls. Functionally this mostly still works because Supabase clients are stateless-ish wrappers around HTTP calls to the same project (there's no connection-pool exhaustion risk the way there would be with a raw Postgres driver), but it's still a real maintainability problem: if the auth logic (e.g., custom headers, a different Supabase project for auth vs data, retry logic) ever needs to change, there are three places to update and keep in sync, and it's not obvious from a quick read which of `config/supabase.js` vs `config/database.js` is the "real" one other routes should import (in fact, checking usage, `database.js`'s export appears to be **unused entirely** — every route imports from `config/supabase.js` instead, making `database.js` dead code that happens to still run its own connection-test side effect on import for no consumer). The fix: **one canonical client factory** (`config/supabase.js`), imported everywhere — including `middleware/auth.js`, which should import the shared client instead of constructing its own — and delete `config/database.js` entirely once confirmed unused, after moving its useful connection-test-on-boot logic (if desired) into the single canonical file.

**TL;DR:** Three separate `createClient()` calls (`config/supabase.js`, `config/database.js` — apparently dead/unused, and inline in `middleware/auth.js`) duplicate the same setup; consolidate to one canonical client module imported everywhere, and delete the unused duplicate.

**Key mappings:** `backend/src/config/supabase.js`, `backend/src/config/database.js` (check for actual importers — appears unused), `backend/src/middleware/auth.js` (inline `createClient`)

---

### Q50: An interviewer says: "Tell me about a challenging bug you found and fixed." How would you narrate the deployment fixes in `walkthrough.md` as a strong behavioral answer?

**Polished answer:** Structure it as a STAR (Situation/Task/Action/Result) story, and be specific about the *reasoning*, not just the list of fixes — interviewers are testing debugging methodology, not trivia recall. **Situation:** the app worked locally but failed when deployed to Vercel as a serverless split (frontend + backend as separate projects). **Task:** diagnose and fix a chain of environment-specific failures that don't reproduce locally. **Action** — walk through the *reasoning chain*, not just the fix list: "First I found a naming mismatch — a route called `stockPriceService.getQuote()` but the exported function was actually `getStockQuote()` — that's a bug that only a real code-path exercise (not just `npm run build` succeeding) would catch, since JS doesn't statically check that. Then, once that was fixed, the server would still crash specifically in production — I traced it to `app.listen()` being called unconditionally, which is fine for a long-running process but fatal-adjacent for a serverless function that's invoked per-request rather than run persistently, so I gated it behind a `VERCEL` environment check. Separately, the frontend build was failing in the Vercel pipeline but not locally — I found `react-router-dom` was misclassified as a `devDependency`, which `npm install --production`-style installs (used in CI) strip out, so a package the app genuinely needs at runtime was silently missing in the production bundle." **Result:** a working two-project deployment, plus (if you want to go further and show initiative beyond the original walkthrough) "I also noticed while reading the rest of the codebase that the *same* dev/prod dependency misclassification pattern still affects a few other packages that weren't caught in the original fix" (Q37) — that closing line is a strong way to demonstrate you don't just fix the reported bug, you generalize the root cause and look for recurrences of it elsewhere, which is exactly the instinct senior engineers are being screened for.

**TL;DR:** Frame the walkthrough's bug list as a reasoning chain (each fix enabled discovering the next failure), not a flat list — and close by mentioning you found the *same* dependency-misclassification bug pattern recurring elsewhere in the repo (Q37), showing you generalize root causes rather than just patching symptoms.

**Key mappings:** `walkthrough.md`, `implementation_plan.md`

---

## Quick-reference: bugs/smells found while reading (for rapid recall before an interview)

| # | Where | Issue |
|---|---|---|
| 1 | `routes/stocks.js` PUT `/:id` | Discards `update()`'s return value, responds with stale pre-update data |
| 2 | `routes/stocks.js` PUT `/:id` | Mass assignment — spreads `req.body` with no field allow-list |
| 3 | `services/stockPriceService.js` | `setInterval`-based scheduling is incompatible with serverless; `startPeriodicUpdates()` is never even called |
| 4 | `routes/stocks.js` `/:ticker/history` | Returns a fabricated random walk, not real historical data (a real implementation exists unused in the service) |
| 5 | `frontend/package.json` | `@tremor/react`, `@heroicons/react`, `react-icons`, `framer-motion` still in `devDependencies` despite runtime use (same bug class as the "fixed" `react-router-dom` issue) |
| 6 | `store/stockSlice.js` | Empty file — the configured Redux store is entirely dead code |
| 7 | `pages/Dashboard.jsx` | `NaN` risk in `percentageGain` and avg-cost-basis calc on empty portfolios |
| 8 | `routes/assets.js` | Troy-ounce conversion constant (28.34) looks incorrect (~31.1035 is the real value) |
| 9 | `routes/stocks.js` `test-finnhub`/`test-search` | Unauthenticated debug routes leak API-key-presence info |
| 10 | `middleware/auth.js`, `services/api.js` | Logs partial JWT/token previews — credential-hygiene violation |
| 11 | `components/modals/AddStockModal.jsx` | Creates a second, interceptor-less axios instance (no auth header attached) |
| 12 | `config/supabase.js` + `config/database.js` + `middleware/auth.js` | Three separate, duplicated `createClient()` calls; `database.js` appears entirely unused |
| 13 | `components/dashboard/TransactionHistory.jsx` | No backing `transactions` table/route exists anywhere in the backend |
| 14 | Various routes | Inconsistent error handling — some leak raw `error.message`, others don't |
| 15 | `pages/Portfolio.jsx` search | No debounce on the ticker-search-as-you-type API calls |

Good luck — you clearly already have the raw problem-solving chops from competitive programming; these questions are aimed at the *other* half interviewers screen for: reading real code critically, reasoning about trade-offs, and articulating "why," not just "what."
