My role: [to be filled by me]

# Authorization Service: Project Context

Paths relative to `services/authorization-service/` unless stated.

## 1. Summary
C++17 (`CMakeLists.txt`: `CMAKE_CXX_STANDARD 17`) Kafka-to-Postgres validation service. Libraries: librdkafka C++ (`rdkafka++`), libpqxx, nlohmann/json (`find_package(nlohmann_json 3.2.0)`), apt packages on Ubuntu 22.04 (`Dockerfile`). Consumes `payment_initiated` (group `authorization-service-group`), checks business rules in Postgres, publishes to `payment_authorized` (`status=AUTHORIZED`) or `payment_failed` (`status=FAILED`). Unparseable messages go to `payment_initiated.DLT`. Target 10,000 RPS appears in the `main.cpp` banner, root `README.md` and JMX files; no result backs it (section 8).

## 2. Flow
1. `main()` (`src/main.cpp`) reads `KAFKA_BROKERS` via `Config::getKafkaBrokers()` (default `localhost:9092`), `DB_CONN`, `THREADS`, `DB_CONNS` via `Config::getEnv` (`src/config/Config.cpp`). Builds `ThreadPool`, `ConnectionPool`, `AuthorizationEngine`, `KafkaProducer`, `KafkaConsumer`.
2. `KafkaConsumer::start` (`src/kafka/KafkaConsumer.cpp`) loops `consume(1000)`; `ERR_NO_ERROR` calls the callback with the payload string; other errors are only logged.
3. Callback copies the payload and calls `tpool.enqueue_detach(...)`, returning at once.
4. Worker: `nlohmann::json::parse` → `PaymentInitiatedEvent` (`from_json`, `include/PaymentModels.h`) → `AuthorizationEngine::authorize(req, reason)` (`src/services/AuthorizationEngine.cpp`).
5. Check order and exact `reason` strings:
   1. no row: `User not found`
   2. `intent_status` null: `Payment intent not found or mismatch`
   3. amount text differs: `Payment amount mismatch with intent`
   4. `method_type` null: `Payment method not found or does not belong to user`
   5. CARD and expired: `Payment method expired`
   6. `balance` null: `User account not found`
   7. `balance < amount`: `Insufficient funds in account balance`
   - `pqxx::sql_error` → `Database error during validation: <what>`; other `std::exception` → `Internal verification error: <what>`. Both return false: DB errors become `FAILED`, not retries.
6. `main.cpp` builds a `PaymentAuthorizedEvent` (6 input fields + `status` + `reason`, `"Clear"` on success), serializes with `to_json` (camelCase), calls `KafkaProducer::publish(topic, json)`.
7. Worker `catch (std::exception)`: log, `publish("payment_initiated.DLT", payload_copy)`.

## 3. Components
- Consumer (`KafkaConsumer.cpp`): conf only `bootstrap.servers`, `group.id`, `auto.offset.reset=earliest`. `subscribe` without rebalance callback. If `subscribe` fails, `start` logs and returns, so the process exits 0.
- Producer (`KafkaProducer.cpp`): conf only `bootstrap.servers`; `produce(..., PARTITION_UA, RK_MSG_COPY, key=NULL)` then `poll(0)`. Destructor `flush(10000)`. One shared instance.
- ThreadPool (`include/ThreadPool.h`): `std::vector<std::thread>`, `std::queue<std::function<void()>>`, one mutex + condition variable; `enqueue_detach` is fire-and-forget. Size from `THREADS` (default 100). Destructor sets `stop`, drains queue, joins.
- DB pool (`include/ConnectionPool.h`, `src/services/ConnectionPool.cpp`): `std::vector<shared_ptr<pqxx::connection>>`, one mutex + cv. Constructor opens all `DB_CONNS` (default 100) eagerly; if the DB is down at startup the constructor throws and `main` prints "Startup error", returns 1. `get_connection()` waits with no timeout, pops, checks `is_open()`, reconnects if closed; if reconnect throws it returns the dead connection anyway.
- RAII: `ConnectionGuard` (defined in `AuthorizationEngine.cpp`) returns the connection in its destructor.
- Models/JSON (`PaymentModels.h`): custom `from_json`/`to_json`; input accepts snake_case or camelCase, output camelCase. `amount` may be string or number, parsed to `double`; missing amount becomes 0.0.
- Shutdown: SIGINT/SIGTERM set `volatile bool running = false`; a second signal calls `std::exit`. Then consumer `close()`, pool drains, producer flushes. Not tested.
- `src/models/PaymentEvent.h` and service `README.md` are empty. `build/` is committed.

## 4. Authorization SQL (`AuthorizationEngine.cpp`)
```sql
SELECT u.id AS user_exists, pi.status AS intent_status, pi.amount::text AS intent_amount,
       pm.method_type, pm.expiry_year, pm.expiry_month, a.balance
FROM users u
LEFT JOIN payment_intents pi ON pi.id = $2::uuid AND pi.user_id = u.id
LEFT JOIN payment_methods pm ON pm.id = $3::uuid AND pm.user_id = u.id
LEFT JOIN accounts a ON a.user_id = u.id AND a.account_type = 'USER_WALLET'
WHERE u.id = $1::uuid
```
- Bound via `pqxx::nontransaction::exec_params(query, user_id, payment_intent_id, payment_method_id)`; server-side params, no concatenation.
- Covers: user exists, intent exists and belongs to user, amount (compared as text), method belongs to user, `USER_WALLET` balance.
- Second query only for `CARD`: `SELECT EXTRACT(YEAR FROM CURRENT_DATE)::int AS yr, EXTRACT(MONTH ...) AS mo` (DB clock). Autocommit, no explicit transaction.
- Not checked: `intent_status` value (only null), currency, duplicate intent, `transaction_id`. No reservation or debit.
- Column names are confirmed by root `seed.sql` (INSERTs into `users`, `payment_methods`, `accounts`, `payment_intents`; intents seeded with `status='CREATED'`). Column types are not in the repo; `numeric(15,2)` is [unverified] (code comment only).

## 5. Failure and delivery behaviour
- Offsets: no `enable.auto.commit`, manual commit or `store_offset` anywhere. librdkafka defaults apply (auto-commit on, 5 s interval; offset auto-stored when `consume()` returns the message), so offsets advance before the worker runs.
- Worker throws (bad JSON, `std::stod` failure, wrong JSON type): payload copied to `payment_initiated.DLT`. Missing IDs are not exceptions (`j.value` returns `""`); `""::uuid` raises `pqxx::sql_error`, so result is `FAILED` with a database-error reason, not DLT.
- DLT publish failure: `publish()` returns false and never throws, so the inner `catch` cannot fire and "routed to DLT" is logged regardless. No delivery-report callback, so async produce failures are invisible. Return values for `payment_authorized`/`payment_failed` are ignored too.
- Crash/restart: queued and in-flight tasks are lost; their offsets may already be committed, so they are not redelivered. Messages consumed after the last auto-commit are redelivered and reprocessed, giving duplicate output events.
- Guarantee provided: neither at-least-once nor exactly-once. Loss is possible (crash with queued tasks, silent produce failure); duplicates are possible (replay after crash, no idempotency key).

## 6. Hard problems solved
1. Amount vs `numeric(15,2)`: false "amount mismatch" (code comment "FIX 3" cites `13644.959999`). Cause: `std::to_string(double)`. Fix: compare `pi.amount::text` with `std::fixed << std::setprecision(2)` (`AuthorizationEngine.cpp`). No test.
2. camelCase vs snake_case: gateway DTO `PaymentInitiatedEvent` emits camelCase. Fix: nested `j.value("snake", j.value("camel", ""))` plus string-or-number `amount` (`PaymentModels.h`). No test.
3. Blocking consumer: fix is `enqueue_detach` so `consume()` never waits on DB work (`main.cpp`). No test.
4. Connection exhaustion: fixed pool, blocking `get_connection()`, RAII guard; `authorization.yaml` points `DB_CONN` at `pgbouncer-service` (`pgbouncer.yaml`: `POOL_MODE=transaction`, `DEFAULT_POOL_SIZE=800`, `MAX_CLIENT_CONN=15000`, 2 replicas). No test.
Commit history of these fixes is [unverified] (no `.git`).

## 7. Concurrency and shared state
- Shared: `ThreadPool` queue (mutex+cv), `ConnectionPool` vector (mutex+cv), one `KafkaProducer`, `AuthorizationEngine` (stateless besides pool). No deadlock path found; locks are held briefly.
- `volatile bool running` is written in a signal handler (not `sig_atomic_t`).
- Risks: (a) unbounded `tasks` queue, no backpressure; if DB is slower than ingest, memory grows until the 1Gi limit OOMs the pod and queued messages are lost, with offsets already committed; (b) `THREADS=100` vs `DB_CONNS=20` in `authorization.yaml`: up to 80 workers wait on the cv; (c) `get_connection()` has no timeout; (d) failed reconnect hands out a dead connection (result then `FAILED`);
- Parallelism cap: `KafkaConfig.java` (gateway) declares `payment_initiated` with `PARTITIONS = 3`, while broker default is `KAFKA_CFG_NUM_PARTITIONS=100` (`kafka.yaml`). HPA is 15 to 60 pods (`hpa.yaml`). With 3 partitions only 3 pods in the group get messages. Live partition count [unverified].

## 8. Testing and measured results
- Tests for this service: none (no test directory, no gtest).
- JMeter load tests (`tests/`, `load-testing/`) call gateway `POST /api/v1/payments`, not this service. `tests/load-test.jmx`: 3000 threads, 180 s, `ConstantThroughputTimer` 600000/min ("Limit to 10000 req/sec"), fixed body `amount: 500.00`, `currency: INR`. `tests/load-test-2500rps.jmx`: 1500 threads, 30 s ramp, 120 s.
- `seed.sql`: one user, one `USER_WALLET` 50000.00, 2,000,000 intents of 500.00.
- Results: no result files, no latency or RPS numbers. 10,000 RPS: [unverified in repo]. `script.md` claim that the gateway scaled to 80 pods: [unverified in repo].

## 9. Known limitations and next improvements
- Manual offset commit after publish; delivery reports; retry with backoff.
- Bounded queue or consumer pause/resume (backpressure).
- Idempotency keyed on `transaction_id`; today a replay re-authorizes and re-publishes.
- No balance reservation here; concurrent payments from one wallet can all pass. `double` used for money.
- Empty IDs reach SQL; DB outage yields `FAILED`.
- No metrics, probes (`authorization.yaml`) or structured logging.
- Secrets: plaintext `adminpassword` in `DB_CONN` (`authorization.yaml`), no Kubernetes Secret.

## 10. Likely interview questions
1. Why a thread pool instead of processing in the consumer thread? `main.cpp` `enqueue_detach`.
2. What delivery guarantee do you provide? Section 5; no commit config in `KafkaConsumer.cpp`.
3. What if the DB is down? `authorize` catch blocks return `FAILED`.
4. How do you prevent SQL injection and bad UUIDs? `exec_params`, `$n::uuid`.
5. Why compare amount as text? `AuthorizationEngine.cpp` FIX 3 comment.
6. Why 100 threads but 20 connections? `authorization.yaml`.
7. Where can the queue grow without bound? `ThreadPool.h` `tasks`.
8. How does the pool avoid leaks? `ConnectionGuard`.
9. How do you reach 10k RPS with 3 partitions? `KafkaConfig.java`, `hpa.yaml`.
10. How do you stop double-spending? No reservation in the query; balance is debited later by `persistence-worker` (`PersistenceService.java`).

## 11. Interfaces and discrepancies
Interfaces (confirmed in other services' code only):
- In: gateway publishes `payment_initiated` (`PaymentEventPublisher.java`), key `transactionId`, `JsonSerializer`, `acks=1`, fields `paymentIntentId, transactionId, userId, paymentMethodId, amount (BigDecimal), currency, status, timestamp`.
- Out: gateway `KafkaConfig.java` declares `payment_authorized` and `payment_failed` (3 partitions, 3 replicas). Clearing reads `transactionId`, `amount` from `payment_authorized` (`ClearingService.java`). No consumer of `payment_failed` or `payment_initiated.DLT` found. `payment_initiated.DLT` is created nowhere in the repo; broker auto-create is [unverified].
- Balance is only changed downstream: `persistence-worker` runs `UPDATE accounts SET balance = balance + ?` with the negated amount, so auth reads a balance that lags.

Discrepancies:
- `script.md` says failures are retried 3 times with backoff then sent to a DLQ; that is Clearing (`@RetryableTopic`). Auth has no retry; DLT publish is one attempt.
- `script.md` says duplicates are stopped "at the Auth Service" (line 14); code has no dedup (gateway uses Redis, `IdempotencyService.java`).
- Root `README.md` says it validates "fraud rules"; none exist.
- `authorization.yaml` `replicas: 2` ("Matched to your HPA minimum") vs HPA `minReplicas: 15`.
- Code defaults `THREADS`/`DB_CONNS` 100 vs deployment `DB_CONNS=20`.
