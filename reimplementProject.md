# Prompt Template: Reimplementing an Existing Project

This prompt scaffolds a complete monorepo that reimplements an existing system using test-first, black-box compatibility testing. The same test suite runs against both the original system and the reimplementation.

Requires: `newProject.md` conventions (biome, vitest, pnpm, gitleaks, check scripts).

---

PROMPT FOR LLM (replace placeholders before use):

Check /home/damir/prjs/prompts/newProject.md

I need new monorepo project "{{PROJECT_NAME}}" with compose.

You are a senior TypeScript backend architect and test engineer.

Your task is to generate a COMPLETE monorepo project that implements:

1) A {{ORIGINAL_SYSTEM}}-compatible server implementation
2) A comprehensive compatibility test harness
3) Shared utilities for testing
4) Test-first development workflow

The tests are the specification.

The project must allow running the SAME compatibility tests against:

A) Official {{ORIGINAL_SYSTEM}} backend
B) Our reimplementation

The implementation must gradually satisfy these tests.

The architecture must enforce strict black-box compatibility testing.

==================================================
PRIMARY GOAL
==================================================

Build a {{ORIGINAL_SYSTEM}}-compatible backend implementation using:

- Test-first design
- Compatibility tests derived from real API behaviour
- Golden baseline comparison with official {{ORIGINAL_SYSTEM}}

The compatibility harness must verify:

{{COMPATIBILITY_DIMENSIONS}}

Default dimensions (remove or extend as needed):

• HTTP contract compatibility
• JSON schema compatibility
• error behaviour compatibility
• concurrency behaviour

Additional dimensions to include if relevant:

• SSE / WebSocket streaming compatibility
• multipart file upload compatibility
• real client integration compatibility
• webhook delivery compatibility
• auth flow compatibility

==================================================
MONOREPO STRUCTURE
==================================================

Use a pnpm workspace.

Structure:

{{PROJECT_NAME}}/
  package.json
  pnpm-workspace.yaml
  tsconfig.json
  tsconfig.check.json
  biome.json
  vitest.config.ts
  README.md

  apps/
    server/
      package.json
      tsconfig.json
      src/

    compat-tests/
      package.json
      tsconfig.json
      vitest.config.ts
      src/
      tests/
      goldens/
        official/
        reimpl/

  packages/
    test-utils/
      package.json
      tsconfig.json
      src/

Do NOT create unnecessary packages.

Only these packages are allowed initially:

apps/server
apps/compat-tests
packages/test-utils

==================================================
CRITICAL ARCHITECTURE RULE
==================================================

compat-tests MUST treat server as a remote system.

Tests may NOT:

• import server code
• access server database
• call internal functions
• bypass HTTP

Tests communicate with server ONLY via HTTP.

The tests must also run against official {{ORIGINAL_SYSTEM}}.

==================================================
TEST HARNESS DESIGN
==================================================

The compatibility harness must include three layers.

----------------------------------
LAYER 1 — HTTP CONTRACT TESTS
----------------------------------

Direct HTTP calls to {{ORIGINAL_SYSTEM}} API endpoints.

Test groups (adapt to {{ORIGINAL_SYSTEM}} API):

{{TEST_GROUPS}}

Example test groups:

Ping / connectivity
Resource CRUD (list, create, get, update, delete)
Main action endpoint (e.g. prediction, execution, send)
Streaming responses (SSE, WebSocket, or chunked)
File uploads / attachments
Auth behavior
Header behavior
Concurrency
Error cases

Tests must validate:

status codes
response headers
JSON response structure
error response structure

Use Zod schemas for validation where response shapes are stable.

----------------------------------
LAYER 2 — GOLDEN BASELINE TESTS
----------------------------------

Tests must support recording baseline behaviour.

Mode controlled by env variable:

RECORD_GOLDENS=1

When enabled:
• responses from official {{ORIGINAL_SYSTEM}} are stored as goldens

When disabled:
• responses from reimplementation are compared against goldens

Golden normalization must remove unstable fields.

Common unstable fields:

ids
timestamps
token counts
generated text variability
request-specific UUIDs
session identifiers

Add project-specific unstable fields:

{{UNSTABLE_FIELDS}}

Golden storage structure:

goldens/
  official/
  reimpl/

Normalization functions must be implemented in test-utils.

----------------------------------
LAYER 3 — CLIENT INTEGRATION TESTS (optional)
----------------------------------

Only add this layer if {{ORIGINAL_SYSTEM}} has an official client SDK
and you need to verify SDK compatibility specifically.

Skip this layer by default — HTTP contract tests + golden comparison
are sufficient for most reimplementations. Serving the original UI
(see SERVING THE ORIGINAL UI) provides better visual validation
than SDK smoke tests.

==================================================
EDGE CASE COVERAGE
==================================================

Include tests for (adapt to {{ORIGINAL_SYSTEM}} API):

invalid ids
missing fields
invalid JSON
wrong content-type
empty required fields
very long inputs
unicode input
concurrent requests
client disconnect

Add project-specific edge cases:

{{EDGE_CASES}}

==================================================
TEST UTILITIES PACKAGE
==================================================

Create packages/test-utils.

This package provides reusable utilities used by compat-tests.

Required utilities:

HTTP client wrapper (JSON, multipart, raw body, headers, auth injection)
Golden recorder
Golden comparator
Normalization helpers
Temp file generator
Concurrency helpers (run N tasks in parallel, collect results)
Retry helpers
Test logger
Config loader (reads BASE_URL, AUTH_TOKEN, TARGET_NAME, RECORD_GOLDENS from env)

Include if needed:

SSE parser (if streaming via SSE)
WebSocket client (if streaming via WebSocket)

==================================================
COMPAT TESTS IMPLEMENTATION
==================================================

apps/compat-tests/tests/api/

{{TEST_FILES}}

Example test file list:

01_ping.test.ts
02_resource_crud.test.ts
03_main_action.test.ts
04_main_action_errors.test.ts
05_streaming.test.ts
06_file_uploads.test.ts
07_auth_headers.test.ts
08_concurrency.test.ts
09_regression_quirks.test.ts

apps/compat-tests/tests/integration/

01_client_smoke.test.ts

==================================================
TEST EXECUTION TARGETS
==================================================

Tests must run against two targets.

Use env variables:

BASE_URL          — server API base URL (required)
AUTH_TOKEN         — bearer token for auth (optional)
TARGET_NAME       — "official" or "reimpl"
RECORD_GOLDENS    — set to "1" to record golden baselines

Example:

BASE_URL=http://localhost:3000/api/v1

compat-tests vitest.config.ts must pass these env vars to test workers.

==================================================
SERVER IMPLEMENTATION
==================================================

Create apps/server.

Use:

TypeScript
Node.js
Fastify (preferred) or Express

The server must implement {{ORIGINAL_SYSTEM}}-compatible endpoints.

Initial endpoints:

{{ENDPOINTS}}

Implementation may initially stub business logic.

Focus on matching HTTP behaviour: status codes, headers, response shapes, error formats.

==================================================
SERVER ARCHITECTURE
==================================================

apps/server/src/

server.ts
routes/
  {{ROUTE_FILES}}

services/
  {{SERVICE_FILES}}

storage/
  inMemoryStore.ts

Add if needed:
sse/
  sseWriter.ts

==================================================
TEST-FIRST WORKFLOW
==================================================

For each endpoint or group of endpoints:

1. Stub — return `[]` or static data so the UI doesn't break
2. Write compat tests for the endpoint contract
3. Run tests against the original {{ORIGINAL_SYSTEM}} — confirm they pass
4. Implement the real handler in our server
5. Run the same tests against our server — fix until green

Stubs come first so the original UI can load against our backend.
Tests are written before implementation to avoid shaping tests to match bugs.

==================================================
IMPLEMENTATION PLAN
==================================================

After traffic recording and endpoint discovery, create `apps/compat-tests/PLAN.md`.

The plan should:

- List all discovered endpoints grouped into implementation steps
- Start with boot-time stubs (Step 1) — endpoints the UI calls on every page load
- Follow with CRUD groups ordered by dependency and complexity
- Track status per endpoint (done / stub / not started)
- Reference the test scripts (`test:official`, `test:reimpl`, `test:record`)
- Describe the per-step cycle explicitly

Step 1 (boot stubs) only executes cycle steps 1–3.
Later steps upgrade stubs to real implementations (cycle steps 4–5).

Put simple CRUDs before complex endpoints (e.g., a dynamic node catalog
is harder than variables CRUD — do variables first).

==================================================
SERVING THE ORIGINAL UI
==================================================

If {{ORIGINAL_SYSTEM}} has a web UI, serve it against our backend for visual validation.

Use a reverse proxy (Caddy) — do NOT add static file serving to the server code.

Architecture:

```
Browser → Caddy (:3000)
  /api/* → our server (:3000 internal)
  /*     → UI static files (extracted from original image)
```

Use Docker Compose profiles so the UI service is optional:

```yaml
services:
  ui-copy:
    image: {{ORIGINAL_IMAGE}}
    profiles: [ui]
    # extract static files to a shared volume
  caddy:
    image: caddy:alpine
    profiles: [ui]
    # serve UI + proxy /api/* to our server
```

This keeps the server pure API, while letting you visually verify
that the UI works against our reimplementation.

==================================================
CODE QUALITY
==================================================

Code must be production-grade.

Requirements:

clear module boundaries
clean TypeScript types
good logging
good error handling
use async/await
strong test failure diagnostics

==================================================
README REQUIREMENTS
==================================================

Generate a README explaining:

project structure
how to install
how to run tests
how to record goldens
how to run against official {{ORIGINAL_SYSTEM}}
how to run against reimplementation
test architecture explanation (two layers: HTTP contract + golden comparison)
how to add new compatibility tests

Do not mention author or license in README.

==================================================
OUTPUT FORMAT
==================================================

Output full project.

For each file:

Provide file path
Provide code block

Example:

/apps/server/src/server.ts
```ts
code here
```

Do this for all files.

==================================================
QUALITY BAR
==================================================

The result must look like it was designed by a senior engineer building a long-term compatibility testing platform.

Avoid toy examples.

Focus on maintainability and extensibility.

---

## API Contract Discovery via Traffic Recording

When the original system has a UI (web app, CLI, SDK) that exercises the backend, you can discover the real API contract by recording traffic between the client and the original backend.

### Architecture

```
Client (browser/you) → Reverse proxy (serves UI) → mitmproxy (records) → Original backend
```

### Setup

Use Docker Compose with a `record` profile or a separate `docker-compose.record.yml`:

- **Original backend**: official image running normally
- **mitmproxy**: `mitmproxy/mitmproxy` image in reverse proxy mode, forwarding to the backend
- **Reverse proxy (Caddy/nginx)**: serves UI static files, routes API traffic through mitmproxy
- **Captures volume**: mitmproxy addon writes JSONL to a mounted directory

### mitmproxy addon

Write a small Python addon (~30 lines) that:
- Intercepts each completed request/response pair
- Skips static assets (`.js`, `.css`, `.ico`, `.png`, `.svg`, `.woff`, etc.)
- Writes one JSON line per exchange to a session file
- Captures: method, path, query, status, request/response headers, request/response bodies, content type, duration, timestamp
- For SSE responses: captures the full event stream text

Store addon in `compose/mitmproxy-addon.py`.

### Capture storage

Store raw captures in `apps/compat-tests/captures/` (gitignored):
```
apps/compat-tests/captures/
  session-20260307-171500.jsonl   # one line per request/response pair
```

Use JSONL format — simple, appendable, easy to process with `jq` or TypeScript.

### Workflow

```bash
# Start original system + mitmproxy + UI
pnpm compose:record:up

# Open the UI in your browser, click through key scenarios:
# - page load (discover boot-time API calls)
# - CRUD operations
# - streaming/SSE actions
# - file uploads

# Stop recording
pnpm compose:record:down

# Analyze captures with jq
jq -r '[.method, .path, .status] | @tsv' captures/session-*.jsonl | sort | uniq -c | sort -rn
```

### Boot endpoint discovery

The first recording session is the most important. Just load the UI — don't click anything.

The captured requests reveal which endpoints the UI calls on every page load.
These are your Phase 1 stubs. Without them, the UI shows errors or blank screens.

Common boot endpoints:
- Node/component catalog (usually the largest response)
- Resource lists (return `[]` initially)
- Config/settings
- Auth/API key checks

Stub all of them first, then visually verify the UI loads cleanly.

### From captures to implementation plan

1. **Record boot traffic** — load the UI once, identify all boot-time endpoints
2. **Record user actions** — click through CRUD, streaming, uploads
3. **Create PLAN.md** — list all endpoints, group into implementation steps
4. **Stub boot endpoints** — return `[]` or static data for each
5. **Write compat tests** — verified against the original before implementing
6. **Implement one step at a time** — upgrade stubs to real handlers, verify tests pass

### Key principles

- **Manual recording, not automation** — clicking through the UI is faster and more reliable than fragile Playwright selectors. Automate later when the routes are stable.
- **Use existing tools** — mitmproxy for recording, `jq` for analysis. Don't build custom proxies.
- **Response shape, not data** — tests should verify field names, types, and structure. Exact data values differ between instances.
- **Iterate** — record → analyze → implement one endpoint → test → repeat.

---

## Placeholder Reference

| Placeholder | Description | Example |
|---|---|---|
| `{{PROJECT_NAME}}` | Name of the new project | `my-reimpl` |
| `{{ORIGINAL_SYSTEM}}` | System being reimplemented | `SomeAPI` |
| `{{COMPATIBILITY_DIMENSIONS}}` | What the harness must verify | HTTP contract, JSON schema, error behaviour |
| `{{TEST_GROUPS}}` | Test categories matching the original API | Ping, Resources CRUD, Main Action, Errors |
| `{{UNSTABLE_FIELDS}}` | Fields to normalize in goldens | `requestId`, `createdAt`, `sessionId` |
| `{{ORIGINAL_CLIENT}}` | Real client SDK for integration tests (optional) | `some-api-sdk` |
| `{{ORIGINAL_IMAGE}}` | Docker image of the original system | `someorg/someapi:1.0` |
| `{{EDGE_CASES}}` | Project-specific edge cases to test | concurrent writes, large payload |
| `{{TEST_FILES}}` | Numbered test file list | `01_ping.test.ts`, `02_resources_crud.test.ts`, ... |
| `{{ENDPOINTS}}` | API endpoints to implement | `GET /ping`, `POST /resources`, `GET /resources/:id` |
| `{{ROUTE_FILES}}` | Route module filenames | `ping.ts`, `resources.ts` |
| `{{SERVICE_FILES}}` | Service module filenames | `resourceService.ts` |
