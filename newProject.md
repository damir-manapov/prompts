# Prompt Template: Creating New TypeScript Project

This prompt scaffolds a complete TypeScript project with strict linting, testing, and security checks.

---

PROMPT FOR LLM (replace placeholders before use):

I need new project, project root should be in {{PROJECT_ROOT}}

Project is about {{PROJECT_DESCRIPTION}}

Author is Damir Manapov, licence is MIT

My stack is TypeScript, pnpm, vitest, tsx, eslint, prettier, gitleaks

Make sure project uses latest versions of dependencies.

## Lint, format, check

Make sure eslint rules and tsconfig are strict.

Tests should also be checked by tsconfig, lint and formatting.

Make sure lint output errors if
* deprecations used in code
* "any" used

Use latest version of eslint and latest file config format.

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
* Make commands in `package.json` with compose prefix (`up`, `down`, `restart`, `reset`)
* On reset don't forget to delete volumes and orphans
* Don't put version to compose file, it's obsolete
* Use latest versions of images, but pin them by version tags
* Add starter service to compose file that waits all other services started successfully. Target it when starting docker compose. Place that service first
* Don't set `restart` and `container_name` options in compose file

## Notes
* If you are going to use `tseslint.config` for `eslint.config.js` - it is deprecated
