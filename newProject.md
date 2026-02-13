# Prompt Template: Creating New TypeScript Project

This prompt scaffolds a complete TypeScript project with strict linting, testing, and security checks.

---

PROMPT FOR LLM (replace placeholders before use):

I need new project, project root should be in {{PROJECT_ROOT}}

Project is about {{PROJECT_DESCRIPTION}}

Author is Damir Manapov, license is MIT

My stack is TypeScript, pnpm, vitest, bun, biome, gitleaks

Make sure project uses latest versions of dependencies.

## Lint, format, check

Make sure biome rules and tsconfig are strict.

Tests should also be checked by tsconfig, lint and formatting.

Make sure lint output errors if
* deprecations used in code
* "any" used

Use latest version of biome and latest file config format.

I need 2-spaced across project files.

Make sure there are no vulnerabilities.

At the root of project create scripts:

* `check.sh` - runs formatting (fixing issues), check lint, check build (without emitting), run tests
* `health.sh` - checks gitleaks (including git), check dependencies used have up-to-date versions, that there are no vulnerabilities. If any outdated dep or vulnerability found script should fail
* `all-checks.sh` - runs both scripts

If you add instructions to install some software don't use project root for that and don't forget to add instructions on deleting temporary files if any.

If you are writing bash/sh scripts keep them simple, no fancy formatting. Make sure output is informative. Make sure such scripts fail fast.

Don't forget about `.gitignore`, `README.md`.

Don't mention author and license in README, mentioning in `package.json` is enough.

Run `all-checks.sh`, make sure it passed without errors.

In `package.json` dependencies should be above devDependencies, author and license in upper part of file.

## If you need docker compose
* Use `docker compose` instead of `docker-compose`
* Make commands in `package.json` with `compose:` prefix (`compose:pull`, `compose:up`, `compose:down`, `compose:restart`, `compose:reset`)
* On reset don't forget to delete volumes and orphans
* Don't put version to compose file, it's obsolete
* Use latest versions of images, but pin them by version tags
* Add starter service to compose file that waits all other services started successfully. Target it when starting docker compose. Place that service first
* Don't set `restart` and `container_name` options in compose file

## If project has Docker images, Python deps, or other non-npm dependencies

For simple TypeScript-only projects, `pnpm outdated` in `health.sh` is enough. But if the project includes Docker Compose files, Python dependencies, or other package ecosystems, use Renovate for comprehensive dependency checking.

Add `renovate.json` to project root:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"]
}
```

Add `pep621` matcher if project has Python (`pyproject.toml`):
```json
{
  "pep621": {
    "fileMatch": ["pyproject.toml"]
  }
}
```

Create `renovate-check.sh` that runs `npx -y renovate --platform=local --dry-run`, parses JSON output, and lists outdated dependencies by category (Docker, npm, Python, etc.). Script should exit with code 1 if any outdated dependencies found. Handle corner cases:

Node.js 24+ "happy eyeballs" IPv6 bug workaround:
```bash
PRELOAD_SCRIPT=$(mktemp)
echo "require('net').setDefaultAutoSelectFamily(false);" > "$PRELOAD_SCRIPT"
trap "rm -f $PRELOAD_SCRIPT" EXIT
export NODE_OPTIONS="${NODE_OPTIONS:-} --require=$PRELOAD_SCRIPT --dns-result-order=ipv4first"
```

Network error detection (exit with code 2):
```bash
OUTPUT=$(LOG_FORMAT=json LOG_LEVEL=debug npx -y renovate --platform=local --dry-run 2>&1 || true)
if echo "$OUTPUT" | grep -q '"result":"external-host-error"'; then
  echo "⚠ Renovate couldn't reach external hosts (network issue)"
  exit 2
fi
```

Update `health.sh` to call `renovate-check.sh` instead of `pnpm outdated`.

## Commits and Releases

Use conventional commits format:
- `feat:` new features
- `fix:` bug fixes
- `perf:` performance improvements
- `refactor:` code refactoring
- `docs:` documentation changes
- `chore:` maintenance tasks
- `test:` test changes
- `build:` build system changes
- `ci:` CI configuration

### If project will be published to npm

Set up release-it with conventional changelog:

```bash
pnpm add -D release-it @release-it/conventional-changelog
```

Create `.release-it.json`:
```json
{
  "git": {
    "commitMessage": "chore: release v${version}",
    "tagName": "v${version}",
    "requireCleanWorkingDir": true,
    "requireBranch": "main"
  },
  "npm": {
    "publish": true
  },
  "github": {
    "release": false
  },
  "plugins": {
    "@release-it/conventional-changelog": {
      "preset": {
        "name": "conventionalcommits",
        "types": [
          { "type": "feat", "section": "Features" },
          { "type": "fix", "section": "Bug Fixes" },
          { "type": "perf", "section": "Performance" },
          { "type": "refactor", "section": "Refactoring" },
          { "type": "docs", "section": "Documentation" },
          { "type": "chore", "hidden": true },
          { "type": "test", "hidden": true },
          { "type": "build", "hidden": true },
          { "type": "ci", "hidden": true }
        ]
      },
      "infile": "CHANGELOG.md",
      "header": "# Changelog"
    }
  },
  "hooks": {
    "before:init": ["pnpm run lint", "pnpm run typecheck", "pnpm run test"]
  }
}
```

Add scripts to `package.json`:
```json
{
  "scripts": {
    "release": "release-it",
    "release:dry": "release-it --dry-run"
  }
}
```

Create initial `CHANGELOG.md`:
```markdown
# Changelog
```

## Lessons Learned

### Biome 2.x breaking changes
* Config schema changed significantly from 1.x to 2.x
* `organizeImports.enabled` moved to `assist.actions.source.organizeImports: "on"`
* `files.ignore` replaced with `files.includes` using negation patterns: `["**", "!**/node_modules", "!**/dist"]`
* `noDeprecated` rule removed from nursery - no direct replacement in 2.x
* Always run `biome migrate --write` after updating biome to auto-fix config

### TypeScript strict mode with fetch
* `response.json()` returns `unknown` in strict mode
* Must use type assertions: `(await response.json()) as MyType`
* Or define response interfaces and cast explicitly

### Use proper clients over raw fetch
* For Trino: use `trino-client` package instead of raw HTTP calls
* Handles pagination (nextUri), connection management, retries
* Same applies to other services - prefer official/well-maintained clients

### Docker Compose healthchecks
* Use `service_healthy` condition for dependencies
* Some services need longer `retries` (Trino, Debezium can take 20+ retries)
* For Trino healthcheck: `trino --execute 'SELECT 1'`
* For Kafka: `kafka-broker-api-versions --bootstrap-server localhost:9092`

### Nessie with PostgreSQL backend
* Nessie can use PostgreSQL for metadata storage (instead of RocksDB)
* Create separate schema: `CREATE SCHEMA IF NOT EXISTS nessie`
* Configure via: `QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://host:5432/db?currentSchema=nessie`
### Trino Iceberg connector with Nessie (Trino 468+)
* Property names changed: `iceberg.nessie.*` → `iceberg.nessie-catalog.*`
* Nessie API v2: use `http://nessie:19120/api/v2` (not v1)
* Required properties:
  ```properties
  connector.name=iceberg
  iceberg.catalog.type=nessie
  iceberg.nessie-catalog.uri=http://nessie:19120/api/v2
  iceberg.nessie-catalog.ref=main
  iceberg.nessie-catalog.default-warehouse-dir=s3://warehouse/
  ```

### Zookeeper healthcheck
* Default `ruok` command disabled in recent versions
* Enable via: `KAFKA_OPTS: "-Dzookeeper.4lw.commands.whitelist=ruok,stat"`
* Healthcheck: `echo ruok | nc localhost 2181 | grep imok`

### Apache Iceberg Kafka Connect
* Official connector donated by Tabular to Apache Iceberg
* **No pre-built uber-jar available** - Maven Central JAR is thin (no dependencies)
* Must build from source to get all dependencies bundled:
  ```dockerfile
  # Multi-stage build from source
  FROM eclipse-temurin:17-jdk AS builder
  ARG ICEBERG_VERSION=1.7.1
  RUN apt-get update && apt-get install -y git && \
      git clone --depth 1 --branch apache-iceberg-${ICEBERG_VERSION} \
      https://github.com/apache/iceberg.git /iceberg
  WORKDIR /iceberg
  RUN ./gradlew :iceberg-kafka-connect:iceberg-kafka-connect-runtime:shadowJar \
      -x test -x integrationTest

  FROM confluentinc/cp-kafka-connect:8.1.1
  COPY --from=builder /iceberg/kafka-connect/kafka-connect-runtime/build/libs/iceberg-kafka-connect-runtime-*-all.jar \
      /usr/share/confluent-hub-components/iceberg-kafka-connect/iceberg-kafka-connect-runtime.jar
  ```
* Build takes ~10-15 minutes (downloads Iceberg source + Gradle build)
* Connector class: `org.apache.iceberg.connect.IcebergSinkConnector`
* Use `DebeziumTransform` SMT for CDC: `org.apache.iceberg.connect.transforms.DebeziumTransform`

### Renovate output formatting
* Raw JSON output is hard to read
* Parse `branchesInformation` from output for clean display
* Color-code by update type: 🔴 major, 🟡 minor, 🟢 patch
* Show table with dep name, current version → new version

### Docker Compose file organization
* Keep docker-compose.yml in `compose/` folder for cleaner root
* Update package.json scripts: `docker compose -f compose/docker-compose.yml`
* Update relative paths inside compose file (remove `./compose/` prefix)

### KRaft Kafka (Zookeeper-less)
* Confluent 8.x supports KRaft mode - no Zookeeper required
* Kafka runs as both broker and controller in single node:
  ```yaml
  KAFKA_PROCESS_ROLES: broker,controller
  KAFKA_NODE_ID: 1
  KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
  CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk  # Generate with: kafka-storage random-uuid
  ```
* Listener configuration for KRaft:
  ```yaml
  KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
  KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
  KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
  ```
* Remove Zookeeper service and related volumes (zookeeper-data, zookeeper-logs)
* Remove `KAFKA_ZOOKEEPER_CONNECT` environment variable