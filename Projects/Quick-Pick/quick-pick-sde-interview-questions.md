# Quick-Pick — SDE Interview Preparation: Top 50 Questions

This set is derived from the actual Quick-Pick project in the uploaded archive. The ranking is optimized for an SDE interview where the interviewer wants to test whether you can **own the system end-to-end**, explain trade-offs, identify correctness/security problems, and propose production-grade improvements.

## Project snapshot

- **Frontend:** React 18, Vite, React Router, Redux Toolkit, Axios, Tailwind CSS, Radix UI components.
- **Backend:** Node.js, Express, Mongoose/MongoDB.
- **Authentication:** bcrypt password hashing, JWT, HTTP-only cookie, authentication middleware, admin-role middleware.
- **Storage/integrations:** MongoDB, Cloudinary image uploads, PayPal sandbox payments.
- **Main business flows:** registration/login, role-based admin access, product CRUD, image upload, catalog filtering/sorting/search, cart, addresses, checkout, PayPal redirect/capture, orders, reviews, admin order-status management.
- **Deployment shape:** frontend and backend are configured for Vercel; backend exports the Express app and caches the MongoDB connection.

## How to use this file

For each question, first answer from memory in 60–120 seconds. Then compare against the polished answer. The most valuable preparation is to be able to connect every answer back to the exact files and data flow in this project.

---

# Top 10

## 1. Qno: Explain the complete architecture and end-to-end request flow of Quick-Pick.

**Polished answer:**  
Quick-Pick is a client-server e-commerce application with a React/Vite frontend and an Express/Mongoose backend. The frontend uses React Router for navigation and Redux Toolkit for application state. Axios calls the backend API, and Axios is configured to send credentials so the browser can attach the JWT cookie. The backend applies CORS, parses cookies and JSON, ensures MongoDB connectivity, then routes requests into controller functions. Controllers perform business logic and use Mongoose models to access MongoDB. External services are used for Cloudinary image storage and PayPal payments.

A typical request is: user clicks **Add to Cart** → React handler dispatches `addToCart` → Redux `createAsyncThunk` sends `POST /api/shop/cart/add` → Express route runs `authMiddleware` → controller validates the payload and updates the Cart document → JSON response returns → Redux reducer stores the new cart → React re-renders the cart UI.

For checkout, the flow is longer: frontend builds an order payload → backend creates a PayPal payment → backend stores a pending order → frontend redirects to PayPal → PayPal redirects back with payment identifiers → frontend calls capture → backend marks the order paid, decrements stock and deletes the cart.

**TL;DR:** React UI → Redux thunk → Axios → Express middleware → controller → Mongoose → MongoDB/external service → response → Redux → UI.

**Key mappings:**
- `client/src/main.jsx`
- `client/src/App.jsx`
- `client/src/store/store.js`
- `server/server.js`
- `server/routes/**`
- `server/controllers/**`
- `server/models/**`

---

## 2. Qno: How does authentication and authorization work, and what is the difference between them in this project?

**Polished answer:**  
Authentication answers **who the user is**; authorization answers **what that authenticated user is allowed to do**.

During registration, the server checks whether the email already exists, hashes the password with bcrypt, and stores the hash. During login, it loads the user by email, compares the plaintext password with the hash, then signs a JWT containing `id`, `role`, `email`, and `userName`. The JWT is placed in an HTTP-only cookie. The browser therefore does not need to expose the token to application JavaScript.

For protected routes, `authMiddleware` reads `req.cookies.token` and verifies it with the JWT secret. If verification succeeds, it places the decoded claims in `req.user`. `adminMiddleware` then checks `req.user.role === "admin"` for admin-only endpoints.

The frontend has a `CheckAuth` component that performs route-level UX protection, but this is not the security boundary. A malicious client can bypass React entirely and call the API directly. The real security boundary is the backend middleware.

**TL;DR:** bcrypt verifies credentials, JWT represents the session, `authMiddleware` authenticates, `adminMiddleware` authorizes.

**Key mappings:**
- `server/controllers/auth/auth-controller.js`
- `server/routes/auth/auth-routes.js`
- `client/src/store/auth-slice/index.js`
- `client/src/components/common/check-auth.jsx`
- `client/src/App.jsx`

---

## 3. Qno: What are the most important security vulnerabilities in this implementation, and how would you fix them?

**Polished answer:**  
The largest issue is **trusting client-controlled identity and business data**. Many authenticated endpoints accept `userId` from the request body or URL instead of deriving it from `req.user.id`. That can allow an authenticated user to act on another user's cart, addresses or orders by changing the ID.

The payment flow has an even more serious trust problem. The client supplies `cartItems`, prices, `totalAmount`, `userId`, and other order fields. The capture endpoint accepts `paymentId`, `payerId`, and `orderId`, then marks the order as paid without independently verifying the payment with PayPal. A production system must treat the browser as untrusted: derive the user from the verified session, load prices and stock from the database, calculate totals on the server, verify the payment with the provider, and make the capture idempotent.

Other improvements include input validation, rate limiting, secure headers, stricter cookie configuration, file upload limits and MIME validation, escaping search patterns, audit logging, and database constraints/indexes.

**TL;DR:** Never trust client-supplied identity, price, stock, payment status, or authorization. Re-derive sensitive values on the server.

**Key mappings:**
- `server/controllers/auth/auth-controller.js`
- `server/controllers/shop/address-controller.js`
- `server/controllers/shop/cart-controller.js`
- `server/controllers/shop/order-controller.js`
- `server/controllers/shop/product-review-controller.js`
- `server/server.js`

---

## 4. Qno: Explain the checkout and PayPal payment lifecycle, including what can go wrong.

**Polished answer:**  
The checkout page calculates a cart total and sends an order payload to `POST /api/shop/order/create`. The backend constructs a PayPal payment request in sandbox mode, including line items and the total amount, then saves a pending order and returns the PayPal approval URL plus the internal order ID.

The frontend redirects the browser to PayPal. After approval, PayPal redirects to `/shop/paypal-return` with `paymentId` and `PayerID`. The React page reads those query parameters and combines them with `currentOrderId` stored in `sessionStorage`. It then calls `/api/shop/order/capture`.

The current capture implementation changes the order to `paid` and `confirmed`, decrements product stock, deletes the cart, and saves the order. The production concerns are: payment verification, client-controlled totals, duplicate capture, insufficient stock, concurrent checkouts, partial failure between stock changes and order persistence, and the need for webhook/reconciliation handling.

A stronger design is: create an immutable pending order on the server, calculate its total from database prices, create a provider payment for that amount, verify the provider transaction during capture, atomically reserve/decrement stock, make capture idempotent, and use provider webhooks for eventual reconciliation.

**TL;DR:** Checkout is a multi-step distributed workflow; payment verification, idempotency, stock consistency and server-side pricing are mandatory.

**Key mappings:**
- `client/src/pages/shopping-view/checkout.jsx`
- `client/src/pages/shopping-view/paypal-return.jsx`
- `client/src/store/shop/order-slice/index.js`
- `server/controllers/shop/order-controller.js`
- `server/helpers/paypal.js`

---

## 5. Qno: How is cart and inventory consistency handled, and why is the current implementation unsafe under concurrency?

**Polished answer:**  
The Cart document stores product references plus quantities. The UI checks stock before adding or incrementing an item, and the server checks that the referenced product exists. However, the server does not enforce the stock limit when adding or updating cart quantity. The final stock decrement happens during payment capture.

This creates a classic race condition. Suppose stock is 1 and two users both see stock 1. Both can add the item to their carts. Both can reach checkout. Both capture successfully, and both decrement stock. The database can become negative or otherwise inconsistent because the read-check-write sequence is not atomic.

A production design would reserve stock or use an atomic conditional update such as: decrement only when `totalStock >= requestedQuantity`, then inspect the update result. For multiple products, I would use a MongoDB transaction where appropriate. I would also validate stock again at checkout/capture, because client-side checks are only a UX optimization.

**TL;DR:** UI stock checks are not protection. Inventory correctness requires server-side validation plus atomic updates/transactions.

**Key mappings:**
- `client/src/components/shopping-view/cart-items-content.jsx`
- `client/src/pages/shopping-view/checkout.jsx`
- `server/controllers/shop/cart-controller.js`
- `server/controllers/shop/order-controller.js`
- `server/models/Product.js`

---

## 6. Qno: Why are the MongoDB models designed this way, and where would you use embedding versus references?

**Polished answer:**  
The project uses both references and snapshots. A Cart references Products through `productId`, which is appropriate because the cart wants the current product entity. `populate()` is then used to retrieve selected product fields. An Order, on the other hand, stores a snapshot of `title`, `image`, `price`, `quantity`, and other checkout information directly inside `cartItems`.

That snapshot is a good design choice because historical orders should not change just because a product is renamed, repriced or deleted later. The trade-off is duplication and more complex update logic.

The current schema is inconsistent in how identifiers are represented: `Cart.userId` is an ObjectId reference, while `Order.userId`, `Order.cartId`, and `Review.productId/userId` are strings. I would standardize these identifiers where possible and add references where they provide value. I would still preserve immutable order snapshots even if the order also keeps references to users/products.

**TL;DR:** Use references for live relationships; use embedded snapshots for historical or immutable business records.

**Key mappings:**
- `server/models/Cart.js`
- `server/models/Order.js`
- `server/models/Product.js`
- `server/models/Review.js`
- `server/controllers/shop/cart-controller.js`

---

## 7. Qno: Why use Redux Toolkit and `createAsyncThunk` here instead of keeping everything in React component state?

**Polished answer:**  
Local component state is appropriate for transient UI state such as an open dialog, selected address or form inputs. Redux becomes useful for shared asynchronous state that multiple components need. In this project, authentication, products, cart, addresses, orders, reviews, search results and admin data all live in separate slices.

`createAsyncThunk` standardizes the request lifecycle into pending, fulfilled and rejected actions. For example, `fetchCartItems` performs the Axios call, the fulfilled reducer stores `action.payload.data`, and the rejected reducer resets the relevant state.

This separation keeps API logic out of many components and gives multiple parts of the UI access to the same server state. However, the current architecture is fairly manual: errors are mostly reduced to empty arrays/null values, and there is no normalized entity cache. In a larger system I would consider stronger error modeling, request deduplication and possibly RTK Query for server-state caching and invalidation.

**TL;DR:** React state owns local UI state; Redux manages shared asynchronous application state; RTK Query could simplify server-state management further.

**Key mappings:**
- `client/src/store/store.js`
- `client/src/store/auth-slice/index.js`
- `client/src/store/shop/cart-slice/index.js`
- `client/src/store/shop/products-slice/index.js`
- `client/src/store/shop/order-slice/index.js`

---

## 8. Qno: How does route protection work in React, and why can’t frontend route guards be treated as security?

**Polished answer:**  
`App.jsx` renders route groups for authentication, admin, and shopping pages. `CheckAuth` examines the current path, authentication state and user role, then uses React Router's `Navigate` to redirect users.

For example, unauthenticated users are redirected to login, non-admin users trying to access admin paths are sent to `/unauth-page`, and admins are redirected away from shopping routes. The component also handles the root route by sending the user to an appropriate dashboard.

This is valuable for user experience but not sufficient for authorization because the browser is fully under the attacker's control. A user can ignore React and send HTTP requests directly. Therefore, every privileged API route must independently enforce authorization. Quick-Pick does this with `authMiddleware` and `adminMiddleware` on admin endpoints.

**TL;DR:** Frontend guards improve UX; backend authorization protects data and actions.

**Key mappings:**
- `client/src/components/common/check-auth.jsx`
- `client/src/App.jsx`
- `server/controllers/auth/auth-controller.js`
- `server/routes/admin/**`

---

## 9. Qno: Explain the Express middleware chain and the CORS/cookie setup.

**Polished answer:**  
The server creates an Express app, loads environment variables, configures CORS, then registers `cookieParser()` and `express.json()`. A global asynchronous middleware calls `connectDB()` before each request. After that, route groups are mounted under `/api/...`.

CORS is configured with an allowlist derived from `FRONTEND_URL`, enables credentials, and permits the methods/headers used by the application. The frontend sets Axios `withCredentials = true`, which allows the browser to send the authentication cookie across origins when the CORS and cookie policies permit it.

The important interview point is that CORS is not authentication. CORS controls which browser origins can read cross-origin responses; it does not prevent non-browser clients from calling the API. Authorization still comes from server-side authentication middleware.

For cookies, production should use `HttpOnly`, `Secure`, an intentional `SameSite` policy and matching attributes when clearing the cookie. The current project uses `secure: true` and `sameSite: "None"` on login, so the deployment must be HTTPS-aware.

**TL;DR:** Middleware composes cross-cutting concerns; CORS governs browser access, while cookies/JWT govern identity.

**Key mappings:**
- `server/server.js`
- `server/controllers/auth/auth-controller.js`
- `client/src/store/auth-slice/index.js`
- `client/src/store/shop/**`

---

## 10. Qno: Explain the image-upload pipeline and how you would make it production-safe and scalable.

**Polished answer:**  
The admin selects a file in `ProductImageUpload`. The browser creates a `FormData` payload and posts it to `/api/admin/products/upload-image`. The route uses Multer's memory storage, so the file is available as `req.file.buffer`. The controller converts the buffer into a base64 data URL and sends it to Cloudinary through `imageUploadUtil`. Cloudinary returns the hosted URL, which the frontend later stores as the product image URL.

The current design is simple, but memory storage and base64 encoding increase server memory usage. There are also no visible file-size, file-type, dimension or upload-rate restrictions. A production system should validate MIME type and actual content, cap file sizes, use a streaming or direct-to-Cloudinary upload strategy when possible, and enforce authorization before the upload route.

For larger scale, I would prefer browser-to-Cloudinary direct upload using signed parameters or an upload preset, so the application server does not become a file transport bottleneck.

**TL;DR:** Current flow is browser → Multer memory buffer → base64 → Cloudinary; production systems should minimize server-side file handling and enforce strict upload validation.

**Key mappings:**
- `client/src/components/admin-view/image-upload.jsx`
- `server/routes/admin/products-routes.js`
- `server/controllers/admin/products-controller.js`
- `server/helpers/cloudinary.js`

---

# 11–25

## 11. Qno: Why is accepting price and total amount from the client dangerous in `createOrder`, and what should the server calculate?

**Polished answer:**  
The current checkout sends `cartItems` and `totalAmount` from React, and the backend uses those values to build the PayPal request. A browser user can modify the request before it reaches the server, so trusting those fields permits price manipulation.

The server should accept only stable identifiers and user intent, such as product IDs, quantities, address ID and payment method. It should then load current product prices from MongoDB, determine whether the sale price applies, validate quantities and stock, calculate the subtotal and final total, and construct the payment request from those server-owned values.

The order should save the exact price actually charged as an immutable snapshot. The payment provider amount should equal the server-calculated total, not a number copied from the browser.

**TL;DR:** The client can request an order; only the server should decide what it costs.

**Key mappings:**
- `client/src/pages/shopping-view/checkout.jsx`
- `server/controllers/shop/order-controller.js`
- `server/models/Product.js`
- `server/models/Order.js`

---

## 12. Qno: What is wrong with the current `capturePayment` implementation, and how would you make payment confirmation trustworthy?

**Polished answer:**  
The endpoint receives `paymentId`, `payerId` and `orderId`, loads the local order, and immediately marks it `paid` and `confirmed`. It does not show an independent PayPal API verification that the payment exists, belongs to the expected order, and matches the expected amount and currency.

A secure implementation would retrieve the payment/capture status from PayPal using server-side credentials and compare provider data with the pending order. It should verify the amount, currency, order identity and current payment state before changing local state.

The endpoint should also be idempotent. If the same capture request arrives twice, it should return the existing result rather than decrementing stock twice. A stronger production architecture would additionally consume PayPal webhooks and reconcile local orders when browser redirects are interrupted.

**TL;DR:** A payment callback is untrusted input until the backend verifies the provider transaction.

**Key mappings:**
- `server/controllers/shop/order-controller.js`
- `server/helpers/paypal.js`
- `client/src/pages/shopping-view/paypal-return.jsx`

---

## 13. Qno: How would you make order creation, stock decrement and cart deletion atomic?

**Polished answer:**  
The current capture flow updates multiple documents independently: each product's stock is decremented, the cart is deleted, and the order is saved. If one operation fails after earlier operations succeed, the system can become inconsistent.

For MongoDB, I would use a transaction for the local database state when the deployment's MongoDB topology supports transactions. Inside the transaction, I would verify the order is still pending, atomically decrement stock with a condition, update the order to confirmed/paid, and delete the cart. If any required operation fails, the transaction aborts and all local changes roll back.

Payment-provider calls should generally happen outside the database transaction because external systems are not part of the MongoDB transaction. That is why the overall design becomes a distributed workflow: verify the external payment, then perform the local state transition atomically and idempotently.

**TL;DR:** Use atomic conditional updates/transactions for local consistency and idempotency for external payment workflows.

**Key mappings:**
- `server/controllers/shop/order-controller.js`
- `server/models/Order.js`
- `server/models/Cart.js`
- `server/models/Product.js`

---

## 14. Qno: Why does an order store product details again instead of only storing `productId`?

**Polished answer:**  
An order is a historical business record. If a product currently costs 40 but was purchased for 25 during a sale, an old order must continue to show 25 even after the product changes. Likewise, a product might be deleted, renamed or have a different image later.

Therefore the order stores a snapshot of title, image, price and quantity. This is denormalization, but it protects historical correctness. I would usually keep a product reference as well if future navigation/reporting benefits from it, while treating the snapshot price and item details as the authoritative historical record.

**TL;DR:** Orders need immutable historical values; live product data can change after purchase.

**Key mappings:**
- `server/models/Order.js`
- `server/models/Product.js`
- `server/controllers/shop/order-controller.js`
- `client/src/components/shopping-view/order-details.jsx`

---

## 15. Qno: What does Mongoose `populate()` do here, and what problems can arise when referenced products are deleted?

**Polished answer:**  
`populate()` replaces a referenced ObjectId with the corresponding MongoDB document, subject to the selected fields. The cart fetch uses `populate()` to load only `image`, `title`, `price` and `salePrice`, then transforms those documents into a frontend-friendly response.

This is convenient, but references can become dangling when the target product is deleted. The cart fetch explicitly filters out items whose populated product is missing. However, the delete-cart endpoint assumes `item.productId._id` exists after population, so a dangling reference can trigger an exception there.

I would either enforce lifecycle rules such as preventing deletion of referenced products, clean up dependent cart entries when deleting a product, or make all cart operations null-safe.

**TL;DR:** `populate()` is application-level join behavior; you still need a strategy for broken references.

**Key mappings:**
- `server/models/Cart.js`
- `server/models/Product.js`
- `server/controllers/shop/cart-controller.js`

---

## 16. Qno: Explain how product filtering and sorting work from the UI to MongoDB.

**Polished answer:**  
The listing page stores filters as arrays such as `category: ["men", "women"]` and `brand: ["nike"]`. It serializes them into query parameters like `category=men,women&brand=nike`. The Redux thunk sends those parameters to `/api/shop/products/get`.

On the backend, the controller splits comma-separated values and converts them to `$in` filters. Sorting is mapped from known option names to MongoDB sort objects: price ascending/descending or title ascending/descending. Then `Product.find(filters).sort(sort)` executes the database query.

The important design principle is that the client expresses the desired filter, while the backend owns the database query. In production I would whitelist every filter field, validate allowed values, add pagination and appropriate indexes, and reject unknown sort fields rather than allowing arbitrary database expressions.

**TL;DR:** UI filters become query parameters; the API translates them into safe MongoDB predicates and sort definitions.

**Key mappings:**
- `client/src/pages/shopping-view/listing.jsx`
- `client/src/store/shop/products-slice/index.js`
- `server/controllers/shop/products-controller.js`
- `server/routes/shop/products-routes.js`

---

## 17. Qno: What are the performance and security concerns with the current product search implementation?

**Polished answer:**  
The backend constructs `new RegExp(keyword, "i")` and searches title, description, category and brand with a MongoDB OR predicate. This is flexible but has two main concerns.

First, the pattern is built directly from user input, so regex metacharacters are interpreted as regex syntax. A user can therefore produce unexpectedly expensive patterns. Second, arbitrary regex matching across multiple fields can become slow as the product collection grows, especially without suitable indexes.

For a small catalog this may be acceptable. At scale, I would escape user input if literal substring matching is intended, impose length limits, add indexes appropriate to the search pattern, or move search to MongoDB Atlas Search/OpenSearch/Elasticsearch depending on requirements.

**TL;DR:** User-supplied regex over several fields is convenient for a demo but needs escaping, limits and a search strategy at scale.

**Key mappings:**
- `server/controllers/shop/search-controller.js`
- `server/routes/shop/search-routes.js`
- `client/src/store/shop/search-slice/index.js`
- `client/src/pages/shopping-view/search.jsx`

---

## 18. Qno: Walk through the major REST endpoints and explain how you would organize them.

**Polished answer:**  
The API is grouped by domain:

- `/api/auth`: register, login, logout, check-auth.
- `/api/admin/products`: upload-image, add, edit, delete, get.
- `/api/admin/orders`: list, details, update-status.
- `/api/shop/products`: filter/sort products and product details.
- `/api/shop/cart`: add, fetch, update quantity, delete item.
- `/api/shop/address`: add, list, update, delete.
- `/api/shop/order`: create, capture, list by user, details.
- `/api/shop/search`: search by keyword.
- `/api/shop/review`: add and list reviews.
- `/api/common/feature`: public feature-image retrieval and admin-protected creation.

This separation is readable and follows business domains. A production redesign could make resources more REST-consistent and derive the authenticated user from the session rather than exposing `userId` in many paths. I would also centralize validation and error formatting so every endpoint follows consistent API semantics.

**TL;DR:** Domain-based route organization is good; ownership and validation should be enforced centrally and consistently.

**Key mappings:**
- `server/routes/**`
- `server/controllers/**`
- `client/src/store/**`

---

## 19. Qno: How good is the error handling, and what would you improve?

**Polished answer:**  
The controllers generally use `try/catch` and return JSON containing `success` and `message`. The API uses some meaningful status codes such as 400 for invalid input, 401 for unauthenticated requests, 403 for forbidden access and 404 for missing records.

However, the handling is inconsistent. Some failures return plain `res.json()` without an explicit status. Several client reducers simply turn an error into an empty list or null, which hides the actual reason from the UI. There is also at least one backend bug where a catch block refers to `error` even though the caught variable is named `e`, which can mask the original error.

A stronger implementation would use a centralized error middleware, typed/domain-specific error classes, consistent response shapes, structured logging with request IDs, and explicit client-side error state.

**TL;DR:** There is basic error handling, but production code needs centralized, consistent and observable failure handling.

**Key mappings:**
- `server/controllers/**`
- `server/server.js`
- `client/src/store/**`
- `server/controllers/shop/products-controller.js`

---

## 20. Qno: Where should input validation happen, and what validation is missing here?

**Polished answer:**  
Validation should happen at the server boundary because the server cannot trust a browser. The client should also validate for UX, but server validation is authoritative.

The current code checks some required fields manually, such as addresses and cart quantities, but there is no comprehensive schema validation for user registration, product creation, order payloads, review ratings, IDs, prices, stock, or search keywords. There are also no explicit constraints such as rating range 1–5, sale price not exceeding regular price, stock being a non-negative integer, or safe state transitions for order status.

I would use a schema validator such as Zod, Joi or express-validator, validate and transform request data centrally, and reject unexpected fields where appropriate.

**TL;DR:** Client validation improves UX; server schema validation protects the system.

**Key mappings:**
- `server/controllers/auth/auth-controller.js`
- `server/controllers/admin/products-controller.js`
- `server/controllers/shop/order-controller.js`
- `server/controllers/shop/product-review-controller.js`
- `server/controllers/shop/address-controller.js`

---

## 21. Qno: Why use bcrypt for passwords instead of encryption or plain hashing?

**Polished answer:**  
Passwords should not be reversible, so encryption is the wrong abstraction for password storage. They also should not be stored with a fast general-purpose hash such as plain SHA-256 because attackers can test guesses extremely quickly.

The project uses `bcrypt.hash(password, 12)` during registration and `bcrypt.compare()` during login. bcrypt is deliberately slow and uses a work factor, making offline password guessing more expensive. The server stores only the bcrypt hash, not the original password.

In production I would also enforce password policy, protect login endpoints with rate limiting or progressive throttling, avoid revealing whether an email exists when appropriate, and consider modern password hashing options such as Argon2 where supported.

**TL;DR:** Passwords need salted, adaptive, intentionally expensive password hashing, not reversible encryption or fast hashes.

**Key mappings:**
- `server/controllers/auth/auth-controller.js`
- `server/models/User.js`
- `server/package.json`

---

## 22. Qno: Explain the JWT lifecycle in this application and discuss session expiry.

**Polished answer:**  
After login, the server signs a JWT with the user ID, role, email and username. The token expires after 60 minutes and is stored in an HTTP-only cookie. Each protected request sends the cookie automatically, and `authMiddleware` verifies the signature and expiry before populating `req.user`.

On page load, the frontend dispatches `checkAuth()`, which calls `/api/auth/check-auth`. A valid token restores the frontend authentication state without requiring the user to log in again during the token lifetime.

The trade-off is that role data in the JWT is a snapshot. If a user's role changes in MongoDB, an already-issued token still contains the old role until it expires. For higher-security systems, I would consider short-lived access tokens plus refresh-token rotation, revocation strategy, or server-side session storage depending on the threat model.

**TL;DR:** JWT makes the session stateless, but claims become stale until token expiry unless you add revocation or revalidation.

**Key mappings:**
- `server/controllers/auth/auth-controller.js`
- `client/src/App.jsx`
- `client/src/store/auth-slice/index.js`

---

## 23. Qno: What is the relationship between `withCredentials`, CORS, `SameSite`, and `Secure` cookies?

**Polished answer:**  
For a cross-origin browser request, JavaScript must use credentials mode, which Axios enables through `withCredentials`. The server must also return a CORS response that allows that specific origin and credentials. Wildcard `Access-Control-Allow-Origin: *` cannot be combined with credentialed browser access.

The cookie itself has browser policy constraints. `HttpOnly` prevents client JavaScript from reading it. `Secure` requires HTTPS in normal production use. `SameSite` controls cross-site sending behavior; the project uses `SameSite=None`, which is commonly needed for cross-site cookies and therefore requires `Secure`.

A correct deployment needs the frontend origin, backend CORS allowlist, cookie settings, HTTPS and reverse-proxy behavior to agree. A common interview point is that “CORS error” and “authentication failure” are separate layers even though they often appear together during debugging.

**TL;DR:** Credentialed cookies require coordinated browser, CORS and cookie policy configuration.

**Key mappings:**
- `client/src/store/auth-slice/index.js`
- `server/server.js`
- `server/controllers/auth/auth-controller.js`
- `server/.env.example`

---

## 24. Qno: Why does the backend cache the MongoDB connection, and why is that useful on Vercel/serverless?

**Polished answer:**  
The backend is configured for Vercel, where requests can execute inside short-lived or reused serverless instances. Opening a new MongoDB connection for every request is expensive and can exhaust connection limits.

The `connectDB()` function keeps a module-level `cachedConnection` promise. If the Mongoose connection is already ready, the same connection is reused; otherwise a connection attempt is created and awaited.

This is a common serverless optimization because module state can survive across warm invocations. A production implementation should also handle rejected connection promises carefully so a failed cached promise does not poison future requests, and should tune MongoDB connection pool sizes for the serverless concurrency model.

**TL;DR:** Cache the DB connection across warm invocations to reduce connection overhead and protect MongoDB from connection storms.

**Key mappings:**
- `server/server.js`
- `server/vercel.json`
- `server/.env.example`

---

## 25. Qno: Explain the deployment architecture and environment-variable flow.

**Polished answer:**  
The frontend is built with Vite and deployed through a Vercel rewrite that sends unknown client-side paths to `/`, which supports React Router history fallback. The backend uses a Vercel Node build and routes all requests to `server.js`.

The frontend receives `VITE_BACKENDURL` at build time because Vite exposes variables prefixed with `VITE_` to browser code. The server separately reads secrets such as MongoDB credentials, JWT secret, Cloudinary credentials and PayPal credentials from `process.env`.

The important security distinction is that `VITE_*` variables are not secrets because they become part of the client bundle. Secrets must stay server-side. The backend also uses `FRONTEND_URL` for CORS and PayPal redirect URLs, so deployment configuration has to keep the two applications consistent.

**TL;DR:** Public frontend config can be embedded in the browser; secrets must remain in backend environment variables.

**Key mappings:**
- `client/.env.example`
- `client/vercel.json`
- `client/vite.config.js`
- `server/.env.example`
- `server/vercel.json`

---

# 26–50

## 26. Qno: How is React component architecture organized, and why are layouts plus `Outlet` useful here?

**Polished answer:**  
The application separates shared layouts from individual pages. `AuthLayout`, `AdminLayout` and `ShoppingLayout` render persistent structure and an `<Outlet />` where nested routes appear.

This avoids duplicating headers, sidebars and wrappers across every page. For example, the shopping layout owns `ShoppingHeader`, while the nested routes render home, listing, checkout, account, search and payment pages. The admin layout similarly owns the sidebar and header.

This is a good separation of page-level composition from route-specific content. It also makes route protection easier because a wrapper such as `CheckAuth` can protect an entire subtree.

**TL;DR:** Layout routes centralize persistent UI and let child routes focus on page-specific behavior.

**Key mappings:**
- `client/src/App.jsx`
- `client/src/components/auth/layout.jsx`
- `client/src/components/admin-view/layout.jsx`
- `client/src/components/shopping-view/layout.jsx`

---

## 27. Qno: Why is `CommonForm` plus configuration arrays a useful abstraction?

**Polished answer:**  
The project defines form metadata such as label, field name, type and component type in configuration arrays like `loginFormControls`, `registerFormControls`, `addProductFormElements` and `addressFormControls`.

`CommonForm` maps over those definitions and dynamically renders inputs, selects and textareas. This reduces duplicated JSX and makes it easy to add or remove fields by changing configuration rather than rewriting the component structure.

The trade-off is that highly dynamic forms can become harder to type, validate and customize. A production version should pair this pattern with a schema definition so the UI metadata and validation rules do not drift apart. It should also fix small implementation issues such as using an undefined `id` for the textarea field.

**TL;DR:** Config-driven forms reduce repetition, but validation and field semantics should be centralized too.

**Key mappings:**
- `client/src/components/common/form.jsx`
- `client/src/config/index.js`
- `client/src/pages/auth/login.jsx`
- `client/src/pages/auth/register.jsx`

---

## 28. Qno: How do the `useEffect` calls drive data fetching, and what dependency mistakes should you look for?

**Polished answer:**  
The project uses effects to trigger initialization and react to state changes: `App` calls `checkAuth`, listing fetches products when filters/sort change, account pages fetch addresses/orders, and product details trigger review loading.

The interviewer should look for whether every value used by an effect is represented in the dependency array, whether the effect can fire repeatedly, and whether it needs cancellation. There are examples where dependencies are incomplete, such as address fetching depending on `user` but only listing `[dispatch]`.

The correct mental model is that an effect synchronizes React with something outside the render calculation. It should be deterministic with respect to its dependencies, and asynchronous work should handle stale results or component unmounting when necessary.

**TL;DR:** `useEffect` is synchronization, not a general-purpose “run code after render” hook; dependencies must match referenced reactive values.

**Key mappings:**
- `client/src/App.jsx`
- `client/src/components/shopping-view/address.jsx`
- `client/src/components/shopping-view/orders.jsx`
- `client/src/components/shopping-view/product-details.jsx`
- `client/src/pages/shopping-view/listing.jsx`

---

## 29. Qno: Is the search input debounced correctly? Explain the current behavior and the fix.

**Polished answer:**  
The search page uses `setTimeout(..., 1000)` inside `useEffect`, but it never clears the previous timeout. That means every qualifying keystroke schedules a request one second later. Typing quickly can therefore produce multiple stale API calls instead of one debounced request.

A real debounce keeps a timer handle and calls `clearTimeout()` in the effect cleanup before scheduling the next one. Even better, the API layer can cancel stale requests with `AbortController` or Axios cancellation when appropriate.

This matters both for performance and correctness: an older response can arrive after a newer response and overwrite current search results.

**TL;DR:** The code delays requests but does not debounce them; cleanup is required to cancel the previous timer/request.

**Key mappings:**
- `client/src/pages/shopping-view/search.jsx`
- `client/src/store/shop/search-slice/index.js`

---

## 30. Qno: Why is `sessionStorage` used for filters and the current order ID, and what are the trade-offs?

**Polished answer:**  
The listing page stores filter state in `sessionStorage` so navigation between category views and listing routes can preserve filters. The payment flow stores the current order ID so the PayPal return page can associate the provider callback with the locally created order.

`sessionStorage` is scoped to the browser tab and survives page refreshes during that tab's lifetime. That is useful for temporary UI continuity, but it is not trusted storage. A malicious user can edit it, so it must never be treated as proof of authorization or payment status.

For the order ID specifically, a stronger design would have the payment provider callback contain a server-verifiable correlation identifier, and the backend would still authenticate the user and verify the provider transaction before finalizing the order.

**TL;DR:** `sessionStorage` is good for temporary UX state, not for security decisions.

**Key mappings:**
- `client/src/pages/shopping-view/listing.jsx`
- `client/src/pages/shopping-view/checkout.jsx`
- `client/src/pages/shopping-view/paypal-return.jsx`
- `client/src/components/shopping-view/header.jsx`

---

## 31. Qno: Is the cart UI doing optimistic or pessimistic updates, and what would you choose?

**Polished answer:**  
The current cart uses a server-first, effectively pessimistic model. When a user adds, updates or deletes an item, the client dispatches the API request and only then updates Redux with the returned server data.

That is safer for inventory-sensitive state because the UI does not permanently assume the operation succeeded. The downside is visible latency. A more advanced UI could optimistically update local state and then reconcile with the server, but that requires rollback logic when the server rejects the operation because of stock or authentication issues.

For this domain I would keep server authority and potentially use optimistic rendering for non-critical presentation while still reconciling quickly from the server response.

**TL;DR:** Server-confirmed cart state is safer; optimistic UX is possible but needs rollback and reconciliation.

**Key mappings:**
- `client/src/store/shop/cart-slice/index.js`
- `client/src/components/shopping-view/cart-items-content.jsx`
- `server/controllers/shop/cart-controller.js`

---

## 32. Qno: Why are client-side stock checks still useful even though they are not security controls?

**Polished answer:**  
The listing, product details and cart components check stock before allowing the user to increase quantity. This prevents an obvious bad UX such as clicking “plus” when the displayed inventory is already insufficient.

However, the client may be stale, modified, or racing another user. Therefore the server must repeat the stock validation and enforce it atomically. The client-side check is a latency optimization and user-experience feature; the backend is the source of truth.

The ideal architecture is therefore duplicate validation at different layers with different responsibilities: the UI prevents predictable bad interactions, while the API protects the business invariant.

**TL;DR:** Client validation improves UX; backend validation protects correctness.

**Key mappings:**
- `client/src/pages/shopping-view/listing.jsx`
- `client/src/components/shopping-view/product-details.jsx`
- `client/src/components/shopping-view/cart-items-content.jsx`
- `server/controllers/shop/cart-controller.js`
- `server/controllers/shop/order-controller.js`

---

## 33. Qno: How should money be represented and calculated in an e-commerce backend?

**Polished answer:**  
The current project stores product prices as JavaScript/Mongoose numbers and calculates totals with ordinary arithmetic. That is easy for a prototype, but floating-point arithmetic can create precision problems for currency.

A stronger design stores money as integer minor units, such as cents, or uses a decimal type. All calculations should happen server-side from authoritative database values. The provider amount should use the exact server-calculated total, and the order should snapshot the charged unit price.

The current `Order` schema also stores `cartItems.price` as a string while `Product.price` is a number, which is a type inconsistency worth fixing.

**TL;DR:** Treat money as a precise domain value, not an uncontrolled JavaScript float.

**Key mappings:**
- `server/models/Product.js`
- `server/models/Order.js`
- `server/controllers/shop/order-controller.js`
- `client/src/pages/shopping-view/checkout.jsx`

---

## 34. Qno: Explain the review system and discuss whether the average-review calculation scales well.

**Polished answer:**  
A review stores the product ID, user ID, username, text and numeric rating. Before insertion, the backend checks whether the user has an order containing the product and rejects a duplicate review for the same user/product pair. After saving, it loads all reviews for that product, calculates the average, and writes the result back to the Product document.

The business idea is sound, but the implementation has correctness and scalability gaps. The purchase check is not restricted to a delivered/confirmed order, and identity comes from the client request rather than `req.user`. The duplicate check is application-level and can race without a database uniqueness constraint.

The average calculation is O(number of reviews) on every new review. At scale, I would use aggregation, maintain review count plus rating sum, or use a transactional/atomic strategy depending on consistency requirements.

**TL;DR:** Review eligibility should be server-owned and the aggregate should be maintained efficiently with database guarantees.

**Key mappings:**
- `server/controllers/shop/product-review-controller.js`
- `server/models/Review.js`
- `server/models/Product.js`
- `client/src/components/shopping-view/product-details.jsx`

---

## 35. Qno: Walk through the admin product-management workflow.

**Polished answer:**  
Admin product management starts in the React admin page. A form collects product metadata, while `ProductImageUpload` sends the selected file to the protected image-upload endpoint. The returned Cloudinary URL is included when creating a product.

Redux dispatches `addNewProduct`, `fetchAllProducts`, `editProduct` and `deleteProduct`. The backend routes all write operations through both authentication and admin middleware. Product controllers then create, update or delete the MongoDB document.

The important interview distinction is that UI state such as `currentEditedId`, open dialogs and form values is local, while the product collection is server state held in Redux. Production improvements include validation, audit logs, soft deletion where appropriate, optimistic concurrency for edits and stronger file constraints.

**TL;DR:** Admin UI manages local form state; Redux synchronizes the authoritative product collection with protected API endpoints.

**Key mappings:**
- `client/src/pages/admin-view/products.jsx`
- `client/src/components/admin-view/product-tile.jsx`
- `client/src/components/admin-view/image-upload.jsx`
- `client/src/store/admin/products-slice/index.js`
- `server/controllers/admin/products-controller.js`

---

## 36. Qno: Model the order-status flow as a state machine. Why is that better than arbitrary strings?

**Polished answer:**  
The UI exposes statuses such as `pending`, `inProcess`, `inShipping`, `delivered` and `rejected`, while the payment flow also uses `confirmed`. The backend currently stores `orderStatus` as an unrestricted string and accepts whatever status the admin sends.

A state machine would explicitly define legal transitions. For example, `pending -> confirmed`, `confirmed -> inProcess`, `inProcess -> inShipping`, `inShipping -> delivered`, and specific cancellation/rejection paths. It would reject invalid transitions such as `delivered -> pending` unless an explicit business operation allows them.

This improves data quality, auditability and reasoning. It also lets different roles have different transition permissions.

**TL;DR:** Business state should be modeled as a constrained transition graph, not just a free-form string.

**Key mappings:**
- `server/models/Order.js`
- `server/controllers/admin/order-controller.js`
- `client/src/components/admin-view/order-details.jsx`
- `client/src/components/admin-view/orders.jsx`

---

## 37. Qno: Identify the authorization problem in the address APIs and explain the correct ownership check.

**Polished answer:**  
The address routes use `authMiddleware`, but the target `userId` comes from the request body or URL. The controller then queries addresses using that supplied ID. Authentication proves that the caller is logged in, but it does not prove that the caller owns the supplied user ID.

The correct pattern is to ignore a client-supplied user ID for authorization and derive the identity from the verified session: `const userId = req.user.id`. Then an address query becomes something like “find address by `addressId` and authenticated user ID”.

This same rule should be applied consistently across cart and order APIs.

**TL;DR:** Authentication without resource ownership checks still permits horizontal privilege escalation.

**Key mappings:**
- `server/routes/shop/address-routes.js`
- `server/controllers/shop/address-controller.js`
- `server/controllers/auth/auth-controller.js`
- `client/src/components/shopping-view/address.jsx`

---

## 38. Qno: Identify the authorization problem in the cart APIs and explain how you would fix it.

**Polished answer:**  
The cart endpoints accept `userId` from the request body or URL and then query the cart using that value. Since the only server-side check is “is the requester authenticated?”, an attacker who knows another user ID could potentially target that cart.

The fix is to derive the owner from `req.user.id` and remove the ownership-critical `userId` parameter from the client API entirely. For example, `POST /api/shop/cart/items` could accept only `productId` and `quantity`, while the controller internally uses the authenticated user ID.

This is a classic broken-object-level-authorization problem. Resource IDs and ownership claims supplied by clients must never be trusted as authorization evidence.

**TL;DR:** Do not let a request choose which user's cart it is modifying.

**Key mappings:**
- `server/controllers/shop/cart-controller.js`
- `server/routes/shop/cart-routes.js`
- `server/controllers/auth/auth-controller.js`

---

## 39. Qno: What authorization checks are missing from order listing and order-detail endpoints?

**Polished answer:**  
The user order routes accept a `userId` path parameter for listing and an order ID for details. Authentication is enforced, but the controller does not show a comparison between the requested user/order ownership and the authenticated user.

A user could therefore attempt to query another user's orders by changing the identifier. The order-detail endpoint is even more direct because it simply uses `findById(id)`.

The secure pattern is `Order.findOne({ _id: orderId, userId: req.user.id })` for user-owned data. Admin endpoints can intentionally bypass that ownership restriction because admin authorization is a different policy.

**TL;DR:** Every read of private resources needs both authentication and ownership authorization.

**Key mappings:**
- `server/routes/shop/order-routes.js`
- `server/controllers/shop/order-controller.js`
- `server/routes/admin/order-routes.js`

---

## 40. Qno: Why is the review endpoint vulnerable to identity spoofing, and how should user identity be modeled?

**Polished answer:**  
The frontend sends `userId` and `userName` when adding a review. The backend trusts those values, uses the supplied `userId` to determine purchase eligibility, and stores the supplied username.

An attacker could therefore submit another user's ID and potentially pass the purchase check if that user owns the product. Even if the attacker cannot exactly impersonate the user, the design is fundamentally wrong because identity must come from authentication, not the request body.

The backend should use `req.user.id` as the reviewer identity and load the display name from the authenticated user record or trusted claims. The request should contain only review-specific data such as product ID, rating and message.

**TL;DR:** Identity is an authentication output, not a client-provided form field.

**Key mappings:**
- `server/controllers/shop/product-review-controller.js`
- `server/routes/shop/review-routes.js`
- `client/src/components/shopping-view/product-details.jsx`

---

## 41. Qno: What validations should exist for product image uploads?

**Polished answer:**  
The current flow accepts one uploaded file and keeps it in memory. There is no visible server-side restriction on file size, extension, MIME type, dimensions or the number of uploads a user can issue.

A secure upload pipeline should validate the actual content type rather than trusting the filename, enforce a small maximum size, reject dangerous or unsupported formats, optionally resize images, and rate-limit the upload endpoint. It should also avoid holding arbitrarily large files in memory.

If using Cloudinary, direct signed uploads from the browser can shift bandwidth away from the application server while the backend still controls who is allowed to request an upload signature.

**TL;DR:** File uploads are an untrusted input surface and need stricter validation than ordinary JSON.

**Key mappings:**
- `client/src/components/admin-view/image-upload.jsx`
- `server/helpers/cloudinary.js`
- `server/routes/admin/products-routes.js`
- `server/controllers/admin/products-controller.js`

---

## 42. Qno: What production security middleware is missing from this Express application?

**Polished answer:**  
The server configures CORS, cookies and JSON parsing, but there is no visible rate limiter, security-header middleware such as Helmet, centralized request-size policy beyond Express defaults, request ID/correlation middleware, structured logging, or explicit abuse protection.

For authentication endpoints I would add rate limiting and possibly account lockout/progressive delays. For search and upload endpoints I would add tighter request limits. For all traffic I would add security headers, structured logs, monitoring and alerting.

I would also make error responses less revealing and ensure secrets never appear in logs. Security middleware is layered defense; it complements, rather than replaces, correct authorization and input validation.

**TL;DR:** CORS and JWT are not a complete security posture; add abuse prevention, headers, observability and boundary controls.

**Key mappings:**
- `server/server.js`
- `server/package.json`
- `server/controllers/auth/auth-controller.js`

---

## 43. Qno: What makes an API “RESTful” here, and what would you improve in the endpoint design?

**Polished answer:**  
The application already uses resource-oriented routes and HTTP verbs: `GET` for reads, `POST` for creation, `PUT` for updates and `DELETE` for deletion. Domain prefixes like `/api/shop/cart` and `/api/admin/products` provide useful grouping.

However, several endpoints are action-oriented, such as `/add`, `/get`, `/capture` and `/update/:id`. That is common in application APIs and not inherently wrong, but a more resource-oriented design could use patterns such as `POST /cart/items`, `PATCH /cart/items/:productId`, `POST /orders/:id/payment/capture` or a separate payment resource depending on the domain.

I would prioritize consistency over purity: stable resource naming, predictable status codes, validation errors, pagination conventions and idempotency behavior matter more than matching a textbook REST checklist.

**TL;DR:** The API is reasonably resource-grouped; the next step is consistency, ownership semantics and well-defined resource state transitions.

**Key mappings:**
- `server/routes/**`
- `server/controllers/**`
- `client/src/store/**`

---

## 44. Qno: What is weak about the current Redux error/loading state model?

**Polished answer:**  
Most slices have a simple `isLoading` boolean and then either useful data or a fallback such as an empty array/null. This is easy to understand but loses important information: whether a request failed, why it failed, whether another request is still in flight, and whether stale data should remain visible.

For example, a failed order-list request becomes `orderList = []`, which is visually ambiguous between “the user has no orders” and “the API failed.” A stronger state model would contain `status`, `data`, `error`, and potentially request IDs or timestamps.

For server state, RTK Query could reduce the manual lifecycle boilerplate and provide caching, invalidation, deduplication and stale-state management.

**TL;DR:** Empty data is not the same thing as failed data; model loading, success, failure and stale state explicitly.

**Key mappings:**
- `client/src/store/auth-slice/index.js`
- `client/src/store/shop/order-slice/index.js`
- `client/src/store/shop/cart-slice/index.js`
- `client/src/store/admin/order-slice/index.js`

---

## 45. Qno: How would you prevent duplicate order creation and duplicate payment capture?

**Polished answer:**  
A user can double-click, refresh, retry after a network timeout, or receive repeated callbacks. Therefore a payment flow must tolerate retries.

I would assign an idempotency key to the checkout attempt and persist it with the pending order. Creating an order with the same key would return the existing pending order rather than creating a second one. During capture, the server would first check the current order state: if already paid, return success without changing stock; if still pending, perform the verified transition.

The stock update should also be conditional or transactional so a retry cannot decrement stock twice.

**TL;DR:** Payment endpoints must be designed for retries, not just successful one-time execution.

**Key mappings:**
- `server/controllers/shop/order-controller.js`
- `server/models/Order.js`
- `client/src/store/shop/order-slice/index.js`
- `client/src/pages/shopping-view/paypal-return.jsx`

---

## 46. Qno: How would you scale product, order and admin list endpoints when the data grows?

**Polished answer:**  
The current endpoints generally call `find({})` and return the complete collection. That is fine for a small project but does not scale because response size, database work and frontend rendering grow linearly with total records.

I would add pagination using `limit` plus a stable cursor or page strategy, return only required fields, and add indexes matching common filters and sorts. Orders could be queried by authenticated user plus date/status. Products could use indexes for category/brand and possibly price/title depending on query patterns.

For very large datasets, cursor pagination is typically more stable than large `skip` offsets. Admin dashboards may also need server-side aggregation rather than loading every row into the browser.

**TL;DR:** Never assume `find({})` will remain cheap; pagination, projection and indexes are part of API design.

**Key mappings:**
- `server/controllers/admin/order-controller.js`
- `server/controllers/admin/products-controller.js`
- `server/controllers/shop/products-controller.js`
- `server/controllers/shop/order-controller.js`

---

## 47. Qno: Which database indexes and constraints would you add?

**Polished answer:**  
The User model already marks `email` and `userName` as unique, which is important for identity. For the rest of the system, I would index the fields used in frequent queries and ownership checks.

Examples include `Cart.userId`, `Address.userId`, `Order.userId`, `Review.productId`, and possibly compound indexes such as review uniqueness on `(productId, userId)`. Product filtering may benefit from indexes on category and brand, while order history may benefit from `(userId, orderDate)`.

Indexes should be chosen from real query patterns because every index increases write cost and storage. Constraints should enforce business invariants where possible instead of relying exclusively on application code.

**TL;DR:** Index around real access patterns and use database constraints for invariants that must never be violated.

**Key mappings:**
- `server/models/User.js`
- `server/models/Cart.js`
- `server/models/Address.js`
- `server/models/Order.js`
- `server/models/Review.js`
- `server/controllers/**`

---

## 48. Qno: How would you test this project from unit level to end-to-end level?

**Polished answer:**  
I would divide testing by responsibility.

At the unit level, test pure logic such as price calculation, filter serialization, state-transition rules and validation. At the controller/service level, test authorization, error responses, stock checks, review eligibility and order lifecycle using mocked external services. For integration tests, run against a test MongoDB and verify real Mongoose behavior. For API tests, verify the full authentication/authorization flow with cookies.

For end-to-end tests, cover registration, login, catalog search/filter, cart operations, checkout initiation, payment callback handling and admin order updates. Payment-provider behavior should be mocked or run through a sandbox rather than charged live.

Security tests should explicitly attempt cross-user access by replacing path/body IDs with another user's identifiers.

**TL;DR:** Test business rules, persistence, API boundaries, user flows and security assumptions separately and together.

**Key mappings:**
- `server/controllers/**`
- `server/routes/**`
- `client/src/store/**`
- `client/src/pages/**`
- `server/models/**`

---

## 49. Qno: If Quick-Pick grew from a small project to a high-traffic e-commerce platform, what would you change first?

**Polished answer:**  
I would first protect correctness and reliability, then scale individual bottlenecks.

For the backend: introduce service-layer boundaries, centralized validation/errors, strong ownership checks, transactional inventory updates, verified/idempotent payments, pagination and indexes. For state: consider RTK Query or another server-state strategy. For search: move beyond broad regex scans. For files: use direct cloud uploads. For observability: add structured logs, traces, metrics and alerting.

At larger scale, asynchronous queues become useful for email, image processing, analytics and other non-critical work. Caching can help read-heavy catalog traffic. Payment reconciliation and inventory reservation deserve explicit domain services. A microservice split would only come after clear scaling or ownership boundaries justify it; I would not introduce microservices merely because the system is “large.”

**TL;DR:** Fix correctness and security first, then scale data access, search, files, background work and observability based on measured bottlenecks.

**Key mappings:**
- `server/server.js`
- `server/controllers/**`
- `server/models/**`
- `client/src/store/**`
- `server/helpers/**`

---

## 50. Qno: If the interviewer asks you to review this codebase live, what concrete issues would you identify and fix first?

**Polished answer:**  
I would present the review in priority order rather than listing cosmetic issues.

**First: security and correctness.** Derive identity from `req.user` instead of client-supplied user IDs; verify payment status with PayPal; calculate order prices on the server; enforce stock atomically; protect order/address/cart/review ownership; make payment capture idempotent.

**Second: reliability.** Add centralized validation and error handling, use database transactions/conditional updates where required, improve the DB connection cache failure handling, and add explicit order-state transitions.

**Third: performance.** Add pagination and indexes, replace broad regex search with a scalable search strategy, avoid full collection reads, and move large file uploads off the application server.

**Fourth: frontend quality.** Fix incomplete `useEffect` dependencies, replace the search timeout with a real debounce, model errors separately from empty data, reset stale state deliberately, and add React `key` props where list items are mapped.

The strongest interview answer is not “this code is bad.” It is: “Here is the invariant, here is where the current code can violate it, here is the threat or failure mode, and here is the least-complex fix that makes the invariant explicit.”

**TL;DR:** Review by impact: security → correctness → reliability → performance → maintainability → cosmetics.

**Key mappings:**
- `server/controllers/auth/auth-controller.js`
- `server/controllers/shop/order-controller.js`
- `server/controllers/shop/cart-controller.js`
- `server/controllers/shop/address-controller.js`
- `server/controllers/shop/product-review-controller.js`
- `client/src/pages/shopping-view/search.jsx`
- `client/src/pages/shopping-view/listing.jsx`
- `server/controllers/shop/products-controller.js`

---

# Final interview checklist

Before an SDE interview, be able to answer these without opening the project:

1. What happens from clicking **Login** until the user becomes authenticated?
2. Where exactly is the JWT stored, who reads it, and who verifies it?
3. Why does the server need both authentication and authorization middleware?
4. Why can the frontend not be the security boundary?
5. How does a Redux thunk map to an Express controller?
6. Why is the cart a reference-based model while the order contains snapshots?
7. What breaks when a product is deleted while it is still in a cart?
8. Why must price and stock be recomputed on the backend?
9. Why is payment capture a distributed-systems problem rather than just an API call?
10. How would you make payment capture idempotent?
11. How would you prevent two buyers from overselling one unit of stock?
12. Which endpoints have object-level authorization risks?
13. Where would you add database indexes?
14. Where would pagination be required first?
15. Why is arbitrary regex search a scalability concern?
16. Why is `sessionStorage` acceptable for UI continuity but not for security decisions?
17. What should move from React component state into Redux, and what should remain local?
18. Why might RTK Query be a better fit for this server-state-heavy application?
19. What would you test first in a payment/inventory workflow?
20. What would you change before calling this production-ready?

## The single mental model to remember

**UI intent → authenticated API request → server-side authorization → validated business command → database/external-service operation → durable state transition → API response → Redux state → UI.**

For every feature, be able to identify the **source of truth**, the **authorization boundary**, the **business invariant**, the **failure mode**, and the **concurrency/idempotency strategy**. That is the level at which most strong SDE project discussions become much more impressive than a simple framework walkthrough.
