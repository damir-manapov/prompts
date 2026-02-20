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

## If project is a pnpm monorepo

Use `pnpm-workspace.yaml` with `packages: ['packages/*']`.

Each package gets its own `tsconfig.json` extending the root, with `composite: true` and `outDir: "dist"`.

Root `tsconfig.json` lists all packages in `references`:
```json
{
  "references": [
    { "path": "packages/core" },
    { "path": "packages/client" }
  ],
  "files": [],
  "include": []
}
```

### Monorepo type-checking

Tests are not part of composite project references (they depend on dev tools like vitest). Two typecheck scripts are needed:

```json
{
  "scripts": {
    "typecheck": "tsc -b --noEmit",
    "typecheck:tests": "tsc -p tsconfig.check.json"
  }
}
```

* `tsc -b --noEmit` — graph-aware type-check across all referenced projects in dependency order, without emitting `.js`, `.d.ts`, or `.tsbuildinfo`. Loses incremental cache but works as a fast CI check.
* `tsconfig.check.json` — flat config (no project references, no `composite`) with `noEmit: true` and `include` covering both `src` and `tests`:

```json
{
  "compilerOptions": {
    "noEmit": true
  },
  "include": ["packages/*/src/**/*.ts", "packages/*/tests/**/*.ts"],
  "exclude": ["node_modules", "**/dist"]
}
```

`check.sh` should run both:
```bash
echo "=== Typecheck ==="
pnpm typecheck

echo "=== Typecheck (tests) ==="
pnpm typecheck:tests
```

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

Choose between **release-it** and **changesets**:

* **release-it** — single-package repos, sole author, conventional commits drive version bumps automatically. Lightweight, manual CLI release flow with dry-run support.
* **changesets** — pnpm monorepos with multiple published packages, or team workflows where each PR includes a changeset file describing the change. Better for coordinating releases across packages.

Pick release-it unless you have a monorepo with multiple npm packages.

#### Option A: release-it

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

To release, use `--ci` without specifying an explicit increment:
```bash
pnpm exec release-it --ci
pnpm exec release-it --ci --dry-run
```

**Do not** pass an explicit increment with `--ci` (e.g., `pnpm run release -- --ci patch`). This causes release-it to pass `"--ci"` as the increment to conventional-changelog, which makes `semver.inc()` return `null`, resulting in broken changelog headers (`## []` with `vnull` in compare URLs). Let conventional commits determine the bump type automatically (`fix:` → patch, `feat:` → minor, `feat!:` / `BREAKING CHANGE` → major).

Create initial `CHANGELOG.md`:
```markdown
# Changelog
```

#### Option B: changesets

Set up changesets for monorepo or team workflows:

```bash
pnpm add -D @changesets/cli @changesets/changelog-github
pnpm changeset init
```

Update `.changeset/config.json`:
```json
{
  "$schema": "https://unpkg.com/@changesets/config@3.1.1/schema.json",
  "changelog": ["@changesets/changelog-github", { "repo": "owner/repo" }],
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch"
}
```

Add scripts to `package.json`:
```json
{
  "scripts": {
    "changeset": "changeset",
    "release": "changeset publish",
    "version": "changeset version"
  }
}
```

Workflow:
1. After making changes, run `pnpm changeset` to create a changeset file describing the change and bump type
2. When ready to release, run `pnpm version` to consume changesets and bump versions
3. Run `pnpm release` to publish to npm
4. Commit and push the version bump and changelog updates

## Lessons Learned

### Strict TypeScript flags beyond `strict: true`
Enable these in `tsconfig.json` for stricter compile-time safety:
* `noUncheckedIndexedAccess: true` — array/record indexing returns `T | undefined` instead of `T`. Forces null-checks on `obj[key]` and `arr[i]`, catches real bugs where index may be out of bounds.
* `exactOptionalPropertyTypes: true` — `prop?: string` means "may be missing" but NOT "may be `undefined`". To allow explicit `undefined`, declare as `prop?: string | undefined`. Prevents accidental `undefined` assignments.
* `verbatimModuleSyntax: true` — requires `import type { Foo }` for type-only imports. Prevents importing types as values which would create empty runtime imports.

These flags surface real bugs but require defensive coding:
```typescript
// noUncheckedIndexedAccess: must guard array access
const item = arr[i] // type is T | undefined
if (item !== undefined) { /* safe to use */ }

// exactOptionalPropertyTypes: explicit undefined needs union
interface Opts {
  label?: string           // missing OK, undefined NOT OK
  label?: string | undefined  // both missing and undefined OK
}
```

### Dual ESM/CJS npm packages
When publishing a package that may be consumed by both ESM and CJS projects (e.g., projects with `"moduleResolution": "node16"`):
* Main `tsconfig.json` builds ESM to `dist/`
* Create `tsconfig.cjs.json` extending the base, overriding `module: "CommonJS"`, `moduleResolution: "node"`, `outDir: "dist/cjs"`, `declaration: false`
* Build script must generate a `package.json` with `"type": "commonjs"` inside `dist/cjs/`, otherwise Node.js treats `.js` files there as ESM (because root `package.json` has `"type": "module"`) and CJS consumers crash with `ReferenceError: exports is not defined in ES module scope`:
  ```
  "build": "tsc && tsc -p tsconfig.cjs.json && echo '{\"type\":\"commonjs\"}' > dist/cjs/package.json"
  ```
* Package exports must include all three conditions:
  ```json
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/cjs/index.js"
    }
  }
  ```
* Order matters: `types` first, then `import`, then `require`
* Without `require` condition, CJS consumers with `node16` resolution will fail to resolve the package

### Exclude spec files from build output
* Add `"src/**/*.spec.ts"` to `exclude` in **both** `tsconfig.json` and `tsconfig.cjs.json`
* Otherwise test files leak into `dist/` and get published to npm
* The exclude must be repeated in the CJS config even if it extends the base (TypeScript does not merge `exclude` arrays)

### Gitleaks 8.22+ CLI changes
* `--source` flag removed, path is now a positional argument
* Old: `gitleaks git --source . --verbose` → New: `gitleaks git . --verbose`
* Old: `gitleaks dir --source . --verbose` → New: `gitleaks dir . --verbose`

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

### pnpm may silently bump other dependencies
* When running `pnpm add <package>`, pnpm may auto-bump other dependencies in `package.json` to newer versions
* Always check `git diff package.json` after adding a new dependency
* Revert unintended bumps with `pnpm add [-D] <package>@<original-version>`
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