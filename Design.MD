# Attention

this design implemented by a human and AI only edited the format of MD and checked grammer issues.

# Wallet Service

## 1. Java-Based API

- Implementation:  
  Built in Java using Spring Boot for RESTful APIs, leveraging Java’s concurrency utilities for safe multi-threading.
- Multi-Threading Safety:  
  Critical sections (e.g., balance deduction) are protected with distributed locks using a tool like Redis Distributed Lock or ZooKeeper. This ensures that multiple threads or instances don’t deduct funds simultaneously, preventing race conditions.
- APIs:
  - `GET /wallet/{userId}/balance`: Fetches current balances (read from slaves).
  - `POST /wallet/{userId}/deduct`: Deducts funds atomically (writes to master).
- Distributed Locking:
  - Before deducting funds, acquire a lock based on `userId` (e.g., Redis key: `lock:user:{userId}`).
  - Lock timeout ensures no deadlocks, with retry logic for failed attempts.
- High Availability:  
  Deploy multiple instances behind a load balancer (e.g., Nginx or AWS ALB). Instances coordinate via distributed locks.

## 2. Database Layer (PostgreSQL with PgBouncer)

- PgBouncer:  
  Acts as a connection pooler to manage database connections efficiently, reducing overhead on PostgreSQL. Configured to route write queries to the master and read queries to slaves.
- Load Balancer:  
  Sits in front of PgBouncer instances (if multiple are deployed for redundancy). Distributes incoming API requests across Wallet Service instances and ensures even load on the database layer.
- PostgreSQL Setup:
  - **Master Node**: Handles all writes (e.g., balance updates, transaction logs). Uses Write-Ahead Logging (WAL) for durability.
  - **Slave Nodes**:
    - Replicated from the master using PostgreSQL streaming replication.
    - Partitioned by user location (e.g., sharding key: `user_location`, such as "SOUTH EAST", "SOUTH WEST").
    - Each slave node is assigned a subset of the data based on its position (e.g., Slave 1: SOUTH EAST Iran users, Slave 2: SOUTH WEST Iran users, etc.).
    - Read-only queries (e.g., balance checks) are load-balanced across slaves.
- Partitioning Logic:  
  Data is sharded by `userId` hashed with location metadata, ensuring even distribution. Slaves are geographically distributed to reduce latency for location-based reads.
- Consistency:  
  Master-slave replication introduces slight read-after-write latency. Use synchronous replication for critical operations (e.g., fund deduction) if strong consistency is needed, or rely on eventual consistency for non-critical reads.

# Advertising Management Service

## 1. Overview

- **Purpose**: Manages advertising campaigns, updates their status (e.g., active, paused) based on user budget and wallet balance, and ensures ads run only when funds are available.
- **Implementation**: Built as a microservice (e.g., using PHP Laravel or Java Spring Boot) for scalability and ease of deployment.

## 2. Database Layer (MongoDB)

- **Choice**: MongoDB is used as the primary database due to its schemaless nature, which allows flexibility in storing campaign data with varying structures. It supports higher write throughput and can efficiently populate data from other services that create campaigns.
- **Data Population**:
  - Campaigns are ingested into MongoDB from external services (e.g., a campaign creation tool) via APIs or event streams (e.g., Kafka topics like `campaign_created`).
  - Stores campaign details such as `campaignId`, `userId`, budget, status, and metadata.
- **Sharding**: MongoDB is sharded by `userId` to distribute campaign data across nodes, ensuring scalability for high write loads.

## 3. Key Features

- **Event-Driven Updates**:  
  Subscribes to a message broker (e.g., Kafka) for events like `balance_updated` from the Wallet Service to react in real-time without polling.
- **Campaign Management**:
  - Maintains a cache in Redis of active campaigns and their budgets to reduce database queries.
  - Campaigns are moved to Redis only when sufficient funds are confirmed via `GET /wallet/{userId}/balance`.
  - Inactive campaigns (e.g., paused due to low funds) are kept in MongoDB but removed from Redis.
- **APIs**:
  - `GET /campaigns/{userId}`: Returns active campaigns for a user (fetched from Redis if available, otherwise MongoDB).
  - `POST /campaigns/{campaignId}/update`: Updates campaign status (e.g., pause/resume) in MongoDB and syncs Redis.

## 4. Optimization

- **Caching**:
  - Stores wallet balances and active campaign statuses in Redis with a short TTL (e.g., 5 minutes) to minimize API calls to the Wallet Service and MongoDB queries.
  - Redis keys are structured as `campaign:{campaignId}` with budget and status fields.
- **Asynchronous Processing**: Publishes events (e.g., `campaign_paused`) to the message broker for downstream services (like Ad Display) to consume.

## 5. High Availability

- Deployed across multiple instances with a load balancer. Uses health checks to ensure only healthy instances handle traffic.
- MongoDB is deployed as a replica set with automatic failover to ensure database availability.

# Ad Display Service

## 1. Overview

- **Purpose**: Delivers advertisements to end users based on active campaigns managed by the Advertising Management Service.
- **Implementation**: A lightweight, stateless service (e.g., using PHP or ... with a CDN) optimized for low-latency ad serving.

## 2. Key Features

- **Ad Serving**:  
  Queries the Advertising Management Service via `GET /campaigns/{userId}` to fetch active ads.  
  Serves ads to users and logs impressions/clicks.
- **Event Logging**:  
  Publishes impression/click events to a Kafka topic (e.g., `ad_events`) for cost calculation and reporting.
- **Content Delivery**:  
  Uses a CDN (e.g., Cloudflare) to cache ad assets (images, videos) and reduce latency globally.

## 3. High Availability

- Horizontally scalable with multiple instances behind a load balancer. Stateless design ensures easy scaling and failover.
