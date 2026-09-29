# Invest-Tracker — SDE Interview Preparation: Top 50 Questions

This question bank is derived from the complete `Invest-Tracker-main` repository: frontend, backend, configuration, deployment notes, authentication flow, Supabase access, external market/news services, portfolio calculations, and the gaps between what the UI expects and what the backend currently exposes.

## Project snapshot

- **Frontend:** React 18 + Vite + React Router + Tailwind CSS + Tremor + Recharts/Framer Motion.
- **Backend:** Node.js + Express + Axios.
- **Auth:** Supabase Auth on the client; JWT is sent as `Authorization: Bearer ...`; backend verifies the token with Supabase.
- **Database:** Supabase Postgres accessed through `@supabase/supabase-js`.
- **External services:** Finnhub for stock data, Marketaux for news, MetalPriceAPI for gold/silver.
- **Deployment model:** Frontend and backend intended as separate Vercel projects; backend exposes an Express app through Vercel.
- **Main user flows:** sign up/sign in, portfolio CRUD, stock search, price enrichment, precious-metal assets, dashboard aggregation, news, calendars, charts, and theme state.

## Important interview rule for this repository

Some parts of the code represent the **current implementation**, while other files such as `implementation_plan.md` and `walkthrough.md` describe the **intended/fixed architecture**. In an interview, distinguish the two. Do not claim a feature is implemented merely because the UI or plan references it.

---

# Top 10

## 1. Qno: Explain the Invest-Tracker project end-to-end. What happens from login to viewing a user's portfolio on the dashboard?

**Polished answer:**

Invest-Tracker is a full-stack portfolio tracking application. The React/Vite frontend handles routing, UI state, authentication UX, and presentation. Supabase Auth establishes the user session on the client. Once signed in, protected React routes render pages such as Dashboard and Portfolio.

For authenticated API requests, the frontend Axios instance reads the current Supabase session and adds its JWT to the `Authorization: Bearer <token>` header. The Express backend receives the request, and `authenticateUser` calls Supabase `auth.getUser(token)` to verify the JWT. The authenticated user is attached to `req.user`.

Portfolio endpoints then query Supabase Postgres, usually filtering by `user_id = req.user.id`. Stock creation additionally calls Finnhub to obtain the latest quote before inserting the position. The dashboard fetches both `/stocks` and `/assets`, combines the results in React, computes values/gains, and passes the normalized data into analytics components.

So the core path is: **React Router -> Supabase session -> Axios interceptor -> Express middleware -> Supabase/Postgres and external APIs -> JSON response -> React state -> derived portfolio metrics -> UI**.

**TL;DR:** React UI -> Supabase Auth -> JWT -> Express auth middleware -> Supabase/market APIs -> JSON -> React state -> dashboard.

**Key mappings:** `frontend/src/App.jsx`, `frontend/src/context/AuthContext.jsx`, `frontend/src/services/api.js`, `backend/src/middleware/auth.js`, `backend/src/routes/stocks.js`, `backend/src/config/supabase.js`, `frontend/src/pages/Dashboard.jsx`.

---

## 2. Qno: Why did you split the application into frontend, backend, database, and external API layers? Explain the architecture and responsibilities.

**Polished answer:**

The separation is mainly about responsibility and security. The frontend is responsible for presentation, navigation, browser-side session handling, and user interactions. The backend provides the server-side API boundary, validates authentication, performs database operations, and keeps privileged external-service credentials away from the browser. Supabase/Postgres is the persistence layer. Finnhub, Marketaux, and MetalPriceAPI are external data providers.

This separation also makes deployment and scaling more flexible. The frontend can be deployed as a Vite application, while the backend can run independently as a Node/Express service or serverless API. It also prevents the browser from needing direct access to provider API keys such as `FINNHUB_API_KEY` and `METAL_API_KEY`.

In a stronger production design, I would make the API contract explicit, keep provider-specific logic in services, keep database access in a repository/model layer, and keep the route handlers thin. The current project partially follows that pattern through `models` and `services`, but some route files still access Supabase directly.

**TL;DR:** Each layer has one main responsibility: UI, API/security, persistence, and external data.

**Key mappings:** `frontend/`, `backend/src/routes/`, `backend/src/models/`, `backend/src/services/`, `backend/src/config/`.

---

## 3. Qno: How does authentication and authorization work in this project? Why is the frontend session alone not enough?

**Polished answer:**

Authentication is handled by Supabase Auth. The frontend calls `signUp` or `signInWithPassword`, and `AuthContext` keeps the current user in React state. Supabase persists and refreshes the session in the browser.

However, the backend cannot trust a frontend boolean such as `user != null`. For every protected request, the Axios interceptor retrieves the current access token and sends it as a Bearer token. The Express `authenticateUser` middleware verifies that token with Supabase. If verification fails, the API returns HTTP 401.

Authorization is a separate step: after authentication, the route checks whether the requested database record belongs to the authenticated user. For example, the stock update and delete routes load the stock and compare `stock.user_id` to `req.user.id` before allowing the operation.

The most important security principle is: **the client tells us who it thinks the user is; the server independently verifies the identity and enforces access**. In production I would also use Postgres Row Level Security as defense in depth.

**TL;DR:** AuthN verifies identity; AuthZ verifies ownership. Both must happen on the server for protected data.

**Key mappings:** `frontend/src/context/AuthContext.jsx`, `frontend/src/services/api.js`, `backend/src/middleware/auth.js`, `backend/src/routes/stocks.js`.

---

## 4. Qno: Walk me through the Axios authentication interceptor. Why is it useful?

**Polished answer:**

The project creates a shared Axios instance in `frontend/src/services/api.js`. Before every request, a request interceptor calls `supabase.auth.getSession()`. If a session exists, it reads `session.access_token` and injects `Authorization: Bearer <token>` into the request headers.

That centralizes authentication so individual pages such as Portfolio and Dashboard do not have to manually retrieve and attach tokens. It also reduces duplication and keeps the request contract consistent.

There is a response interceptor as well. Successful responses are returned unchanged. For HTTP 401, the project redirects the browser to `/`, which is effectively the login page.

In production, I would improve this by avoiding token previews in logs, distinguishing expired-session recovery from all other 401 cases, and making the redirect behavior router-aware. I would also consider whether every request needs `getSession()` or whether Supabase's client/session state can be reused more efficiently.

**TL;DR:** One interceptor attaches the access token; another centralizes authentication-error handling.

**Key mappings:** `frontend/src/services/api.js`, `frontend/src/context/AuthContext.jsx`.

---

## 5. Qno: How is user data isolated so one user cannot read another user's stocks?

**Polished answer:**

The backend obtains the authenticated user's ID from the verified Supabase JWT and stores it on `req.user.id`. Queries for authenticated portfolio data then filter by that value, for example the stock list uses `.eq('user_id', req.user.id)`. New stock records are also inserted with `user_id: req.user.id` rather than trusting a client-supplied owner ID.

For update and delete, the code first loads the row and then checks `stock.user_id !== req.user.id`. If the owner does not match, it returns HTTP 403.

That is application-level authorization. A stronger design would also enable **Row Level Security (RLS)** in Postgres and define policies based on the JWT subject, so the database itself rejects cross-user access even if an application query is accidentally written without the filter.

Using the backend Supabase **service-role key** also means RLS protections can be bypassed depending on how the client is configured, so explicit authorization is especially important in this repository.

**TL;DR:** The server derives ownership from the verified JWT and filters/checks `user_id`; RLS should add a second security boundary.

**Key mappings:** `backend/src/middleware/auth.js`, `backend/src/routes/stocks.js`, `backend/src/routes/assets.js`, `backend/.env.example`.

---

## 6. Qno: Explain the complete "Add Stock" flow, including validation, external API calls, fallback behavior, and database insertion.

**Polished answer:**

The Portfolio page collects company name, ticker, shares, and buy price. It converts numeric form values with `parseFloat` and posts the data to `/stocks`.

The backend route reads the request body, parses the numeric values, and calls `stockPriceService.getStockQuote(ticker)`. Finnhub returns a quote object where the current price is in field `c`. If the quote is valid and greater than zero, the backend stores that value; otherwise it falls back to the user's buy price so the position can still be created when the provider is unavailable or rate-limited.

The server then constructs the canonical row, normalizing the ticker with `toUpperCase().trim()`, attaching `user_id`, setting `is_in_watchlist` to false, and recording `last_updated`. `Stock.create()` inserts the row into Supabase and returns the created record.

A production version should add stronger server-side validation for all fields, check that the ticker exists, validate upper/lower bounds, and use a more explicit pricing freshness state instead of silently treating the buy price as the market price.

**TL;DR:** Form -> POST `/stocks` -> validate/normalize -> fetch Finnhub quote -> fallback if necessary -> attach user ID -> insert in Supabase.

**Key mappings:** `frontend/src/pages/Portfolio.jsx`, `frontend/src/services/api.js`, `backend/src/routes/stocks.js`, `backend/src/services/stockPriceService.js`, `backend/src/models/Stock.js`.

---

## 7. Qno: How do you integrate third-party APIs safely and reliably in this project?

**Polished answer:**

Third-party calls are isolated mostly inside backend services. `stockPriceService` uses Axios for Finnhub and `assetPriceService` uses Axios for MetalPriceAPI. The API keys are read from server-side environment variables, so the browser does not need the provider credentials.

The services also use seven-second request timeouts so a slow provider does not keep a serverless request hanging indefinitely. `getStockQuote()` returns `null` on failure, while historical-data fetching throws its error to the caller. The stock creation route uses the quote result with a fallback to `buy_price`.

For a stronger production system I would add retries with exponential backoff for transient failures, rate-limit handling, structured logging, response validation, caching, circuit breakers, and explicit stale-data semantics. I would also prevent a provider outage from silently presenting stale data as current data.

**TL;DR:** Keep provider calls server-side, isolate them behind services, use timeouts, and design explicit failure semantics.

**Key mappings:** `backend/src/services/stockPriceService.js`, `backend/src/services/assetPriceService.js`, `backend/.env.example`.

---

## 8. Qno: Explain the periodic stock-price update mechanism. What is good about it and what is problematic at scale?

**Polished answer:**

`updateStockPrices()` queries stocks that either have positive shares or are in the watchlist. It loops over those records, calls Finnhub for each ticker, updates `current_price` and `last_updated`, and waits one second before the next request. `startPeriodicUpdates()` runs the update immediately and then schedules another run every five minutes with `setInterval`.

The good part is that the logic is centralized and only active positions/watchlisted rows are considered. The one-second delay also reduces the chance of aggressively hitting provider limits.

The major issue is architecture. A long-lived `setInterval` inside a serverless environment such as Vercel is unreliable because function instances are ephemeral. It also creates an N+1 pattern: one database read, followed by one external request and one database update per stock. With many users and holdings this becomes slow and expensive.

A better design would use a dedicated scheduled job, batch provider endpoints where available, cache quotes, and perform bulk database updates.

**TL;DR:** The current polling loop works conceptually but is not a scalable serverless architecture.

**Key mappings:** `backend/src/services/stockPriceService.js`, `backend/src/server.js`, `backend/vercel.json`.

---

## 9. Qno: Why does `app.listen()` matter when deploying Express to Vercel serverless functions?

**Polished answer:**

In a traditional Node deployment, Express owns a long-lived TCP server through `app.listen(PORT)`. In a serverless platform such as Vercel, the platform owns the HTTP lifecycle and invokes the exported handler when a request arrives.

The repository addresses this by exporting the Express `app` and conditionally calling `app.listen()` only when it is not running as the Vercel serverless environment. The separate `api/index.js` entry point imports the Express app.

The important architectural distinction is **server process ownership**. In local development, the application owns the process and opens port 5000. In serverless production, Vercel owns the process and routing lifecycle.

This also affects background jobs, connection reuse, in-memory caches, and startup assumptions: anything depending on a permanent Node process must be reconsidered for serverless execution.

**TL;DR:** Local Express can listen on a port; serverless functions should export a handler and let the platform own the request lifecycle.

**Key mappings:** `backend/src/server.js`, `api/index.js`, `backend/vercel.json`, `implementation_plan.md`.

---

## 10. Qno: What are the most important correctness problems you found in the repository, and how would you fix them?

**Polished answer:**

The biggest issues are not cosmetic; several create broken or misleading behavior.

1. The frontend calls endpoints such as `/stocks/:ticker/quote`, `/profile`, `/metrics`, `/news`, `/calendar/earnings`, `/calendar/ipo`, and `/market/status`, but the backend route file does not expose those endpoints. I would either implement them or remove/disable the UI until the contract is complete.
2. The stock-history UI requests `/historical`, while the backend exposes `/:ticker/history`. That is an API-contract mismatch.
3. The backend history endpoint currently generates random synthetic prices rather than real historical data, while the service layer already contains a real Finnhub candle implementation.
4. `OtherAssets` uses stock CRUD routes for edit/delete even though its data model is `assets`; it should expose asset-specific update/delete routes.
5. `stockSlice.js` is empty while `store.js` configures Redux from it, so Redux is effectively unused/misconfigured.
6. The frontend logs the Supabase anon key in `App.jsx`; even though an anon key is intended for client use, secrets/tokens should not be casually logged.

The `implementation_plan.md` and `walkthrough.md` correctly identify several deployment bugs, but the current source should still be validated before claiming those fixes are present everywhere.

**TL;DR:** Before an interview, know the difference between the intended architecture and the repository's actual runtime contract.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/services/stockPriceService.js`, `frontend/src/pages/StockDetail.jsx`, `frontend/src/pages/MarketCalendar.jsx`, `frontend/src/pages/OtherAssets.jsx`, `frontend/src/store/stockSlice.js`, `frontend/src/App.jsx`.

---

# Top 25

## 11. Qno: Why use React Context for authentication instead of passing the user prop through every component?

**Polished answer:**

Authentication is cross-cutting state. Many unrelated components need to know whether a user is logged in or need access to functions such as `signOut`. Passing `user` and authentication handlers through every route would create prop drilling and tightly couple components to the page hierarchy.

`AuthContext` exposes a stable interface through `useAuth()`. `AuthProvider` owns the Supabase session lifecycle, while consumers such as `ProtectedRoute`, `Homepage`, and `Sidebar` only depend on the context API.

The context also hides implementation details: callers do not need to know whether the auth provider is Supabase, another identity provider, or a custom backend.

I would keep authentication in Context but use local component state for page-specific data. For large applications, I would also separate auth state from unrelated global state rather than putting everything into one context.

**TL;DR:** Context is appropriate for cross-cutting auth state and avoids prop drilling.

**Key mappings:** `frontend/src/context/AuthContext.jsx`, `frontend/src/components/auth/ProtectedRoute.jsx`, `frontend/src/pages/Homepage.jsx`, `frontend/src/components/layout/Sidebar.jsx`.

---

## 12. Qno: What is the purpose of `ProtectedRoute`, and why does it wait for `loading` before redirecting?

**Polished answer:**

`ProtectedRoute` implements client-side route gating. It reads `{ user, loading }` from `useAuth()`. While the initial Supabase session is being loaded, it displays a loading state. If loading is complete and no user exists, it navigates back to `/`. Otherwise it renders the protected child component.

Waiting for `loading` is critical. Without it, the initial render could see `user = null` before Supabase has restored a persisted session and incorrectly redirect an already authenticated user to the login page.

This protection improves UX, but it is not the real security boundary. A malicious client can bypass React routing entirely. The backend must still verify the token and enforce authorization on every protected API operation.

**TL;DR:** `ProtectedRoute` prevents premature UI access; backend auth remains the real security boundary.

**Key mappings:** `frontend/src/components/auth/ProtectedRoute.jsx`, `frontend/src/context/AuthContext.jsx`.

---

## 13. Qno: Explain the React `useEffect` lifecycle in this project. Where is cleanup necessary?

**Polished answer:**

The project uses `useEffect` for external side effects: initializing Supabase auth, subscribing to auth changes, listening for theme events, fetching API data, and creating timers.

Cleanup is necessary whenever an effect registers something that persists outside React's render lifecycle. `AuthContext` unsubscribes from the Supabase auth listener. Dashboard removes theme listeners and clears its one-second clock interval. Other pages remove custom theme listeners as well.

A key interview point is that `useEffect` is not a general-purpose place for computation. The dependency array describes when the effect should rerun. External subscriptions, network requests, timers, and event listeners are appropriate uses; pure derived calculations often belong directly in render or `useMemo` when expensive.

One issue in the repository is the storage-listener cleanup in `App.jsx`: the code registers an anonymous callback but later removes `handleThemeChange`, so the same function reference is not used for cleanup. That can leave a listener attached.

**TL;DR:** Effects synchronize React with external systems; cleanup must remove every subscription/timer using the same callback/reference.

**Key mappings:** `frontend/src/App.jsx`, `frontend/src/context/AuthContext.jsx`, `frontend/src/pages/Dashboard.jsx`, `frontend/src/pages/Portfolio.jsx`.

---

## 14. Qno: Why does the project use both React state and Redux, and what would you change?

**Polished answer:**

Most of the actual application state is managed with local React state: stocks, assets, loading flags, modal state, search state, selected period, and theme. Authentication is in Context. The Redux store exists, but `stockSlice.js` is empty and `store.js` imports it as `todoReducer`, so Redux is not meaningfully integrated into the current application.

I would choose one coherent global-state strategy. For this project, React Query/TanStack Query or a lightweight Redux Toolkit slice could be more appropriate for server-state such as stocks and quotes, while local UI state remains in components and auth remains in a dedicated context/provider.

The key distinction is **server state vs UI state**. Stock rows are fetched from an API, can become stale, need refetching/caching, and may be shared across components. Modal visibility does not need global storage.

**TL;DR:** The current Redux layer is effectively dead code; centralize server state only when shared caching/refetching justifies it.

**Key mappings:** `frontend/src/store/store.js`, `frontend/src/store/stockSlice.js`, `frontend/src/pages/Portfolio.jsx`, `frontend/src/pages/Dashboard.jsx`.

---

## 15. Qno: How does stock search/autocomplete work, and why is debouncing useful?

**Polished answer:**

The Portfolio page keeps `searchQuery`, `searchResults`, `isSearching`, and `showDropdown` in React state. Once the user types at least two characters, it calls `/stocks/search?q=...`. The backend forwards the query to Finnhub, filters the results toward common US exchanges, and returns at most ten entries.

Debouncing is intended to reduce API calls while the user is typing. Instead of sending one request per keystroke, the design waits roughly 300 ms before searching.

However, the current `handleSearchChange` creates a timeout inside the event handler and returns a cleanup function that React does not consume because the function itself is not an effect. As a result, this is not a reliable debounce implementation and can still produce multiple requests.

A robust implementation would store the timer in a ref, use a dedicated `useEffect` on `searchQuery`, or use a debounce utility. I would also cancel stale requests so older responses cannot overwrite newer search results.

**TL;DR:** Debounce limits search traffic; implement it with `useEffect`/`useRef` or a proven utility, not a returned cleanup from an event handler.

**Key mappings:** `frontend/src/pages/Portfolio.jsx`, `backend/src/routes/stocks.js`.

---

## 16. Qno: Explain the portfolio value and gain/loss calculations. What formula does the project use?

**Polished answer:**

For a position, portfolio market value is:

`market value = current price × quantity`

The project's dashboard treats `shares` as quantity for stocks and `grams` as quantity for precious-metal assets. Unrealized gain/loss in absolute terms is:

`gain/loss = (current price - buy price) × quantity`

The percentage return should be based on cost basis:

`return % = gain/loss ÷ (buy price × quantity) × 100`

The backend summary uses the equivalent form `totalGain / (totalValue - totalGain) × 100`, because `totalValue - totalGain` reconstructs total cost in that simplified model.

A major interview point is that this model is only valid for a simple single-lot position. A real portfolio with multiple purchases, partial sells, fees, dividends, splits, and realized gains needs a transaction ledger and a cost-basis method such as FIFO, LIFO, or average cost.

**TL;DR:** The repository uses simple mark-to-market math; production-grade portfolio accounting needs transaction-level cost basis.

**Key mappings:** `frontend/src/pages/Dashboard.jsx`, `frontend/src/pages/Portfolio.jsx`, `backend/src/routes/stocks.js`, `frontend/src/components/dashboard/PortfolioAnalytics.jsx`.

---

## 17. Qno: Why is `Promise.allSettled()` used in the stock-detail and market-calendar pages instead of `Promise.all()`?

**Polished answer:**

`Promise.all()` rejects as soon as any one promise rejects. `Promise.allSettled()` waits for every promise and tells us whether each operation was fulfilled or rejected.

That behavior fits pages that aggregate independent external data. For stock detail, quote, profile, metrics, and news are separate requests. If metrics fail but quote succeeds, the UI can still show the price. The code maps rejected results to `null` or an empty list.

The same idea is used for the market calendar so one failed data provider does not necessarily blank the whole page.

The trade-off is that the UI must explicitly distinguish missing data from successful empty data. I would also add per-endpoint error telemetry and possibly retry selected critical requests.

**TL;DR:** `allSettled` provides graceful degradation for independent requests; `all` is better when every dependency is mandatory.

**Key mappings:** `frontend/src/pages/StockDetail.jsx`, `frontend/src/pages/MarketCalendar.jsx`.

---

## 18. Qno: Why should API keys stay in backend environment variables, and how are frontend environment variables different?

**Polished answer:**

Vite exposes variables prefixed with `VITE_` to browser code. That means `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, and `VITE_API_BASE_URL` should be treated as public application configuration rather than secrets.

Provider credentials such as `FINNHUB_API_KEY`, `NEWS_API_KEY`, `METAL_API_KEY`, and the Supabase service-role key belong on the backend. The browser should never call those providers directly with privileged credentials.

The Supabase anon key is intentionally designed for client-side use when combined with proper database policies, but logging it unnecessarily is still poor operational hygiene. The service-role key is highly privileged and should never be shipped to the browser.

I would also validate required variables at startup and fail fast instead of silently using placeholder credentials in production.

**TL;DR:** `VITE_*` is browser-visible config; backend secrets stay server-side and must be validated at startup.

**Key mappings:** `frontend/src/config/supabase.js`, `frontend/.env.example`, `backend/.env.example`, `backend/src/config/supabase.js`.

---

## 19. Qno: How does CORS work in this application, and what exactly does it protect?

**Polished answer:**

The Express server uses the `cors` middleware and defines an allowlist containing known frontend URLs plus an optional `FRONTEND_URL` environment variable. It also handles preflight `OPTIONS` requests with the same CORS configuration.

CORS is a browser security mechanism. It controls which origins are allowed to make cross-origin browser requests and read their responses. It does **not** authenticate users and does not prevent a non-browser client such as curl from calling the backend.

The authentication header is therefore still required for protected endpoints. CORS only controls browser-origin permissions.

For production, I would keep an explicit origin allowlist, avoid `*` with credentials, and ensure the configured frontend origin exactly matches the deployed origin. I would also avoid hardcoding multiple old Vercel URLs once deployment is standardized.

**TL;DR:** CORS controls browser cross-origin access; JWT authentication controls identity and authorization.

**Key mappings:** `backend/src/server.js`, `backend/.env.example`, `walkthrough.md`.

---

## 20. Qno: What is the difference between `models`, `routes`, and `services` in the backend?

**Polished answer:**

The repository follows a lightweight layered backend pattern.

- **Routes** define HTTP contracts, parse request parameters, enforce middleware, choose status codes, and coordinate the request flow.
- **Models** provide database-oriented operations such as `findAll`, `findByPk`, `create`, `update`, and `destroy`.
- **Services** encapsulate external business integrations such as Finnhub and MetalPriceAPI.

This separation improves testability and makes provider changes easier. For example, if I replaced Finnhub with another quote vendor, ideally only the price service would need to change.

The current implementation is partial: some routes call Supabase directly rather than consistently going through the model layer. That creates duplicated query logic and makes it harder to enforce common behavior. I would standardize one data-access layer and one provider-service boundary.

**TL;DR:** Routes orchestrate HTTP; models access data; services integrate external systems.

**Key mappings:** `backend/src/routes/`, `backend/src/models/`, `backend/src/services/`.

---

## 21. Qno: Why is explicit server-side validation necessary even though the frontend validates the forms?

**Polished answer:**

Frontend validation is primarily a UX feature. It can prevent obvious mistakes before a network request, but the server must assume the client is untrusted because requests can be crafted manually.

For stocks and assets, the backend should validate required fields, numeric types, finite values, positivity/ranges, ticker format, and ownership. It should also validate domain rules such as whether a target price is meaningful and whether an asset name is one of the supported metals.

This project uses `parseFloat` and a few checks, but the validation is incomplete and inconsistent between endpoints. For example, the stock creation route does not explicitly reject malformed values before using them in arithmetic.

I would centralize request validation with a schema library such as Zod/Joi/Ajv or dedicated validators, and return structured 400 responses.

**TL;DR:** Client validation improves UX; server validation enforces correctness and security.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/routes/assets.js`, `frontend/src/pages/Portfolio.jsx`, `frontend/src/components/modals/AddStockModal.jsx`.

---

## 22. Qno: Why does the backend add `last_updated`, and how would you use freshness correctly in a financial application?

**Polished answer:**

`last_updated` records when the stored quote was last refreshed. That timestamp is important because a market price is time-sensitive. It lets the UI distinguish a recently fetched price from stale data and helps support debugging, monitoring, and cache policies.

The current dashboard mostly displays the value without making staleness explicit. A stronger design would attach a quote source and freshness timestamp, for example `price_source`, `price_fetched_at`, and possibly `is_stale`.

For a financial application, I would avoid representing stale data as if it were current. A provider outage should result in a visible stale indicator rather than silently using the buy price and pretending that is the current market price.

**TL;DR:** Timestamps are part of data quality, not just metadata; financial UI must expose stale-price semantics.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/services/stockPriceService.js`, `frontend/src/components/dashboard/StockDetail.jsx`.

---

## 23. Qno: How would you redesign the current database model for real portfolio accounting?

**Polished answer:**

The current `stocks` table stores a position snapshot: name, ticker, shares, buy price, current price, target price, watchlist flag, user ID, and timestamp. That is simple, but it cannot accurately represent transaction history.

I would introduce a `transactions` table containing user ID, asset ID, transaction type (`BUY`, `SELL`, `DIVIDEND`, etc.), quantity, execution price, fees, timestamp, and external reference if needed. The holdings table could then be treated as a materialized/derived position snapshot.

This enables accurate realized and unrealized P&L, average cost, partial sells, multiple purchase lots, fees, dividends, splits, and auditability. It also gives me an immutable source of truth.

For a placement interview, the key design principle is: **store events/transactions when history matters; derive current state from those events**.

**TL;DR:** Replace single-row position snapshots with transaction history plus a derived holdings layer.

**Key mappings:** `backend/src/models/Stock.js`, `backend/src/routes/stocks.js`, `frontend/src/components/dashboard/TransactionHistory.jsx`.

---

## 24. Qno: How would you design the API contract for stocks so the frontend and backend do not drift apart?

**Polished answer:**

I would define explicit REST resources and document the request/response schemas. For example:

- `GET /api/stocks`
- `POST /api/stocks`
- `GET /api/stocks/:id`
- `PUT /api/stocks/:id`
- `DELETE /api/stocks/:id`
- `GET /api/stocks/search?q=...`
- `GET /api/stocks/:ticker/quote`
- `GET /api/stocks/:ticker/history?period=...`

Then I would use OpenAPI or a shared TypeScript schema so the frontend cannot silently call endpoints that do not exist.

This repository has exactly that problem: the UI references several stock-detail and calendar endpoints that the backend route file does not currently define, and `/history` vs `/historical` is inconsistent. Contract tests or generated clients would catch this during CI.

**TL;DR:** Define a source-of-truth API contract and validate both client and server against it.

**Key mappings:** `backend/src/routes/stocks.js`, `frontend/src/pages/StockDetail.jsx`, `frontend/src/pages/MarketCalendar.jsx`.

---

## 25. Qno: Why is it important to distinguish `401 Unauthorized` from `403 Forbidden` in this project?

**Polished answer:**

HTTP 401 means the request does not have valid authentication credentials, while 403 means the identity is known but the caller is not allowed to perform the requested action.

In this project, `authenticateUser` returns 401 when the Bearer token is missing or invalid. The stock update/delete routes return 403 when a valid user tries to modify a stock whose `user_id` does not match the authenticated user.

That distinction matters both semantically and operationally. The frontend can treat a 401 as a session problem and redirect to login. A 403 should not blindly trigger re-authentication because the user's identity may be valid; the resource is simply not accessible.

**TL;DR:** 401 = authenticate; 403 = authenticated but not authorized.

**Key mappings:** `backend/src/middleware/auth.js`, `backend/src/routes/stocks.js`, `frontend/src/services/api.js`.

---

# Top 50

## 26. Qno: Why is the current `Stock` model abstraction useful, and what would you improve about it?

**Polished answer:**

The `Stock` model hides Supabase query construction behind methods like `findAll`, `findByPk`, `create`, and `count`. That gives route handlers a more domain-oriented interface and creates one place for database behavior.

`findByPk` also returns helper methods such as `update()` and `destroy()`, which mimics a traditional ORM-style object interface even though the underlying implementation is Supabase.

The main weakness is that `findAll` only supports a small custom subset of operators (`eq`, `gt`, `lt`) and the routes do not consistently use the model. I would either build a clearly scoped repository interface or embrace Supabase directly and avoid creating a partial ORM abstraction. Partial abstractions often become harder to maintain than either simple queries or a real data-access layer.

**TL;DR:** The model improves separation, but a half-ORM abstraction can become unnecessary complexity.

**Key mappings:** `backend/src/models/Stock.js`, `backend/src/models/Asset.js`.

---

## 27. Qno: What is the N+1 problem in `updateStockPrices`, and how would you fix it?

**Polished answer:**

The service first loads all relevant stocks, then performs one external quote request per stock and one database update per stock. With N holdings, that creates roughly N provider calls plus N updates after one initial read.

That is acceptable for a tiny demo but scales poorly. Latency grows linearly, provider quotas are consumed quickly, and a single scheduled run can become expensive.

I would first deduplicate tickers, because multiple users may hold the same symbol. Then I would batch quotes if the provider supports it, or use a shared quote cache keyed by ticker. Database writes should be batched where possible, and the job should update only changed/stale prices.

For large scale, I would separate quote ingestion from per-user portfolio computation so one market quote can serve many users.

**TL;DR:** Deduplicate and cache market data; do not fetch the same ticker independently for every holding.

**Key mappings:** `backend/src/services/stockPriceService.js`.

---

## 28. Qno: How would caching improve this system?

**Polished answer:**

Stock prices are shared reference data. If ten thousand users own AAPL, the application should not call Finnhub ten thousand times. A shared cache can store the latest quote by ticker for a short TTL.

A Redis-style cache could use a key such as `quote:AAPL` with a short expiration. The quote service would check the cache first, call the provider on a miss, validate the response, then populate the cache. The database would retain the last persisted quote as durable state.

This reduces provider traffic, latency, and cost. The trade-off is stale data and cache invalidation. In a financial application I would explicitly model freshness and never rely on an unbounded cache.

**TL;DR:** Cache shared quote data by ticker, use a short TTL, and expose freshness rather than hiding staleness.

**Key mappings:** `backend/src/services/stockPriceService.js`, `backend/src/routes/stocks.js`.

---

## 29. Qno: What database indexes would you add for this project?

**Polished answer:**

The most common queries are user-scoped stock reads and direct primary-key access. I would ensure `stocks.id` is the primary key and add an index on `(user_id, is_in_watchlist, shares)` or a workload-specific variant for dashboard queries.

For assets, an index such as `(user_id, name)` could help if the application frequently filters or sorts by user and name. If search becomes database-backed, I would use an appropriate text-search index rather than `LIKE` across a large table.

Indexes should be justified by actual query patterns. Too many indexes slow inserts/updates and consume storage. I would inspect query plans and production metrics before creating every theoretically useful index.

**TL;DR:** Index for real access patterns, especially user-scoped queries and common filters.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/routes/assets.js`, `backend/src/models/Stock.js`.

---

## 30. Qno: Why should Row Level Security be enabled in Supabase for this application?

**Polished answer:**

RLS makes authorization a database-level rule rather than only an application convention. A policy can require that a row's `user_id` match the authenticated JWT subject.

That protects against bugs where a future developer forgets `.eq('user_id', req.user.id)` in a route. Even if the application layer makes a mistake, Postgres can reject the cross-user query.

The catch is that the backend currently uses the Supabase service-role key, which is privileged. That can bypass RLS, so the security model must be designed intentionally. One option is to use the service role only for narrowly trusted server operations while using the authenticated user context where RLS enforcement is desired.

**TL;DR:** RLS is defense in depth, but only works as an enforcement layer when queries use a role/session subject that RLS evaluates.

**Key mappings:** `backend/src/config/supabase.js`, `backend/src/middleware/auth.js`, `.env.example` files.

---

## 31. Qno: What race conditions or concurrency issues could occur when updating portfolio data?

**Polished answer:**

Suppose a user edits shares while a background price update is simultaneously writing `current_price`. If the update operation sends an entire object rather than a field-specific patch, one request could accidentally overwrite another request's changes.

The current `PUT` route spreads `req.body` into the update, which makes the endpoint especially sensitive to stale client state. I would prefer PATCH semantics with a whitelist of mutable fields, optimistic concurrency using a version/timestamp, or database-side conditional updates.

For transactions such as buy/sell, I would use database transactions so cash balance, transaction record, and holdings update atomically.

**TL;DR:** Avoid broad object overwrites; use field-level updates, optimistic locking, and database transactions for financial mutations.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/models/Stock.js`.

---

## 32. Qno: Why is `PUT /stocks/:id` risky in its current form, and when would you prefer `PATCH`?

**Polished answer:**

The route accepts `...req.body` and writes it directly to the stock. That means a client can potentially change fields that should be server-managed, such as `user_id`, `current_price`, `last_updated`, or watchlist state if those fields are not explicitly blocked.

For an update endpoint I would whitelist fields the client is allowed to change: perhaps `name`, `ticker`, `shares`, `buy_price`, and `target_price`. Server-maintained fields would be controlled only by trusted backend workflows.

`PATCH` is a better semantic fit when updating only selected fields. `PUT` implies replacement of the resource representation, although many APIs use it loosely.

**TL;DR:** Whitelist mutable fields and prefer partial-update semantics for targeted edits.

**Key mappings:** `backend/src/routes/stocks.js`, `frontend/src/pages/Portfolio.jsx`.

---

## 33. Qno: How would you prevent duplicate or invalid holdings?

**Polished answer:**

I would enforce domain constraints at multiple layers. Normalize the ticker to uppercase, trim whitespace, and validate the ticker format. Then define whether a user may have multiple rows for the same ticker. If not, enforce a unique database constraint such as `(user_id, ticker, is_in_watchlist)` or a domain-appropriate partial uniqueness rule.

For true portfolio accounting, I would actually allow multiple transactions rather than multiple mutable position rows. The position snapshot would be derived from those transactions.

I would also validate numeric fields as finite positive numbers and reject `NaN`, infinity, zero shares where not allowed, and negative prices.

**TL;DR:** Normalize, validate, and enforce uniqueness in the database; business rules should not depend only on UI checks.

**Key mappings:** `backend/src/routes/stocks.js`, `frontend/src/pages/Portfolio.jsx`.

---

## 34. Qno: What is wrong with using `parseFloat` as the main financial numeric strategy?

**Polished answer:**

JavaScript `number` is binary floating-point. It is fine for many UI calculations, but money computations can accumulate rounding errors. `parseFloat` also does not guarantee a valid finite business value by itself.

For production financial accounting, I would validate numbers with `Number.isFinite`, define explicit decimal precision, and prefer decimal arithmetic or integer minor units for currencies. Database numeric/decimal types are also preferable to floating-point columns for monetary values.

The repository is a portfolio tracker rather than a brokerage ledger, so the current approach is understandable for a demo, but I would state this limitation clearly in an interview.

**TL;DR:** `parseFloat` is input parsing, not a financial precision strategy.

**Key mappings:** `frontend/src/pages/Portfolio.jsx`, `frontend/src/pages/OtherAssets.jsx`, `backend/src/routes/stocks.js`, `backend/src/routes/assets.js`.

---

## 35. Qno: Explain the gold/silver price conversion in the asset flow.

**Polished answer:**

The assets route maps `Gold` to ticker `XAU` and `Silver` to `XAG`, then calls MetalPriceAPI with USD as the base currency. The returned `rates.USDXAU` or `rates.USDXAG` value is divided by `28.34` before being stored as the price per gram.

The important interview point is unit conversion. Gold and silver prices are commonly quoted per troy ounce, while the UI stores quantity in grams. Since one troy ounce is approximately 31.1035 grams, a conversion to grams should be based on the exact provider semantics.

The repository uses `28.34`, so I would not defend that number without verifying the API's unit contract. I would centralize the conversion constant, document units explicitly, and add tests for the conversion.

**TL;DR:** The flow maps XAU/XAG into a per-gram value; unit assumptions must be verified and tested.

**Key mappings:** `backend/src/routes/assets.js`, `backend/src/services/assetPriceService.js`, `frontend/src/pages/OtherAssets.jsx`.

---

## 36. Qno: Why is `Math.random()`-generated price history a serious correctness problem?

**Polished answer:**

The backend `/stocks/:ticker/history` route starts from a current quote and then generates thirty-one pseudo-random daily values. That is not market history; it is synthetic demo data.

Presenting synthetic values as historical market prices is a correctness problem because users may interpret the chart as real information. The repository already contains `getHistoricalData()` in `stockPriceService.js`, which calls Finnhub's candle endpoint and transforms timestamps and close prices. That service should be used instead.

I would also fix the API naming mismatch between the backend `/:ticker/history` route and the frontend `/:ticker/historical` request, then add contract tests that verify the response shape.

**TL;DR:** Never label synthetic demo data as real financial history; use the provider-backed candle service.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/services/stockPriceService.js`, `frontend/src/components/dashboard/StockDetail.jsx`.

---

## 37. Qno: What would you do about time zones and market hours in this project?

**Polished answer:**

The dashboard hardcodes a market schedule described as 7:00 PM to 2:30 AM IST and uses `new Date()` local time. That creates two problems: the assumption may not match the market being represented, and daylight-saving rules can change the offset for markets such as the US.

A robust design stores timestamps in UTC, uses a timezone-aware library or `Intl` with an explicit IANA timezone, and determines market sessions from an authoritative exchange calendar rather than hardcoded hours.

For a US market shown to an Indian user, for example, I would represent exchange-local trading windows and separately render them in the user's selected timezone.

**TL;DR:** Store UTC, convert at the edges, and use exchange calendars instead of hardcoded local clock rules.

**Key mappings:** `frontend/src/pages/Dashboard.jsx`, `frontend/src/pages/MarketCalendar.jsx`, `frontend/src/pages/Settings.jsx`.

---

## 38. Qno: How would you handle API failures differently from validation errors and server bugs?

**Polished answer:**

I would classify failures by responsibility and HTTP semantics.

- **400:** invalid client input or malformed parameters.
- **401:** missing or invalid authentication.
- **403:** authenticated but not allowed to access the resource.
- **404:** resource does not exist.
- **409:** a domain conflict such as a duplicate position.
- **429:** rate limit exceeded.
- **502/503:** upstream provider failure or temporary service unavailability.
- **500:** unexpected server-side fault.

The current code sometimes returns 400 for broad errors and 500 for several unrelated cases. I would add centralized Express error handling and provider-specific error mapping so the frontend can respond appropriately without parsing arbitrary strings.

**TL;DR:** Error classes should be explicit, predictable, and mapped to HTTP semantics.

**Key mappings:** `backend/src/routes/stocks.js`, `backend/src/routes/assets.js`, `backend/src/routes/news.js`, `frontend/src/services/api.js`.

---

## 39. Qno: How would you improve the logging strategy in this repository?

**Polished answer:**

The project uses many `console.log` statements for debugging, including environment checks and token metadata. That is useful during development but not sufficient for production observability.

I would switch to structured logs with fields such as request ID, user ID, endpoint, latency, upstream provider, status code, and error code. Sensitive values such as tokens, API keys, passwords, and full request payloads should never be logged.

I would also add metrics for request rate, latency, external API failure rate, quote freshness, and database errors. For asynchronous jobs, I would log job start/end and per-batch results rather than every low-level loop iteration.

**TL;DR:** Production logs should be structured, correlated, useful for debugging, and free of credentials.

**Key mappings:** `backend/src/middleware/auth.js`, `backend/src/routes/`, `backend/src/services/`, `frontend/src/services/api.js`.

---

## 40. Qno: How would you make the frontend API layer more maintainable?

**Polished answer:**

The project already has a shared Axios instance, which is a good start, but several components still create their own Axios client or call Axios directly. That fragments request configuration and error handling.

I would centralize all API access into a typed service layer or hooks. For example, `stockApi.search()`, `stockApi.create()`, `stockApi.getQuote()`, and `marketApi.getCalendar()` would hide URL construction from components.

I would then use a server-state library for caching, retries, stale-time configuration, and request deduplication. The UI components would receive data and states rather than building HTTP details themselves.

**TL;DR:** Keep network concerns in one API layer and keep pages focused on presentation/state orchestration.

**Key mappings:** `frontend/src/services/api.js`, `frontend/src/pages/Portfolio.jsx`, `frontend/src/pages/StockDetail.jsx`, `frontend/src/pages/News.jsx`.

---

## 41. Qno: What is the role of `createPortal` in the Portfolio and OtherAssets pages?

**Polished answer:**

The add/edit dialogs are rendered with `createPortal(..., document.body)`. Portals let a component render its DOM outside its normal parent hierarchy while remaining part of the same React tree.

This is useful for modals because parent containers may have overflow, z-index, transform, or stacking-context rules that can clip or misplace the dialog. Rendering at the body level makes full-screen overlays easier to position.

The modal still participates in React context and event propagation, so it does not become a completely separate application.

For accessibility I would add focus trapping, restore focus to the trigger after close, support Escape-to-close, and use correct dialog semantics.

**TL;DR:** Portals solve DOM stacking/overflow problems without leaving the React component tree.

**Key mappings:** `frontend/src/pages/Portfolio.jsx`, `frontend/src/pages/OtherAssets.jsx`.

---

## 42. Qno: Why is component decomposition important here, and which components would you extract further?

**Polished answer:**

The project already extracts reusable pieces such as `PortfolioAnalytics`, `StockDetail`, `SearchBar`, UI primitives, `ProtectedRoute`, and layout components. However, several pages remain very large—especially Dashboard, Portfolio, OtherAssets, and Homepage.

I would extract data-fetching hooks, stat cards, tables, modals, validation schemas, and repeated theme classes. For example, Portfolio could become `usePortfolioStocks`, `StockSearch`, `AddStockDialog`, `EditStockDialog`, `PortfolioSummary`, and `HoldingsTable`.

The goal is not to maximize file count. The goal is to isolate units with clear responsibilities and make them independently testable.

**TL;DR:** Extract by responsibility and reuse, not merely because a component is long.

**Key mappings:** `frontend/src/pages/Dashboard.jsx`, `frontend/src/pages/Portfolio.jsx`, `frontend/src/pages/OtherAssets.jsx`.

---

## 43. Qno: What is wrong with storing derived values like `assets = [...stocks, ...metals]` in state?

**Polished answer:**

`assets` is fully derived from `stocks` and `metals`, so it does not need independent state. The current code updates it through a `useEffect` whenever those arrays change.

That creates an extra render cycle: first stocks/metals update, then the effect runs, then assets updates. It also creates a second source of truth that can temporarily be inconsistent.

I would derive it directly during render or with `useMemo` if the operation is expensive:

`const assets = useMemo(() => [...stocks, ...metals], [stocks, metals]);`

For such a small array, even plain computation is sufficient. The interview principle is: **do not store data in state when it can be deterministically derived from other state**.

**TL;DR:** Derived state should usually be computed, not synchronized with another effect.

**Key mappings:** `frontend/src/pages/Dashboard.jsx`.

---

## 44. Qno: How would you optimize Dashboard rendering without prematurely optimizing it?

**Polished answer:**

First I would measure. The Dashboard performs API fetches, recalculates totals on render, renders tables, and includes charts. For a small portfolio that is fine.

If profiling shows unnecessary work, I would extract reusable components, memoize expensive derived calculations, and avoid recreating large objects/functions when child components care about referential equality. `useMemo` is appropriate for expensive calculations such as chart datasets, while `React.memo` can help for stable presentational children.

The bigger optimization is server-state architecture: avoid refetching identical market data and use caching. Rendering optimizations are less valuable if the backend still performs unnecessary provider calls.

**TL;DR:** Measure first; optimize the data layer before chasing tiny React render costs.

**Key mappings:** `frontend/src/pages/Dashboard.jsx`, `frontend/src/components/dashboard/PortfolioAnalytics.jsx`, `frontend/src/services/api.js`.

---

## 45. Qno: How would you test this project?

**Polished answer:**

I would use multiple test levels.

**Unit tests:** portfolio calculations, metal-unit conversion, response transformation, validation functions, and price-freshness logic.

**Component tests:** `ProtectedRoute`, login state transitions, stock search behavior, modal validation, error/loading states, and dashboard rendering for empty/normal/failure data.

**API/integration tests:** authentication middleware, ownership checks, CRUD endpoints, quote-service failure handling, and response schemas.

**End-to-end tests:** sign in, add stock, edit/delete a stock, dashboard calculation, and navigation across protected routes.

I would also add contract tests for every frontend endpoint, because the repository currently contains several client/server mismatches.

**TL;DR:** Test calculations and security rules at unit/integration level, then cover critical user flows end-to-end.

**Key mappings:** `frontend/src/`, `backend/src/`, missing test suite should be a clear improvement item.

---

## 46. Qno: How would you handle quote-provider rate limits and outages?

**Polished answer:**

The current service has a timeout and a one-second delay in the periodic updater, but that is not sufficient rate-limit management.

I would introduce a bounded retry policy with exponential backoff and jitter for transient failures, explicit handling for HTTP 429, and a maximum retry budget. I would cache quotes so repeated requests do not hit the provider unnecessarily. A circuit breaker can stop hammering a failing provider and allow recovery.

For the user-facing API, I would return the last known good quote with freshness metadata when appropriate, rather than silently converting the market price to the buy price. Background ingestion and user requests should be decoupled so the UI is not blocked by provider instability.

**TL;DR:** Rate limits require retries, backoff, caching, circuit breaking, and explicit stale-data semantics.

**Key mappings:** `backend/src/services/stockPriceService.js`, `backend/src/routes/stocks.js`.

---

## 47. Qno: How would you redesign the price-update system for a large production deployment?

**Polished answer:**

I would stop doing recurring work inside the web server. Instead, I would create a scheduled worker or managed cron job that periodically refreshes the global quote cache.

A scalable flow is: scheduler -> queue/job -> deduplicated ticker list -> provider batch requests -> validated quote cache -> durable persistence. Portfolio queries then read the latest known quote rather than triggering provider calls.

If multiple providers are available, the quote service can fail over or mark the source as degraded. Each quote can store `price`, `source`, `fetched_at`, and perhaps market-session metadata.

This architecture scales with the number of distinct tickers rather than the number of users.

**TL;DR:** Refresh shared market data asynchronously and serve portfolios from cached/durable quote state.

**Key mappings:** `backend/src/services/stockPriceService.js`, `backend/src/server.js`, deployment files.

---

## 48. Qno: What would you improve in the Vercel deployment setup?

**Polished answer:**

The intended deployment uses two Vercel projects: `frontend/` as a Vite site and `backend/` as a Node serverless project. The frontend points `VITE_API_BASE_URL` at the backend, and the backend receives provider/database secrets through environment variables.

The important deployment constraints are to export the Express app correctly, avoid unconditional `app.listen()` in serverless mode, configure CORS with the final frontend origin, and ensure React Router's client-side routes resolve to `index.html` on the frontend.

I would also remove dead deployment references, validate all environment variables in CI/CD, and add automated health checks after deployment. Most importantly, I would test the actual production route contract rather than relying only on the deployment plan.

**TL;DR:** Serverless deployment requires correct handler export, no permanent listener, correct CORS/env configuration, and post-deploy contract checks.

**Key mappings:** `api/index.js`, `backend/vercel.json`, `frontend/vercel.json`, `implementation_plan.md`, `walkthrough.md`.

---

## 49. Qno: How would you handle a product requirement to add transaction history and accurate P&L?

**Polished answer:**

I would change the source of truth from a single mutable position row to a transaction ledger. Each buy/sell event would be immutable and would record quantity, execution price, timestamp, and fees. A holding service would derive current quantity, cost basis, realized P&L, and unrealized P&L.

For a simple average-cost method, cost basis is recalculated after each buy. For FIFO, individual lots are consumed during sells. The choice should be explicit because it changes reported realized gain.

I would also separate **market price** from **execution price**. A quote is an external market observation; a transaction price is a user event. Mixing the two makes accounting ambiguous.

This is a strong interview answer because it shows understanding of both software architecture and the domain model.

**TL;DR:** Use immutable transactions as the source of truth and derive holdings/P&L from them.

**Key mappings:** `backend/src/models/Stock.js`, `backend/src/routes/stocks.js`, `frontend/src/components/dashboard/TransactionHistory.jsx`.

---

## 50. Qno: If an interviewer gives you 30 minutes to improve this project, what would you prioritize?

**Polished answer:**

I would prioritize correctness and security before visual polish.

First, I would reconcile the API contract: implement or remove the frontend calls that the backend does not expose, and fix `/history` versus `/historical` naming. Second, I would fix the asset CRUD routes so OtherAssets cannot accidentally update/delete stock records. Third, I would replace synthetic historical prices with the real Finnhub candle service. Fourth, I would tighten server-side validation and field whitelisting on updates. Fifth, I would remove sensitive/debug logging and standardize error handling.

After that I would add tests around auth/ownership, calculations, and endpoint contracts. Then I would address architectural scalability: quote caching, scheduled workers, and a transaction-based portfolio model.

This order demonstrates an engineering mindset: **make the system correct and safe, then make it fast and elegant**.

**TL;DR:** Fix broken contracts and security first; then tests; then scalability and cleanup.

**Key mappings:** `backend/src/routes/`, `backend/src/services/`, `backend/src/middleware/auth.js`, `frontend/src/pages/`, `implementation_plan.md`, `walkthrough.md`.

---

# High-value follow-up questions an interviewer can ask from the Top 50

These are not additional ranked questions; they are likely follow-ups to the questions above.

- Why did you choose Supabase instead of building your own auth system?
- What happens if the JWT expires while a request is in flight?
- Why should the server never trust a client-supplied `user_id`?
- How would you implement RLS policies for `stocks` and `assets`?
- What happens when two users hold the same stock ticker?
- How would you cache prices without serving dangerously stale data?
- How would you make quote refresh idempotent?
- How would you handle stock splits and dividends?
- How would you redesign the schema if crypto and mutual funds were added?
- How would you test a failing Finnhub/Marketaux dependency?
- Why is `Promise.allSettled()` appropriate for independent widgets?
- When would `useMemo`, `useCallback`, or `React.memo` actually help here?
- How would you migrate this from serverless to a containerized API?
- How would you introduce background jobs without making the frontend wait?
- How would you design the API if you expected 1 million users?

# Repository-specific red flags to memorize before the interview

1. **Frontend/backend endpoint drift:** several frontend requests have no matching backend route.
2. **History mismatch:** frontend requests `/historical`; backend exposes `/:ticker/history`.
3. **Synthetic history:** the backend history route generates random prices instead of using the real historical-data service.
4. **Asset CRUD mismatch:** `OtherAssets` edits/deletes through `/stocks/...` rather than `/assets/...`.
5. **Empty Redux slice:** Redux is imported but not actually used for meaningful state.
6. **Derived state:** Dashboard stores `assets` even though it is directly derived from `stocks + metals`.
7. **Debounce bug:** the Portfolio search handler returns a cleanup function from an event handler instead of managing the timer in an effect/ref.
8. **Server-side validation is incomplete:** client constraints cannot be treated as security.
9. **Broad update surface:** `PUT /stocks/:id` spreads request body into the database update.
10. **Serverless background-job mismatch:** `setInterval` is not a reliable production scheduler in serverless execution.
11. **Service-role risk:** backend Supabase credentials are highly privileged; defense-in-depth authorization and RLS strategy must be intentional.
12. **Debug logging:** token metadata and environment checks should not be left in production logging.
13. **Static/mock UI data:** Profile, Settings, and `NewsSection` contain static/demo content rather than live persisted data.
14. **Market-hours assumptions:** Dashboard uses a hardcoded local schedule rather than an authoritative exchange calendar/time-zone model.

# One-minute project pitch

> **Invest-Tracker is a full-stack portfolio tracker built with React/Vite, Express, Supabase, and external market-data APIs. Supabase Auth handles user sessions, the frontend forwards the JWT through an Axios interceptor, and the Express backend verifies the token before accessing user-scoped portfolio data. Stock creation enriches positions with live Finnhub quotes, while precious-metal assets use MetalPriceAPI. The dashboard combines stock and non-stock assets and computes portfolio value and unrealized returns. The main engineering challenges are authentication boundaries, user isolation, third-party API reliability, price freshness, serverless deployment, and keeping the frontend/backend API contract consistent. If I productionized it further, I would add database RLS, a transaction-based portfolio ledger, quote caching and scheduled workers, strict schema validation, contract tests, and stronger observability.**

# Final interview strategy

For every project question, answer in this order:

1. **What the current implementation does.**
2. **Why that design was chosen.**
3. **What trade-off it creates.**
4. **What is wrong or limited in the current code.**
5. **How you would improve it for production.**

That structure lets you demonstrate both that you understand the codebase deeply and that you can reason beyond the code that already exists.
