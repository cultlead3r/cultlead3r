# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub profile statistics generator that creates dynamic SVG badges displaying GitHub activity metrics. The project consists of Python scripts that query the GitHub GraphQL and REST APIs to collect statistics, then generate SVG images that can be embedded in GitHub profile READMEs.

## Architecture

### Core Components

**github_stats.py** - Statistics collection module
- `Queries` class: Handles all GitHub API interactions (both GraphQL v4 and REST v3)
  - Uses asyncio with semaphore-based rate limiting (max 10 concurrent connections)
  - GraphQL queries paginate through repositories (100 per page)
  - REST API queries have retry logic for 202 status codes (up to 60 retries with 2s delays)
  - Falls back to synchronous `requests` if `aiohttp` fails
- `Stats` class: Aggregates and computes statistics
  - Lazy property evaluation pattern - properties fetch data on first access
  - Tracks: stargazers, forks, total contributions, languages, repos, lines changed, views
  - Language statistics computed by occurrence frequency across repositories
  - Supports filtering: excluded repositories, excluded languages, forked repos

**generate_images.py** - SVG generation module
- Reads SVG templates from `templates/` directory
- Uses regex substitution to inject dynamic data into templates
- Generates two badges: `overview.svg` (summary stats) and `languages.svg` (language breakdown)
- Outputs to `generated/` directory

### Data Flow

1. GitHub Actions workflow (`main.yml`) triggers on push to master or daily schedule
2. Environment variables (ACCESS_TOKEN, EXCLUDED, EXCLUDED_LANGS, EXCLUDE_FORKED_REPOS) configure execution
3. `generate_images.py` instantiates `Stats` with user configuration
4. `Stats` queries GitHub APIs asynchronously to gather data
5. SVG templates are populated with collected statistics
6. Generated SVGs are committed back to repository

### Key Design Patterns

- **Async/await throughout**: All API calls use asyncio for concurrent execution
- **Lazy loading**: Stats properties only fetch data when accessed
- **Template rendering**: SVG templates use `{{ variable }}` placeholder syntax
- **Error resilience**: Multiple fallback mechanisms (aiohttp → requests, retry logic)

## Development Commands

### Running Locally

```bash
# Install dependencies
pip install -r requirements.txt

# Set required environment variables
export ACCESS_TOKEN="your_github_token"
export GITHUB_ACTOR="your_username"

# Optional: Configure filtering
export EXCLUDED="repo1,repo2"
export EXCLUDED_LANGS="HTML,CSS"
export EXCLUDE_FORKED_REPOS="true"

# Generate statistics (can test either module)
python3 generate_images.py
python3 github_stats.py  # For testing stats collection only
```

### Testing

No automated test suite exists. Manual testing involves:
1. Running scripts with different environment variable configurations
2. Inspecting generated SVG files in `generated/` directory
3. Verifying SVG rendering in browser or GitHub preview

### GitHub Actions

The workflow runs automatically but can be triggered manually:
```bash
# Trigger workflow via GitHub CLI
gh workflow run "Generate Stats Images"
```

## Important Considerations

### API Rate Limiting and Error Handling
- GitHub GraphQL API has different rate limits than REST API
- The semaphore limits concurrent connections to 10

#### Why Some Repos Cause Errors
The `/repos/{owner}/{repo}/stats/contributors` endpoint has unique caching behavior that causes specific repos to fail:

**202 Status Code (Accepted)**: Statistics are being computed in background
- GitHub computes contributor stats on-demand and caches them by default branch SHA
- Returns 202 until computation completes (requires retrying)
- **Repos that haven't been accessed recently** have expired/missing cache → always return 202
- **Large repos with many commits** take longer to compute → may exceed 60 retry limit (2 minutes)
- **Example**: `cultlead3r/rucode` consistently hits retry limit, suggesting either very large history or stale cache

**204 Status Code (No Content)**: No data available
- Returned for **empty repositories** (no commits)
- Returned for **repositories with only empty commits** (no actual changes)
- Returned for **repos where you're not a contributor** (common in org repos you only have read access to)
- **Example**: `IB5k/teleprompter` and `IB5k/soapbox_scrapers` return 204, indicating you may not be a direct contributor to these repos

**Why the Error Handling Was Broken**:
- Old code tried to parse JSON from 204 responses (which have no body) → aiohttp exception
- Exception handler would retry 60 times using requests library
- Each retry also failed with 204 → resulted in 60+ error messages per repo
- **Fixed**: Now detects 204 immediately and returns empty dict without retrying

### Environment Variables
- `ACCESS_TOKEN`: GitHub personal access token (required, must have repo access)
- `GITHUB_ACTOR`: GitHub username to analyze (required)
- `EXCLUDED`: Comma-separated list of repos to exclude (optional)
- `EXCLUDED_LANGS`: Comma-separated list of languages to exclude (optional)
- `EXCLUDE_FORKED_REPOS`: Set to "true" to ignore forked repositories (optional)

### SVG Templates
Located in `templates/` directory. Use `{{ variable }}` syntax for placeholders:
- `{{ name }}` - GitHub name
- `{{ stars }}` - Total stargazers
- `{{ forks }}` - Total forks
- `{{ contributions }}` - Total contributions
- `{{ lines_changed }}` - Total lines added + deleted
- `{{ views }}` - Page views (last 14 days only)
- `{{ repos }}` - Repository count
- `{{ progress }}` - Language progress bar HTML
- `{{ lang_list }}` - Language list HTML

### Known Limitations and Issues
- **View counts only reflect the last 14 days** (GitHub API limitation)
- **Language statistics are based on occurrence frequency, not actual lines of code per language**
  - A repo with Python is weighted equally whether it has 100 or 10,000 lines
  - The `prop` field represents percentage of repositories using each language, not code volume
- **Malformed API responses are silently skipped** (see github_stats.py:494-496)
- **Branch mismatch**: Workflow is configured for `master` branch but repository uses `main`
  - The workflow won't trigger on push events (would need to change line 5 in main.yml)
  - Currently only runs on schedule (daily at 00:05 UTC) or manual dispatch
- **Error handling produces verbose logs**: Every 202/204 error prints to output, cluttering GitHub Actions logs
- **No graceful degradation**: Repositories that hit retry limits have incomplete stats but no visual indication in SVGs
