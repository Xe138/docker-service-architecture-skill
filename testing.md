# Testing Patterns

Testing infrastructure for Docker-based projects: 3-stage pipeline, environment isolation, and rich progress display.

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

### Git Worktree Isolation

When using git worktrees for parallel feature development, additional isolation is required beyond container and network naming.

**Problem**: Without `COMPOSE_PROJECT_NAME`, Docker Compose uses the directory name as the project name. Since all test directories are typically named `test`, all worktrees share the same image names (e.g., `test-price-db:latest`). When worktree A builds while worktree B is testing, B's containers can fail due to image conflicts.

**Solution**: Set `COMPOSE_PROJECT_NAME` to include the instance ID:

```bash
# scripts/test-service.sh
TEST_INSTANCE_ID=$(./scripts/get-test-instance-id.sh)
export TEST_INSTANCE_ID

# CRITICAL: Isolate Docker images between worktrees
# Without this, all worktrees share image names like "test-price-db:latest"
export COMPOSE_PROJECT_NAME="test-${TEST_INSTANCE_ID}"

cd "services/${SERVICE}/deploy/test"
docker compose up -d --build --wait
```

**Result**: Each worktree gets unique image names:
- `test-feature-auth-price-db:latest` (worktree on feature/auth)
- `test-feature-api-price-db:latest` (worktree on feature/api)
- `test-main-price-db:latest` (main branch)

**What gets isolated with full implementation**:

| Resource | Isolation Variable | Example |
|----------|-------------------|---------|
| Container names | `container_name: mydb-test-${TEST_INSTANCE_ID}` | `mydb-test-feature-auth` |
| Network names | `name: myproject-test-${TEST_INSTANCE_ID}` | `myproject-test-feature-auth` |
| Image names | `COMPOSE_PROJECT_NAME=test-${TEST_INSTANCE_ID}` | `test-feature-auth-mydb:latest` |

### Cleanup Command Scoping

**Problem**: A global `test-clean-all` command that removes all containers matching `name=test-` will interfere with tests running in other worktrees.

**Solution**: Scope cleanup to the current branch's instance ID:

```makefile
# BAD: Removes ALL test containers across all branches
test-clean-all:
	@docker ps -a --filter "name=test-" --format "{{.Names}}" | xargs -r docker rm -f

# GOOD: Only removes containers for current branch
test-clean-all:
	@TEST_INSTANCE_ID=$$(./scripts/get-test-instance-id.sh); \
	echo "Cleaning up test containers for instance: $$TEST_INSTANCE_ID..."; \
	docker ps -a --filter "name=test-$$TEST_INSTANCE_ID" --format "{{.Names}}" | xargs -r docker rm -f 2>/dev/null || true; \
	docker network rm "myproject-test-network-$$TEST_INSTANCE_ID" 2>/dev/null || true
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

## Service Test Script Template

```bash
#!/bin/bash
# scripts/test-service.sh
set -e

SERVICE=$1
TEST_INSTANCE_ID=$(./scripts/get-test-instance-id.sh)
export TEST_INSTANCE_ID

# CRITICAL: Isolate Docker images between worktrees
export COMPOSE_PROJECT_NAME="test-${TEST_INSTANCE_ID}"

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
