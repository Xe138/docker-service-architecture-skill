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

## 3-Stage Testing Pattern

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐
│ Unit Tests  │ ──► │Service Tests│ ──► │Integration Tests│
│ (no Docker) │     │(per-service)│     │  (full stack)   │
└─────────────┘     └─────────────┘     └─────────────────┘
     Fast              Medium               Slow
   Mocked deps      Real DB, mocked      All services
                    external APIs         together
```

**Fail-fast**: Each stage only runs if previous stage passes.

### Stage 1: Unit Tests

- **Location**: `services/<name>/tests/unit/`
- **Dependencies**: All mocked
- **Containers**: None

### Stage 2: Service Tests

- **Location**: `services/<name>/tests/integration/`
- **Config**: `services/<name>/deploy/test/docker-compose.yml`
- **Dependencies**: Real database, mocked external APIs
- **Containers**: Service + its direct dependencies only

### Stage 3: Integration Tests

- **Location**: `tests/integration/` or `services/<name>/tests/integration/`
- **Config**: `deploy/test/docker-compose.yml`
- **Dependencies**: All services running
- **Containers**: Full stack

### Test Performance Optimization

**Requirement**: Minimize total test execution time. Slow tests waste developer time and CI resources.

- **Time all tests**: Use pytest's `--durations=10` to identify slowest tests
- **Investigate outliers**: Tests taking >1s in unit tests or >5s in integration tests need review
- **Common culprit**: Tests waiting for timeouts instead of using condition-based assertions
  ```python
  # BAD: Waits full 5 seconds even if ready immediately
  await asyncio.sleep(5)
  assert result.is_ready()

  # GOOD: Returns as soon as condition is met
  await wait_for(lambda: result.is_ready(), timeout=5)
  ```
- **Cache expensive setup**: Use `pytest` fixtures with appropriate scope (`module`, `session`)
- **Parallelize**: Use `pytest-xdist` for CPU-bound test suites

## Test Environment Isolation

### Branch Isolation

Use `TEST_INSTANCE_ID` derived from git branch for parallel testing:

```bash
# scripts/get-test-instance-id.sh
#!/bin/bash
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
# Sanitize: replace non-alphanumeric with dash, limit length
echo "$BRANCH" | sed 's/[^a-zA-Z0-9]/-/g' | cut -c1-20
```

### Docker Compose with Instance ID

```yaml
# deploy/test/docker-compose.yml
services:
  database:
    container_name: mydb-test-${TEST_INSTANCE_ID}
    # Use tmpfs for ephemeral storage (fast, stateless)
    tmpfs:
      - /var/lib/postgresql/data
    # Dynamic ports - no conflicts
    ports:
      - "5432"  # Docker assigns random host port
    networks:
      - test-network

networks:
  test-network:
    name: myproject-test-${TEST_INSTANCE_ID}
```

### Dynamic Port Discovery

```bash
# After containers start, discover assigned ports
API_PORT=$(docker compose port api 8000 | cut -d: -f2)
DB_PORT=$(docker compose port database 5432 | cut -d: -f2)

export TEST_API_URL="http://localhost:${API_PORT}"
export TEST_DATABASE_URL="postgresql://test:test@localhost:${DB_PORT}/testdb"
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

## CI/CD Pipeline

### Release Workflow (Gitea/GitHub Actions)

```yaml
# .gitea/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+*'

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: --health-cmd pg_isready
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: make test

  build:
    needs: test  # Only build if tests pass
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, worker, frontend]  # Build all services
    steps:
      - uses: actions/checkout@v4

      - name: Extract version
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "VERSION=$VERSION" >> $GITHUB_OUTPUT
          # Detect pre-release (contains hyphen: v1.0.0-beta)
          [[ "$VERSION" == *-* ]] && echo "PRERELEASE=true" >> $GITHUB_OUTPUT

      - name: Build and push
        run: |
          IMAGE=registry.example.com/myproject-${{ matrix.service }}
          docker build -t $IMAGE:${{ steps.version.outputs.VERSION }} \
            -f services/${{ matrix.service }}/Dockerfile .
          docker push $IMAGE:${{ steps.version.outputs.VERSION }}

          # Only tag 'latest' for stable releases
          if [ "${{ steps.version.outputs.PRERELEASE }}" != "true" ]; then
            docker tag $IMAGE:${{ steps.version.outputs.VERSION }} $IMAGE:latest
            docker push $IMAGE:latest
          fi
```

## Rich Test Runner

Context-efficient test output with real-time progress:

```python
# scripts/test-runner.py (simplified structure)
from rich.console import Console
from rich.live import Live
from rich.table import Table

class TestRunner:
    def run_all(self, unit=True, service=True, integration=True):
        with Live(console=self.console) as live:
            if unit:
                self.run_stage("Unit Tests", self.unit_targets)
                if not self.all_passed:
                    return False  # Fail fast

            if service:
                self.run_stage("Service Tests", self.service_targets)
                if not self.all_passed:
                    return False

            if integration:
                self.run_stage("Integration Tests", self.integration_targets)

        self.render_summary()
        return self.all_passed
```

### Output Parsing

```python
# Parse pytest progress: "tests/test_foo.py::test_bar PASSED [ 45%]"
PYTEST_PROGRESS = re.compile(r"\[\s*(\d+)%\]")
PYTEST_SUMMARY = re.compile(r"(\d+) passed(?:.*?(\d+) failed)?")

# Parse vitest: "✓ src/foo.test.tsx (11 tests) 297ms"
VITEST_PASS = re.compile(r"✓\s+(.+\.tsx?)\s+\((\d+)\s+test")
```

### Display Format

```
Unit Tests
  ● price-db      ━━━━━━━━━━░░░░░░░░░░  52%  497/955       12.3s
      → test_data_quality_validation
  ✓ orchestrator  74/74                                     3.2s
  ○ frontend      pending

Service Tests
  ○ price-db      pending
  ○ orchestrator  pending
```

## Makefile Integration

```makefile
VERBOSE ?= 0

test:
	@python scripts/test-runner.py $(if $(filter 1,$(VERBOSE)),-v)

test-verbose:
	@python scripts/test-runner.py -v

test-unit:
	@$(MAKE) -C services/api test-unit VERBOSE=$(VERBOSE)
	@$(MAKE) -C services/worker test-unit VERBOSE=$(VERBOSE)

test-service SERVICE:
	@./scripts/test-service.sh $(SERVICE)

test-integration:
	@./scripts/run-integration-tests.sh

test-infra-up:
	@cd deploy/test && docker compose up -d --build --wait

test-infra-down:
	@cd deploy/test && docker compose down -v
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

## Service Test Script Template

```bash
#!/bin/bash
# scripts/test-service.sh
set -e

SERVICE=$1
TEST_INSTANCE_ID=$(./scripts/get-test-instance-id.sh)
export TEST_INSTANCE_ID

# Start service-specific containers
cd "services/${SERVICE}/deploy/test"
docker compose up -d --build --wait

# Discover ports
API_PORT=$(docker compose port api 8000 | cut -d: -f2)
export TEST_API_URL="http://localhost:${API_PORT}"

# Run tests
cd "../.."
uv run pytest tests/integration/ --integration
TEST_EXIT=$?

# Cleanup
cd "deploy/test"
docker compose down -v

exit $TEST_EXIT
```
