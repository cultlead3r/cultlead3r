# AGENTS.md

## Project shape
- This repo generates GitHub profile SVG stats, not a web app: `generate_images.py` is the main entrypoint, `github_stats.py` does GitHub GraphQL/REST collection, `templates/*.svg` are rendered into `generated/*.svg`.
- `Stats` uses async lazy properties; when adding metrics, expect API work to happen on first `await s.<property>` access, not at construction.
- Language percentages are occurrence-based across repos (`occurrences`), not code-size percentages, even though GraphQL also returns language byte sizes.

## Local run / verification
- Required env for live runs: `ACCESS_TOKEN` and `GITHUB_ACTOR`; optional filters are `EXCLUDED`, `EXCLUDED_LANGS`, `EXCLUDE_FORKED_REPOS`.
- Minimal live verification: `python3 generate_images.py` then inspect `generated/overview.svg` and `generated/languages.svg`.
- Stats-only smoke run: `python3 github_stats.py` prints the aggregated stats and still requires live GitHub API env.
- There is no test runner or lint/typecheck config in this repo; avoid inventing `pytest`, `ruff`, or `mypy` checks unless you add their config.

## CI / runtime assumptions
- GitHub Actions uses Python 3.8 and installs only `requests` and `aiohttp` from `requirements.txt`.
- `.github/workflows/main.yml` runs on pushes to `main`, daily at `00:05 UTC`, and manual dispatch; it commits regenerated SVGs back to the repo.
- The workflow sets `EXCLUDE_FORKED_REPOS: true` and uses `secrets.ACCESS_TOKEN`; `GITHUB_TOKEN` is present but `generate_images.py` currently requires `ACCESS_TOKEN`.

## API and generated-output gotchas
- REST `/repos/{owner}/{repo}/stats/contributors` can return `202` while GitHub computes stats; code retries up to 60 times with 2s sleeps, so full live runs can be slow.
- REST contributor stats may return `204`; keep handling it as empty data, not JSON.
- Traffic views are only for the last 14 days because that is GitHub's API limit.
- Template placeholders use literal `{{ name }}` style substitutions via `re.sub`; update both templates and render code together when adding/removing fields.
- `generated/*.svg` are build artifacts but intentionally tracked because README embeds them.

## Existing notes
- `CLAUDE.md` has longer architecture notes; verify claims against current code/workflow before copying them because this repo is small and workflow details can drift.
