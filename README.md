# Prompts

# Creting project

I need new project, project root chould be in <-->

Project is about <-->

My stack is typescript, pnpm, vitest, tsx, eslint, prettier, gitleaks

Make sure project uses last verions of dependencies.

Make sure eslint rules and tsconfig are strict.

Make sure there is no vulnarabilities

At the root pf project crfeate scripts:

* check.sh - runs formatting, check lint, check build (without emitting), checks gitleaks (including git)
* health.sh - check dependencies used have are up-to date versions, that there is no vulnaralbilities

If you add instructions to install some software dont use project root for that and dont forget to add instructions on deleting temporary files if any.

If you are writing bash/sh scripts keep them simple, no fancy formatting. Make sure output is informative. Make sure such script fail fast.

Dont forget about gitignore, readme
