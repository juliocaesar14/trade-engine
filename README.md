Real-Time Trade Clearing Engine

A high-performance order matching engine built with Node.js, Redis, and MySQL. Handles concurrent limit and market orders with price-time priority matching, snapshot-based crash recovery, and idempotent order submission.

Stack

Node.js and Express for the API layer. Redis for the order book, caching, and idempotency. MySQL for durable persistence of orders, trades, and snapshots. Nginx as a reverse proxy with load balancing. Docker for containerization.

Getting Started

Make sure Docker Desktop is installed and running.

Start all services.

docker compose up -d

Run migrations on first boot only.

npm run migrate

Check everything is up.

curl http://localhost/health

The server, MySQL, Redis, Nginx, phpMyAdmin, and Redis Commander all start together.

API Endpoints

Place an order.

POST /api/orders

Body.

client_id, instrument, side, type, price, quantity.

Pass an Idempotency-Key header to make the request idempotent.

Cancel an order.

POST /api/orders/:order_id/cancel

Get order status.

GET /api/orders/:order_id

Get order book.

GET /api/orders/orderbook?instrument=BTC-USD&levels=20

Get recent trades.

GET /api/orders/trades?limit=50

Health check.

GET /health

How the Matching Engine Works

Orders are stored in Redis sorted sets with price-time priority. Buy side is sorted by price descending, so the highest bid comes first. Sell side is sorted by price ascending, so the lowest ask comes first. Within the same price, the earliest order wins.

When a new order comes in, the engine locks the instrument queue, checks for matching orders on the opposite side, executes partial or full fills, and persists everything to MySQL atomically. Market orders match immediately against the best available limit orders until filled or the book is exhausted.

Race conditions are prevented using per-instrument in-memory queue locks with exponential backoff retry, up to 200 retries starting at 1ms.

Crash Recovery

On startup the engine loads the latest MySQL snapshot into Redis, then replays only the orders placed after that snapshot. This brings recovery time from around 30 to 50 seconds on a full replay of 60 to 100K orders down to 3 to 4 seconds. Snapshots are taken every 5 minutes and also on graceful shutdown. The recovery strategy is logged on startup.

Idempotency

Pass an Idempotency-Key header with any order submission. The engine checks Redis first for speed, then MySQL as a durable fallback. Duplicate requests return the exact same response with no double fills and no duplicate orders created.

Running Tests

Unit and integration tests.

npm test

Load test targeting around 2000 orders per second.

npm run load-test

Dev Tools

phpMyAdmin is available at http://localhost:8080. Login with root and rootpassword.

Redis Commander is available at http://localhost:8081.

Project Structure

trade-engine/
├── server.js                          entry point, starts all services
├── Dockerfile                         builds the Node.js container
├── docker-compose.yml                 runs the full stack locally
├── nginx.conf                         reverse proxy config
├── package.json                       dependencies and scripts
├── .env                               environment variables
│
├── routes/
│   └── order.routes.js                all API endpoint definitions
│
├── services/
│   ├── order.services.js              order creation, cancellation, queries
│   └── snapshot.services.js          periodic snapshots and crash recovery
│
├── helper/
│   └── matcher.js                     core matching engine
│
├── redis/
│   ├── client.js                      Redis connection setup
│   ├── RedisService.js                order book ops, caching, idempotency
│   └── index.js                       exports the Redis service
│
├── db/
│   ├── index.js                       exports DBForge query builder
│   ├── query.js                       lightweight MySQL query builder
│   ├── migration.js                   migration runner
│   └── migrations/
│       ├── 002_create_orders_table.js
│       ├── 003_create_trades_table.js
│       ├── 004_create_idempotency_keys_table.js
│       ├── 005_create_order_book_snapshots_table.js
│       └── 006_create_order_audit_log_table.js
│
├── cli/
│   └── migrate.js                     CLI to run or roll back migrations
│
├── tests/
│   ├── setup.js                       test environment setup
│   ├── helpers.js                     shared test utilities
│   ├── unit/
│   │   └── matcher.test.js            unit tests for the matching engine
│   └── integration/
│       ├── api.test.js                integration tests for all endpoints
│       └── matcher.integration.test.js  end to end matching tests
│
├── test-matching-engine.js            load test script
└── test-2000rps.js                    stress test at 2000 orders per second

Scaling to Production

The current single-node setup handles around 950 orders per second. To scale further, you can run multiple Node instances behind Nginx using docker compose up --scale app=4. Nginx already load balances across them using least-connection routing. For higher throughput, partition matching workers per instrument so BTC-USD and ETH-USD never contend on the same lock, shard the order book across a Redis Cluster, and offload read queries to MySQL replicas.

