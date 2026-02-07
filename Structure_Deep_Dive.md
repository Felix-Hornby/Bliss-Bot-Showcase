# BlissBotV2 - System Architecture Deep Dive

## Executive Summary

I designed BlissBotV2 as a production-grade, multi-server Discord bot ecosystem for ARK: Survival Ascended. In this architecture, I demonstrate advanced distributed patterns by combining real-time Discord interactions with asynchronous game server parsing, RCON command execution, PostgreSQL queries, and Redis caching strategies.

Key Technical Achievements:
- Multi-bot orchestration (6 concurrent Discord bots)
- Distributed architecture spanning Render.com (bot layer) and OVH VPS (data/backend layer)
- Real-time game save file parsing (handling large binary files)
- Custom pagination and embed systems for Discord's API constraints
- Redis-based intelligent caching with expiration listeners
- PostgreSQL database with chunked data insertion for large datasets

---

## Infrastructure

### High-Level Architecture

```
       ┌──────────┐       ┌──────────────────────────┐
       │   USER   │ ◄────►│   Discord API Gateway    │
       └──────────┘       └────────────┬─────────────┘
                                       │ HTTPS / WebSocket
                                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RENDER.COM (Cloud)                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │   StartUpAll.py - Bot Orchestration Layer               │   │
│   │   └─ 6 Discord Bots (Master, Server, AP, GP, etc.)      │   │
│   └─────────────┬───────────────────────────────────────────┘   │
│                 │                                               │
│   ┌─────────────▼────────────┐                                  │
│   │  PostgreSQL Database     │                                  │
│   └──────────────────────────┘                                  │
└───────┬──────────────┬──────────────────────┬───────────────────┘
        │              │                      │
        │ FastAPI      │ Cache Access         │ RCON / HTTPS (Info)
        ▼              ▼                      │
┌──────────────────────────────────────┐      │
│               OVH VPS                │      │
│  ┌────────────────────────────────┐  │      │
│  │ Docker Compose Stack           │  │      │
│  │ ├─ Redis (Server Data Cache)   │  │      │
│  │ └─ FastAPI (Save File Parser)  │  │      │
│  └───────────────┬────────────────┘  │      │
└──────────────────┼───────────────────┘      │
                   │                          │
                   │ HTTPS (Save file stream) │
                   ▼                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                EXTERNAL GAME SERVER (Nitrado)                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Nitrado API & Binary File System (.ark, .profile, .gz)    │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Game Server RCON Connection (Real-time commands/chat)     │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Nitrado Management Endpoints (Status, Health, Data)       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### Render Service (Cloud Layer)
- Role: User-facing bot interactions, asynchronous command processing, and relational data management
- Technology Stack: Python 3.11+, asyncio, discord.py, PostgreSQL (Managed Service)
- Orchestration: Custom `StartUpAll.py` script managing 6 concurrent bot instances
- Scaling Strategy: Vertical scaling for concurrent bot execution and managed database performance

### Bot Ecosystem Detail

The system operates 6 specialized bot instances concurrently, each isolated into its own token and rate-limit bucket for maximum reliability:

1. Master Bot: The central orchestration bot. I designed it to handle core setup tasks, including inviting the 5 other bots to imporve user setup experience and for future guild-specific configurations to allow the system to be used in multiple guilds and game clusters in the future.
2. Server Management Bot: The primary bridge to the game servers. It executes allows remote command execution via RCON (a system that also powers the two-way Discord/ARK chat bridge), interprets real-time logs to track player sessions and in-game chat activity, and sends said information to organised discord threads, provides several commands to fetch key server data through the Nitrado API (including core server info, playerlists, status and server resource utilisation), management of server status, allowing for quick restarts to maximise server uptime and a suite of info commands to query the database for data extracted by the VPS during save file parsing, providing high level insights into game servers.
3. AP Bot (Activity Points): The engine for my automated engagement economy. It calculates player uptime and activity from join/leave logs, awarding "Activity Points" that reflect player commitment and can be redeemed on ingame items and services.
4. GP Bot (Guardian Points): A points system through which points are rewarded for donating to boost the life of the server. It provides an intensive order logging system and automatic logs od any donations made, accessed through the Nitrado API.
5. Ticket Bot: A streamlined support management system. It leverages Discord's modern UI components (buttons, threads) to allow players to open support tickets and staff to manage resolutions efficiently. Although there are many ticket bots available for discord, I noticed that they do not provide intensive automation, so I architected my own with automation in mind (particularly automatic ticket creation).
6. Assistant Bot: General user utilities that do not fit into the other categories. Includes relative iso timestamp generation, and a coordinate proximity utility (finding the closest teleport to a player's desired destination).

#### OVH VPS (Data Layer)
- Role: Heavy computation (binary save file parsing and data streaming)
- Technology Stack:
  - Docker Compose orchestration
  - FastAPI 
  - Redis 7 (caching + pub/sub for keyspace events)
- Scaling Strategy: Vertical scaling with connection pooling (psycopg2 pool)
- Storage: Temporary file system for .ark file processing

---

## Data Flow

### 1. User Command Execution Flow

```
User → Discord → Render Bot → RCON/Database → Game Server/VPS → Response → Discord
```

Example: `/server status` command

1. User Interaction: User invokes slash command in Discord
2. Bot Processing (Render):
   - `ServerManagementBot` receives interaction
   - Validates user permissions via `is_allowed_role()` decorator
   - Retrieves server IP/port from Redis cache (or PostgreSQL if cache miss)
3. RCON Execution:
   - `remote_command_execute()` wraps synchronous RCON client in async executor
   - Sends command to game server (e.g., `ListPlayers`)
   - Handles timeout/connection errors with custom embed responses
4. Response Formatting:
   - Output paginated using `paginate()` (4000 char limit per Discord embed)
   - Custom `CreateEmbed` class auto-truncates titles/descriptions
   - `Paginator` UI view provides ⬅️/➡️ navigation buttons
5. Discord Response: Interaction response sent with embeds + view

### 2. Save File Parsing Flow

```
Scheduler → VPS FastAPI → Nitrado API → Download → Parse → Chunk → PostgreSQL
```

Example: Hourly player data update

1. Trigger: Scheduled task or manual API call to `/parse-server` endpoint
2. File Discovery (VPS):
   - Query Nitrado API for server file list
   - Identify latest `.ark` save file (by `modified_at` timestamp)
3. Download & Decompression:
   - Async download with `httpx` (streaming to avoid memory overflow)
   - Decompress `.gz` files using `gzip` module
   - Store in temporary directory with automatic cleanup
4. Parsing (arkparse library):
   - Initialise `AsaSave` object (binary parser)
   - Extract `PlayerApi`, `DinoApi`, `StructureApi` data
   - Fallback: Download `.arkprofile`/`.arktribe` if main save lacks data
5. Data Transformation:
   - Convert parsed objects to JSON-serializable dictionaries
   - Extract nested properties (e.g., dino stats, structure decay timers)
   - Calculate base decay using custom `calculate_decay()` algorithm
6. Chunked Insertion:
   - Split data into 200-item chunks (avoid PostgreSQL query size limits)
   - Use `execute_values()` for batch inserts (50 rows per page)
   - Atomic transactions with rollback on failure
7. Cache Invalidation:
   - Update Redis with new server details
   - Set 5-minute TTL on cache keys

### 3. Redis Cache Expiration Listener Flow

```
Redis Keyspace Event → Pub/Sub → Async Listener → Re-cache → Set TTL
```

Purpose: Maintain fresh server details without constant API polling

1. Setup:
   - Redis configured with `notify-keyspace-events Ex` (expiration events)
   - `server_details_expiry_listener()` subscribes to `__keyevent@0__:expired`
2. Event Detection:
   - When `server_details:{server_id}` key expires, event fires
   - Listener receives message in async loop
3. Re-caching:
   - Calls `redis_cache()` to fetch fresh data from Nitrado API
   - Implements retry logic with exponential backoff (tenacity library)
   - On 5th retry failure, persists keys (removes TTL) and starts health check loop. Halts calling of server parse endpoint to prevent data being purged (graceful degradation, preserving stale data in favour of no data)
4. Fallback Mechanism:
   - `nitrado_api_health_listener()` polls Nitrado status endpoint every 5 minutes
   - When API recovers, re-enables expiration listener and resumes server parse calls.
Entirely idempotent architecture, safe to call multiple times without side effects.

---

## Deployment Pipeline

### Render.com Deployment

Build Process:
```bash
1. Install Python dependencies (discord.py, psycopg2, redis, etc.)
2. Deploy as a Render Web Service (bound to $PORT for health checks)
```

Runtime Execution:
The system relies on a custom `StartUpAll.py` orchestrator that manages the process lifecycle of the entire bot cluster:
1. Init: Shims missing dependencies and validates the environment layer for compatibility.
2. Bot Layer: Executes all 6 Discord bots concurrently using `asyncio.gather()`, with built-in auto-restart logic for each instance.
3. Health Monitoring: Maintains a minimal lightweight endpoint to satisfy platform health constraints and ensure high uptime.

Startup Sequence:
1. Load environment variables from `.env`
2. Shim missing `arkparse` modules (compatibility layer)
3. Initialise all 6 Discord bots with `asyncio.gather()`
4. Auto-restart bots on crash (infinite loop with exception handling)

Health Checks:
- Render monitors process health
- Bots auto-reconnect to Discord on connection loss

### VPS Deployment

Docker Compose Stack:
```yaml
services:
  redis:
    - Persistent volume for data
    - Keyspace notifications enabled
    - Password-protected
  
  parser:
    - Depends on Redis
    - Mounts shared volume
    - Auto-restart on failure
```

Deployment Steps (Automated via GitHub Actions):
1. **Trigger:** Push to `main` branch.
2. **CI Process:** GitHub Actions validates code and initiates deployment.
3. **File Sync:** `appleboy/scp-action` securely transfers the `vps/` directory to the OVH server.
4. **Remote Execution:** `appleboy/ssh-action` triggers the Docker build and container restart:
   ```bash
   cd ~/ark_server_parser
   docker compose up -d --build
   ```
5. **Success Notification:** Workflow confirms successful deployment and indicates that Render will automatically trigger its own build for the bot layer.

---

## Key Design Decisions

### 1. Why Split Render/VPS Architecture?
- Render Limitations: Limited CPU for heavy binary parsing, disk I/O constraints on large file downloads (.ark saves).
- VPS Strengths: Dedicated compute for large save file processing (some over 1GB in size, particularly older 2+ year old servers). By decoupling the parser and cache (Redis) from the main bots, I ensure that save processing never blocks user interactions or triggers OOM (Out of Memory) kills on the bot layer.
- Database Strategy: I host PostgreSQL on Render as a separate managed service. This provides low-latency access for the 6 bot instances while isolating the heavy persistent storage from the VPS's computation-heavy parsing tasks.
- Cost Optimisation: Render starter tier for bot hosting + Managed DB, VPS only for high-resource parsing tasks. This split saved me ~£13/month while providing 6x the RAM/CPU for the parser compared to a single monolithic Render service.

### 2. Why 6 Separate Bots Instead of 1?
- Discord Rate Limits: Separate tokens = separate rate limit buckets
- Modularity: Each bot has isolated cogs (commands), easier to debug
- Fault Isolation: One bot crash doesn't affect others
- User Experience: There are many different commands (over 50), using different bots with distinct icons helps users to identify the bot they need to use.
- Trade off: User needs to invite 6 bots instead of 1. I mitigated this by implementing a sort of setup wizard through the Master Bot that efficiently guides the user through the process of inviting the other 5 bots.

### 3. Why Redis + PostgreSQL?
- Redis: Fast reads for frequently accessed data (server details, user sessions)
- PostgreSQL: Relational integrity for complex queries (tribe members, base ownership)
- Hybrid Approach: Best of both worlds (speed + reliability)

### 4. Why Custom Pagination Instead of discord.py Built-in?
- Flexibility: Support for dynamic titles/colours per page
- Timeout Handling: Disable buttons on timeout (better UX)
- Better handling of larger query results.
- Reusability: Single `Paginator` class used across all cogs

---

## Performance Optimisations

### 1. Async Executor Pattern
- Wraps blocking I/O (RCON, database) in `loop.run_in_executor(None, func)`
- Prevents Discord bot event loop blocking
- Example: `remote_command_execute()` runs RCON client in thread pool

### 2. Spatial Grid Clustering (Base Detection)
- Groups structures into bases using 2500-unit grid cells
- Reduces O(n²) distance checks to O(n) with spatial hashing
- NumPy vectorization for distance calculations

### 3. Connection Pooling
- PostgreSQL pool (20 connections) shared across all operations
- Avoids connection overhead on every query
- Proper `getconn()`/`putconn()` lifecycle management

### 4. Chunked Data Processing
- Splits large datasets (10k+ players) into 200-item chunks
- Prevents memory overflow and query timeouts
- Atomic commits per chunk (partial success possible)

---

## Security Considerations

1. Environment Variables: All secrets (tokens, passwords) in `.env` (gitignored)
2. Role-Based Access Control: `is_allowed_role()` decorator on admin commands
3. SQL Injection Prevention: Parameterised queries with `psycopg2.sql.SQL()`
4. RCON Password Protection: Never logged or exposed in error messages

---

## Monitoring & Observability

- Logging: Python `logging` module with DEBUG level on VPS
- Health Checks: Render `/health` endpoint + Docker healthchecks
- External Monitoring: Uptime Robot integration for 24/7 service availability tracking
- Error Tracking: Discord embeds with red colour (0xFF0000) for failures
- Database Metrics: `last_updated` timestamps on intermittently updated tables

---

## Future Scalability Paths

- Horizontal Scaling: Move to Kubernetes for multi-pod bot deployment
- Database Sharding: Partition PostgreSQL by server ID for greater scalability
- Message Queue: Add RabbitMQ for async task processing (replace direct API calls)
- Caching Layer: Implement Redis Cluster for high availability

---

## Conclusion

With BlissBotV2, I built a production-ready, distributed system that balances cost efficiency with performance. This architecture demonstrates my expertise in:
- Cloud-native design (Render + VPS hybrid)
- Async Python patterns (asyncio, executors, connection pooling)
- Real-time data processing (binary save file parsing, RCON)
- Discord API Implementation (custom embeds, pagination, slash commands)
- DevOps practices (Docker, CI/CD, health monitoring)


