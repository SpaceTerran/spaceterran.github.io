---
title: "TerraScan: Self-Hosted AI Code Reviews for Gitea"
author: isaac
date: 2026-01-30 07:00:00 -0700
categories: [DevOps, Automation, HomeLab, Tutorials and Guides]
tags: [Gitea, Code Review, AI, Docker, CI/CD, OpenAI, Self-Hosted, Homelab]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/terrascan/terrascan.png
  alt: "TerraScan - AI-Powered Code Review for Gitea"
---

If you've ever looked at AI code review tools like CodeRabbit or Codacy, you've probably noticed a pattern: GitHub and GitLab get all the love. If you're running Gitea in your homelab, you're mostly out of luck.

That's why I built **TerraScan** — a self-hosted AI code reviewer that works natively with Gitea Actions. It runs as a Docker container, talks to OpenAI (or Anthropic, or even local Ollama models), and posts inline comments directly on your pull requests.

No third-party service. No sending your code to someone else's servers (unless you count the AI API). Just a container you control.

## What TerraScan Does

- Automatically reviews PRs when they're opened or updated
- Posts a summary comment plus inline comments on specific lines
- Catches bugs, security issues, and code quality problems
- Supports OpenAI, Anthropic (Claude), and Ollama for local models
- Configurable severity thresholds — block only on critical issues, or run purely advisory

## Prerequisites

Before you start, you'll need:

- **Gitea** with Actions enabled (Gitea 1.25.4 (tested); 1.19+ with Actions enabled recommended)
- **A Gitea runner** with Docker access
- **An OpenAI API key** (or Anthropic key, or local Ollama instance)
- **A Gitea token** with repository write access (for posting comments)

>**Note:** This guide uses OpenAI as the default provider. If you want to use Anthropic or Ollama, the setup is similar — just swap the API key environment variable.
{: .prompt-info }

## Step 1: Understanding How It Works

Here's the flow when a PR is opened:

```text
PR Created
    │
    ▼
Gitea Actions triggers workflow
    │
    ▼
Workflow generates diff (git diff origin/main...HEAD)
    │
    ▼
Diff is piped to TerraScan container
    │
    ▼
Container sends diff to AI (OpenAI/Anthropic/Ollama)
    │
    ▼
AI returns structured review
    │
    ▼
Container posts comments via Gitea API
```

The container is stateless — all configuration comes from environment variables and the built-in config file.

## Step 2: Setting Up Organization Secrets

Set these secrets at your Gitea organization level so all repos can use them:

1. Go to your organization settings: `https://your-gitea/org/YOUR-ORG/settings/actions/secrets`

>**Customize this URL:** Replace `your-gitea` with your Gitea instance domain and `YOUR-ORG` with your organization name.
{: .prompt-info }

2. Add these secrets:

| Secret | Description |
|--------|-------------|
| `OPENAI_API_CODEREVIEW_KEY` | Your OpenAI API key |
| `GTOKEN` | Gitea personal access token with `repo:write` scope |

>**Tip:** Setting secrets at the org level means you configure once and all repositories in that org can use them. You can also set per-repo secrets if needed.
{: .prompt-tip }

## Step 3: Creating the Workflow File

Add this job to your repository's workflow file at `.gitea/workflows/CICD.yml`:

>**Important:** The workflow below uses placeholder values for the Gitea registry URL (`your-gitea-instance.com`) and container path (`your-org/terrascan`). Replace these with your own Gitea instance URL and the org/repo where you're hosting the TerraScan container image.
{: .prompt-warning }

```yaml
  AIReview:
    name: AI Code Review
    # Only run on pull requests
    if: gitea.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: https://gitea.com/actions/checkout@v6
        with:
          # Need full history for proper diff
          fetch-depth: 0

      - name: Run AI Code Review
        run: |
          BASE_REF="${{ gitea.event.pull_request.base.ref }}"
          git fetch origin "$BASE_REF"

          # Three-dot diff shows only changes in PR branch
          # Falls back to two-dot if branches have diverged
          if ! DIFF_CONTENT=$(git diff "origin/$BASE_REF"...HEAD 2>/dev/null); then
            DIFF_CONTENT=$(git diff "origin/$BASE_REF"..HEAD 2>/dev/null) || exit 0
          fi

          # Skip if no changes
          [ -z "$DIFF_CONTENT" ] && echo "No changes to review" && exit 0

          # Pull and run the TerraScan container
          # Replace 'your-gitea-instance.com' and 'your-org' with your values
          echo "${{ secrets.GTOKEN }}" | docker login your-gitea-instance.com -u ${{ gitea.actor }} --password-stdin
          docker pull your-gitea-instance.com/your-org/terrascan:latest

          echo "$DIFF_CONTENT" | docker run --rm -i \
            -e OPENAI_API_KEY=${{ secrets.OPENAI_API_CODEREVIEW_KEY }} \
            -e GITEA_TOKEN=${{ secrets.GTOKEN }} \
            -e REPO_NAME=${{ gitea.repository }} \
            -e PR_NUMBER=${{ gitea.event.pull_request.number }} \
            your-gitea-instance.com/your-org/terrascan:latest
```


### Making It Play Nice With Other Jobs

Here's a common scenario: you have a workflow with multiple jobs (Lint, Security, AIReview), and a final job that depends on all of them completing. The problem is that AIReview only runs on pull requests — so when you manually trigger the workflow via `workflow_dispatch`, AIReview gets skipped. By default, any job that lists AIReview in its `needs:` will also be skipped, even though Lint and Security completed successfully.

To fix this, use a condition that allows the downstream job to run when AIReview is skipped:

```yaml
  CheckMode:
    needs: [Lint, Security, AIReview]
    if: |
      !cancelled() && !failure() &&
      (gitea.event_name == 'pull_request' || gitea.event_name == 'workflow_dispatch')
```

This tells the job: "Run if nothing was cancelled or failed, and we're either on a PR (where AIReview ran) or a manual dispatch (where it was intentionally skipped)."

## Key Configuration Options

The container has sensible defaults, but here are the key environment variables:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | Yes* | — | OpenAI API key |
| `ANTHROPIC_API_KEY` | Yes* | — | Alternative: Anthropic/Claude API key |
| `GITEA_TOKEN` | Yes | — | Token with repo write access |
| `REPO_NAME` | Yes | — | Set via `${{ gitea.repository }}` |
| `PR_NUMBER` | Yes | — | Set via `${{ gitea.event.pull_request.number }}` |

*Provide **one** of `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`, not both (unless using Ollama, then neither is required).

The built-in config (`config/review-config.yml`) controls:
- AI model selection
- Severity thresholds for blocking PRs
- File patterns to ignore
- Maximum comments per PR

## Verifying It Works

1. Create a new branch with some code changes
2. Open a pull request
3. Watch the Actions tab — you should see the "AI Code Review" job run
4. Once complete, check your PR for:
   - A summary comment at the top
   - Inline comments on specific lines (if issues were found)

Example summary comment:

```text
🤖 AI Code Review

## Summary
Reviewed 3 files with 47 lines changed.

### Issues Found
- 🟡 1 warning
- 🔵 2 suggestions

---
**Severity Guide:** 🔴 Critical | 🟠 Error | 🟡 Warning | 🔵 Info
```

## Closing Thoughts

TerraScan fills a gap I kept running into: wanting AI-assisted code review without sending code to a third-party service, and without being locked into GitHub or GitLab.

If you're running Gitea in your homelab, this gives you the same "open a PR, get feedback" workflow that commercial tools provide — just with infrastructure you control.

The project template is available on GitHub. Fork it, host it on your own Gitea instance, and customize to your needs. Drop a comment if you have questions about getting it running.

- **Source:** [github.com/SpaceTerran/TerraScan](https://github.com/SpaceTerran/TerraScan)

<br><br>

<small>
**Note:** OpenAI and GPT are trademarks of OpenAI. Anthropic and Claude are trademarks of Anthropic. Gitea is a trademark of Gitea Limited. CodeRabbit and Codacy are trademarks of their respective owners. This project is not affiliated with or endorsed by any of these companies.
</small>
