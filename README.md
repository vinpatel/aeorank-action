# AEOrank GitHub Action

Scan any URL for AI Engine Optimization (AEO) score and post results as a GitHub Check Run and PR comment.

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-AEOrank-blue?logo=github)](https://github.com/marketplace/actions/aeorank-aeo-scanner)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## Quick Start

```yaml
- uses: vinpatel/aeorank-action@v1
  with:
    url: https://example.com
```

---

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `url` | URL to scan | Yes | — |
| `token` | GitHub token for API calls | No | `${{ github.token }}` |
| `fail-below` | Fail the Check if AEO score drops below this value (0 = never fail) | No | `0` |
| `fail-on-crawler-block` | Fail the PR if GPTBot is blocked. Check fails when GPTBot, ClaudeBot, PerplexityBot, or Google-Extended is disallowed in robots.txt. Missing robots.txt is **unknown**, not blocked. | No | `true` |

Public crawler statuses in `crawlerAccess` and in the Check table are **`allow`**, **`block`**, and **`unknown`**.

---

## Required Permissions

The caller workflow **must** declare these permissions:

```yaml
permissions:
  checks: write
  pull-requests: write
  contents: read
```

**Why these permissions are required:**

`GITHUB_TOKEN` is a GitHub App installation access token automatically provisioned by GitHub Actions for each workflow run. By default it has minimal permissions, but you can expand them at the job or workflow level:

- **`checks: write`** — unlocks `checks.create()` which posts the AEO score as a Check Run on the commit. Without this permission, the API call returns "Resource not accessible by integration".
- **`pull-requests: write`** — unlocks comment creation and updates on PRs. Required for the score table comment to be posted.
- **`contents: read`** — standard read access, needed for checkout (if your workflow uses it).

You do **not** need a separate GitHub App or personal access token. `GITHUB_TOKEN` is sufficient when permissions are declared correctly.

---

## Full Workflow Example

Create `.github/workflows/aeorank.yml` in your repository:

```yaml
name: AEOrank Check

on:
  push:
    branches: [main]
  pull_request:

permissions:
  checks: write
  pull-requests: write
  contents: read

jobs:
  aeorank:
    runs-on: ubuntu-latest
    steps:
      - uses: vinpatel/aeorank-action@v1
        with:
          url: https://example.com
          token: ${{ secrets.GITHUB_TOKEN }}
          fail-below: 70
          fail-on-crawler-block: true
```

Replace `https://example.com` with the URL you want to scan on every push or pull request.

---

## Test the crawler gate

`https://www.nytimes.com/robots.txt` disallows GPTBot:

```
User-agent: GPTBot
Disallow: /
```

The public status for that bot is `block`. This repository's [`.github/workflows/crawler-gate.yml`](.github/workflows/crawler-gate.yml) runs the action in-tree (`uses: ./`) against that URL with `fail-below: 0` and `fail-on-crawler-block: true`. The action step sets `continue-on-error`. The job stays green only when that gate failure names GPTBot in the Check summary or the job log. A low score is not what the assertion accepts.

The same inputs in a consumer workflow:

```yaml
name: Crawler gate (GPTBot disallowed)

on:
  workflow_dispatch:

permissions:
  checks: write
  pull-requests: write
  contents: read

jobs:
  crawler-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: vinpatel/aeorank-action@v1
        with:
          url: https://www.nytimes.com
          token: ${{ secrets.GITHUB_TOKEN }}
          fail-below: 0
          fail-on-crawler-block: true
```

---

## What It Does

**On every push and pull request:**

The action runs `npx aeorank-cli@0.1.2 scan <url> --format json --no-files`. It then posts the result as a **GitHub Check Run** attached to the commit:

- AI crawler table first (GPTBot, ClaudeBot, PerplexityBot, Google-Extended), each `allow`, `block`, or `unknown`
- Score and grade shown in the Check summary
- Full dimension breakdown table in the Check details
- Check conclusion is `success`, `neutral`, or `failure` based on the score and the crawler gate (see table below)

`aeorank-cli@0.1.2` scores **36** dimensions (`dimensionCount` and `dimensions.length`) and writes **8** files: `llms.txt`, `llms-full.txt`, `CLAUDE.md`, `schema.json`, `robots-patch.txt`, `faq-blocks.html`, `citation-anchors.html`, and `sitemap-ai.xml`. This Action passes `--no-files`, so it does not write them. `ai.txt` is scored when the site serves it and is not one of those generated files.

`--fail-on-crawler-block` is also a CLI flag. The flag exits **2** when the gate fails. Scan errors stay **1**. The Action input does not pass the flag; it reads `crawlerGate` and still posts the Check.

**On pull requests only:**

Posts (or updates) a **PR comment** with the score, crawler table, and dimension table. The comment uses a hidden `<!-- aeorank-score -->` marker so re-runs update the existing comment rather than posting new ones. Your PR thread stays clean no matter how many times the workflow runs.

**With `fail-below`:**

Set `fail-below: 70` (or any threshold from 1–100) to make the Check fail whenever the AEO score drops below that number. This lets you enforce a minimum AEO quality bar in CI — PRs that drop the score below your threshold will show a failing Check.

**With `fail-on-crawler-block` (default `true`):**

The Check fails when the public status of GPTBot, ClaudeBot, PerplexityBot, or Google-Extended is **`block`** — even if the overall score is high. The Check summary names which bot is blocked. A missing `robots.txt` is **`unknown`** and does not fail.

Set `fail-on-crawler-block: false` to keep the crawler table and skip this gate.

The CLI JSON contract (`aeorank-cli@0.1.2`):

- `crawlerAccess` emits **`allow`**, **`block`**, or **`unknown`** (for example `{ "GPTBot": "block" }`). That is the public status. `meta.robotsTxt.crawlerAccess` still uses the parser's internal **`allowed`** / **`disallowed`** / **`unknown`**.
- The Check table maps `allowed` → `allow` and `disallowed` → `block`, and shows `allow` / `block` / `unknown` unchanged.
- `crawlerGate.failed`: `true` only when a checked bot's public status is `block`
- `crawlerGate.blockedBots`: names to put in the Check summary

---

## Check Conclusion Logic

| Score Range | Crawler blocked? | `fail-below` triggered? | Conclusion | Display |
|-------------|------------------|------------------------|------------|---------|
| Any | Yes (`block`) | — | `failure` | Red X (bot named) |
| 70–100 | No | No | `success` | Green check |
| 40–69 | No | No | `neutral` | Grey dash |
| 0–39 | No | No | `failure` | Red X |
| Any | No | Yes (score < threshold) | `failure` | Red X |

The `fail-below` input overrides the normal score bands. A crawler **`block`** overrides both. **`unknown`** (including a missing robots.txt) does not.

---

## Fork PR Limitation

When a pull request is opened from a **fork**, GitHub restricts `GITHUB_TOKEN` to read-only permissions for security. This means the action cannot post Check Runs or PR comments on fork PRs — the API calls will fail with permission errors.

**Workaround:** Use `pull_request_target` instead of `pull_request` in your workflow trigger:

```yaml
on:
  pull_request_target:
```

> **Security warning:** `pull_request_target` runs in the context of the base branch (with write permissions) even for fork PRs. Only use this if you understand the [security implications](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#pull_request_target). Never checkout the fork's code and run it in a `pull_request_target` workflow without careful sandboxing.

For most projects, the simplest approach is to skip Check/comment posting on fork PRs and accept that fork contributors won't see the AEOrank score inline until their PR is merged or the branch is pushed to your repo.

---

## Caching (Optional)

The Action installs `aeorank-cli@0.1.2` (package name `aeorank-cli`) from npm on a cold runner, adding 5–15 seconds. You can reduce this with npm caching:

```yaml
jobs:
  aeorank:
    runs-on: ubuntu-latest
    steps:
      - name: Cache npm
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-npm-aeorank-cli
          restore-keys: |
            ${{ runner.os }}-npm-

      - uses: vinpatel/aeorank-action@v1
        with:
          url: https://example.com
```

---

## Links

- [aeorank.dev](https://aeorank.dev) — AEOrank homepage
- [docs.aeorank.dev](https://docs.aeorank.dev) — Full documentation
- [GitHub Marketplace](https://github.com/marketplace/actions/aeorank-aeo-scanner) — Action listing
