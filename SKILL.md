---
name: docker-service-architecture
description: Use when setting up a new project with multiple services that need Docker containerization, multi-environment deployments (dev/test/prod), CI/CD pipelines, or multi-stage testing (unit/service/integration). Also use when adding testing infrastructure to existing Docker projects.
---

# Docker Service Architecture

## Overview

Pattern for organizing multi-service projects with Docker containerization, environment-specific deployments, and a 3-stage testing pipeline. Key principles: **service isolation**, **ephemeral test environments**, **branch-isolated testing**, and **context-efficient test output**.

## When to Use

- Setting up a new multi-service project
- Adding Docker containerization to existing services
- Creating CI/CD pipelines for containerized services
- Implementing multi-stage testing (unit → service → integration)
- Need parallel test execution across git branches

## Related Documentation

| Topic | File | Use When |
|-------|------|----------|
| Testing patterns | testing.md | Setting up 3-stage testing, branch isolation, test runners |
| CI/CD pipelines | ci-cd.md | Configuring GitHub or Gitea automated Docker builds |

## Single-Service Projects

For projects with only one service, simplify the patterns:

**Directory structure** - Flatten by removing `services/` layer:
```
project/
├── src/                    # Source code (not services/api/src/)
├── tests/
│   ├── unit/
│   └── integration/
├── deploy/
│   ├── dev/
│   ├── test/
│   └── prod/
├── Dockerfile
└── Makefile
```

**Testing stages** - Collapse to 2 stages (skip service tests):
```
┌─────────────┐     ┌─────────────────┐
│ Unit Tests  │ ──► │Integration Tests│
│ (no Docker) │     │  (with Docker)  │
└─────────────┘     └─────────────────┘
```

Service tests exist to validate individual services before full-stack integration. With one service, integration tests serve this purpose directly.

**CI/CD** - Remove matrix strategy (single build target):
```yaml
build:
  steps:
    - run: docker build -t myapp:$VERSION .
```

**When to use multi-service structure anyway:**
- Service will likely split into multiple services later
- You want consistency with other multi-service projects
- The "service" is actually multiple containers (app + worker + scheduler)

## Directory Structure

```
project/
├── services/
│   ├── service-a/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml (or package.json)
│   │   ├── service_a/          # Source code
│   │   ├── tests/
│   │   │   ├── unit/           # No containers needed
│   │   │   └── integration/    # Needs service containers
│   │   └── deploy/
│   │       └── test/
│   │           └── docker-compose.yml  # Service-isolated tests
│   └── service-b/
│       └── ... (same structure)
├── deploy/
│   ├── dev/
│   │   ├── docker-compose.yml
│   │   └── .env.example
│   ├── test/
│   │   └── docker-compose.yml  # Full-stack integration
│   └── prod/
│       ├── docker-compose.yml
│       └── .env.example
├── scripts/
│   ├── test-runner.py          # Rich progress display
│   ├── test-service.sh         # Service-level test runner
│   ├── run-integration-tests.sh
│   └── get-test-instance-id.sh
├── .gitea/workflows/           # Or .github/workflows/
│   └── release.yml
└── Makefile
```

## Docker Compose Patterns

### Development Environment

```yaml
# deploy/dev/docker-compose.yml
services:
  api:
    build:
      context: ../..
      dockerfile: services/api/Dockerfile
    ports:
      - "${API_PORT:-8000}:8000"  # Configurable external port
    volumes:
      # Mount source for hot-reload
      - ../../services/api/src:/app/src:ro
    depends_on:
      database:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  postgres-data:
    name: myproject-dev-postgres  # Named volumes persist
```

### Test Environment

```yaml
# deploy/test/docker-compose.yml
services:
  database:
    # Ephemeral storage for speed
    tmpfs:
      - /var/lib/postgresql/data
    # PostgreSQL tuning for tests
    command: >
      postgres
      -c synchronous_commit=off
      -c fsync=off
      -c full_page_writes=off
    healthcheck:
      interval: 5s   # Faster checks for tests
      retries: 10
      start_period: 10s
```

### Production Environment

```yaml
# deploy/prod/docker-compose.yml
services:
  api:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "4"
        reservations:
          memory: 1G
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
```

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Fixed ports in test compose | Parallel runs conflict | Use dynamic ports: `ports: ["5432"]` |
| Persistent test volumes | Slow, stateful tests | Use `tmpfs` for ephemeral storage |
| No healthchecks | Race conditions on startup | Add healthcheck + `depends_on: condition: service_healthy` |
| Verbose test output | Context exhaustion | Use rich progress display, details only on failure |
| Single test stage | Slow feedback, hard to debug | Separate unit → service → integration |
| No branch isolation | Concurrent branches conflict | Use `TEST_INSTANCE_ID` from git branch |
| No `COMPOSE_PROJECT_NAME` | Worktrees share images, builds conflict | Set `COMPOSE_PROJECT_NAME=test-${TEST_INSTANCE_ID}` |
| Global test cleanup | Kills tests in other worktrees | Scope cleanup to current `TEST_INSTANCE_ID` |
| Building all services on every change | Slow CI | Use matrix strategy, only build changed services |

## Quick Reference

| Component | Dev | Test | Prod |
|-----------|-----|------|------|
| Storage | Named volumes | tmpfs | External mounts |
| Ports | Fixed (configurable) | Dynamic | Fixed |
| Healthcheck interval | 30s | 5s | 30s |
| Restart policy | None | None | unless-stopped |
| Resource limits | None | None | Set limits |
| Logging | Default | Default | json-file with rotation |
