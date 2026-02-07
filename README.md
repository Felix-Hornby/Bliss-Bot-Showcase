# BlissBotV2 - Portfolio Showcase

This folder contains the technical documentation for BlissBotV2, a distributed Discord bot ecosystem designed for high-performance game server management.

## Contents

### 1. Structure_Deep_Dive.md
A comprehensive architectural overview covering:
- Infrastructure Design: Hybrid Render.com + OVH VPS architecture
- Data Flow Patterns: User commands, save file parsing, cache expiration
- Deployment Pipeline: Build processes, Docker orchestration, health monitoring
- Performance Optimisations: Async executors, spatial grids, connection pooling
- Scalability Considerations: Future paths for horizontal scaling


### 2. Key_Logic_Snippets.md
Deep dives into the 11 most significant technical challenges I solved, including:
- Asynchronous RCON execution (powering the two-way Discord/ARK chat bridge)
- Custom Discord embed system with auto-truncation
- Interactive pagination with Discord UI views
- Unicode/emoji sanitisation for display names
- Intelligent Redis caching with expiration listeners
- Regex-based log parsing (tracking player activity for automated Activity Points (AP) rewards)
- Spatial grid clustering for base detection (O(n²) → O(n) optimisation)
- Custom decay calculation algorithm
- Chunked database insertion with retry logic
- Multi-bot orchestration with auto-restart

Target Audience: Senior engineers, technical interviewers, code reviewers

## Project Highlights

Scale:
- Six concurrent Discord bots
- Hundreds of active users
- High volume of database records, 27 tables, redis caching
- 10,000s of game structures processed per server per call

Technologies:
- Backend: Python (discord.py, FastAPI, asyncio)
- Databases: PostgreSQL, Redis
- Infrastructure: Docker, Render.com, OVH VPS
- APIs: Discord Gateway, Nitrado API, RCON Protocol

Key Achievements:
- High uptime in production environment
- Significant reduction in API calls via intelligent caching
- Major performance improvement both in terms of time complexity and memory complexity for save file parsing
- Minimal downtime deployments with auto-restart mechanisms

## Metrics

```text
Metric                   Value
-----------------------  -----------------------------------
Project Scale            ~9.3k python. Across 36 files
Test Coverage            High coverage (core utilities)
Architecture             Modular Monolith (6 bots, Parser on VPS)
API Ecosystem            3 (Discord API, Nitrado API, RCON)
Database Tables          PostgreSQL(27 Tables), Redis (Caching)
Infrastructure           Hybrid Deployment (Render + OVH VPS)
```

## What This Demonstrates

- Backend Expertise: Python (discord.py, FastAPI, asyncio), PostgreSQL, Redis
- Distributed Systems: Multi-server architecture, caching, pub/sub patterns
- Performance Engineering: Algorithm optimisation, async programming, connection pooling
- Production Readiness: Error handling, exponential backoff retry logic, health monitoring through Uptime Robot, auto-recovery
- Dependency Management: Use of pyproject.toml, poetry and Dependabot for dependency management
- API Integration: Discord, Nitrado, RCON protocols
- Database Design: Chunked inserts, upserts, connection pooling
- Testing: Parametrised unit tests with pytest, mocking, async test support
- DevOps: Docker, deployment automation, monitoring through Uptime Robot

---

Author: Felix Hornby  
Project Status: Production (Active)
