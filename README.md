# Prompt Collection

Central index of reusable LLM prompts kept in this directory. Each prompt lives in its own markdown file to keep them composable and versionable.

## Available Prompts

- `newProject.md` – Scaffold a new TypeScript project (dependencies, scripts, linting, security, compose notes).
- `repoSummaryTelegram.md` – Generate a concise Russian summary of a GitHub repository for posting to a Telegram channel (English prompt text, Russian output, no emoji, includes link at end).

## Usage Pattern
1. Open the desired prompt file.
2. Replace placeholders (e.g. `{{REPO_URL}}`, project description slots).
3. Copy the block indicated (usually between `BEGIN`/`END`).
4. Paste into your chosen LLM interface.
5. Review output for accuracy before publishing or using.

## Adding New Prompts
When adding a new prompt:
* Keep scope focused (one clear task per file).
* Use placeholders like `{{VARIABLE_NAME}}` for required inputs.
* Indicate explicit output constraints (length, style, language, forbidden items).
* Avoid duplicating logic already present in another prompt; instead reference it.

## Conventions
* File names: `kebabCase` or descriptive phrases; keep `.md` extension.
* No embedded proprietary secrets or tokens.
* Prompts must state if they require external repository/code inspection or operate on provided text only.
* Specify any forbidden elements (e.g. no hashtags, no emoji) explicitly.

## Roadmap Ideas
Potential future prompt files:
- `multiRepoDigest.md` – Summaries across several repos in one post.
- `releaseAnnouncement.md` – Changelog-driven release note synthesis.
- `issueTriager.md` – Classify and prioritize incoming issues.

---

For detailed project scaffolding instructions see `newProject.md`. For Telegram-ready repository summaries see `repoSummaryTelegram.md`.
