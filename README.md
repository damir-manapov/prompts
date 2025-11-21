# Prompts

# Creting project

I need new project, project root should be in <-->

Project is about <-->

Author is <-->, licence is MIT

My stack is typescript, pnpm, vitest, tsx, eslint, prettier, gitleaks

Make sure project uses last verions of dependencies.

## Lint, format, check

Make sure eslint rules and tsconfig are strict.

Tests should also be checked by tsconfig, lint and formatting.

Make sure lint output errors if
* deprections used in code
* "any" used

Use last versioni of eslint and last file config format.

I need 2-spaced across project files.

Make sure there is no vulnarabilities.

At the root of project create scripts:

* check.sh - runs formatting (fixing issues), check lint, check build (without emitting), checks gitleaks (including git), run tests
* health.sh - check dependencies used have up-to date versions, that there is no vulnaralbilities. If any outdated dep or vulnarability found script should fail
* all-checks.sh - runs both scripts

If you add instructions to install some software don't use project root for that and don't forget to add instructions on deleting temporary files if any.

If you are writing bash/sh scripts keep them simple, no fancy formatting. Make sure output is informative. Make sure such scripts fail fast.

Dont forget about gitignore, readme

Run all-checks.sh, make sure it passed without errors.

in package.json dependencies should be above devDependencies, author and license in upper part of file.

## I you need docker compose
* Use `docker compose` instead of `docker-compose`
* Make commands in package.json with compose prefix (up, down, restart, reset)
* On reset dont forget to delete volumes and orphans
* Don't put version to compose file, its obsolete
* Use last verions of images, but pin them vy verion tags
