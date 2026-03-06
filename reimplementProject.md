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
LAYER 3 — CLIENT INTEGRATION TESTS
----------------------------------

Add integration tests using a real client of {{ORIGINAL_SYSTEM}}.

Primary candidate:

{{OFFICIAL_CLIENT}}

These tests verify that a real client can communicate with the backend.

These are smoke tests.

Raw HTTP tests remain the primary compatibility oracle.

If no official client exists, skip this layer.

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

1. Write tests describing expected API behavior
2. Run tests against official {{ORIGINAL_SYSTEM}}
3. Record goldens
4. Implement server
5. Run tests against reimplementation
6. Fix server until tests pass

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
test architecture explanation (three layers)
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

## Placeholder Reference

| Placeholder | Description | Example |
|---|---|---|
| `{{PROJECT_NAME}}` | Name of the new project | `my-reimpl` |
| `{{ORIGINAL_SYSTEM}}` | System being reimplemented | `SomeAPI` |
| `{{COMPATIBILITY_DIMENSIONS}}` | What the harness must verify | HTTP contract, JSON schema, error behaviour |
| `{{TEST_GROUPS}}` | Test categories matching the original API | Ping, Resources CRUD, Main Action, Errors |
| `{{UNSTABLE_FIELDS}}` | Fields to normalize in goldens | `requestId`, `createdAt`, `sessionId` |
| `{{OFFICIAL_CLIENT}}` | Real client library for integration tests | `some-api-sdk` |
| `{{EDGE_CASES}}` | Project-specific edge cases to test | concurrent writes, large payload |
| `{{TEST_FILES}}` | Numbered test file list | `01_ping.test.ts`, `02_resources_crud.test.ts`, ... |
| `{{ENDPOINTS}}` | API endpoints to implement | `GET /ping`, `POST /resources`, `GET /resources/:id` |
| `{{ROUTE_FILES}}` | Route module filenames | `ping.ts`, `resources.ts` |
| `{{SERVICE_FILES}}` | Service module filenames | `resourceService.ts` |
