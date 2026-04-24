# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Python backend learning/notes project (Python 3.11) with a PostgreSQL 15 database. The `docs/` folder contains reference notes in Vietnamese covering system design, authentication/authorization, database indexing, and Docker optimization.

## Package Manager

This project uses **`uv`** for dependency management (not pip or poetry).

```bash
# Install dependencies
uv sync

# Add a dependency
uv add <package>

# Run the app
uv run python main.py

# Run a script or tool
uv run <command>
```

## Database

PostgreSQL 15 runs via Docker Compose, exposed on port **5433** (maps to container's 5432).

```bash
# Start the database
docker compose up -d

# Stop the database
docker compose down

# Connect via psql
psql -h localhost -p 5433 -U myuser -d mydatabase
```

Credentials (from `docker-compose.yml`):

- User: `myuser`
- Password: `mypassword`
- Database: `mydatabase`

## Architecture

- **`main.py`** — entry point (placeholder, to be developed)
- **`pyproject.toml`** — project metadata and dependencies (Python ≥ 3.11)
- **`docker-compose.yml`** — PostgreSQL 15 service with persistent volume `pgdata`
- **`docs/`** — reference notes (Vietnamese):
  - `db-init.md` — PostgreSQL index practice SQL (500k users, 2M orders, ~6M order_items)
  - `db-engineering.md` — Index types (B+Tree, Hash, Bitmap, GIN), query optimization, SARGable queries
  - `system-design.md` — Latency, throughput, scalability, CAP theorem, consistency models
  - `authen-author.md` — Authentication (JWT, OAuth 2.0, OIDC, session, MFA) and authorization
  - `docker-optimization.md` — Multi-stage builds, layer caching, slim/distroless base images
