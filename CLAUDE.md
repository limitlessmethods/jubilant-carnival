# CLAUDE.md

This file provides guidance for AI assistants working in this repository.

## Project Overview

**jubilant-carnival** is a social media automation system built on [n8n](https://n8n.io/), an open-source workflow automation platform. It automates the distribution of short-form video content across eight social media platforms using AI-powered caption optimization.

### Target Platforms

TikTok, Instagram, YouTube, Threads, LinkedIn, X/Twitter, Facebook, Pinterest

### Key Integrations

- **Notion** — Content management database and workflow state store
- **Anthropic Claude API** — AI-driven caption optimization per platform
- **Blotato** — Cross-platform media upload and posting
- **Slack** — Failure alerting via webhooks

## Repository Structure

```
/
├── README.md              # Project description
├── CLAUDE.md              # AI assistant guidance (this file)
├── main-workflow.json     # Primary n8n auto-poster workflow (43 nodes)
└── retry-workflow.json    # Retry handler workflow with exponential backoff (30 nodes)
```

> **Note:** The workflow JSON files may live on a separate feature branch (e.g., `claude/n8n-social-media-workflow-*`). Check `git branch -a` if they are not present on the current branch.

## Architecture

### main-workflow.json — "Pro Short-Form Auto-Poster v2"

Runs on a schedule (9:00 AM and 4:00 PM UTC) and performs:

1. **Query Notion** — Fetches content items with status "Ready To Post" and platform "Short Form"
2. **Processing lock** — Sets `processing_started` timestamp to prevent duplicate runs
3. **Content validation** — Checks media files, captions, YouTube titles, and platform-specific character limits
4. **Claude AI enrichment** — Calls Claude Sonnet to optimize captions per platform (model: `claude-sonnet-4-20250514`, max tokens: 4096)
5. **Blotato upload** — Uploads media to CDN
6. **Sequential platform posting** — Posts to all 8 platforms with 30-second delays between each to avoid rate limiting
7. **Result logging** — Logs per-platform results to a Notion error log database
8. **Status aggregation** — Calculates overall status (Posted / Failed / Partial)
9. **Slack notification** — Alerts on failures

### retry-workflow.json — Retry Handler

Runs every 30 minutes and performs:

1. Queries the Notion error log for failed posts with `retry_count < 3`
2. Re-uploads media if the original upload failed
3. Routes to the correct platform via a switch node
4. Re-attempts posting with exponential backoff (30 min → 2 hrs → 8 hrs)
5. Updates the error log and recalculates master page status
6. Sends Slack alert after 3 permanent failures

### Key Architectural Patterns

| Pattern | Description |
|---|---|
| FIFO queue with lock | Notion `processing_started` field prevents race conditions |
| Validation gate | Required fields and character limits checked before posting |
| Sequential posting with delays | 30-second waits prevent API rate limiting |
| AI caption optimization | Platform-specific prompts with brand voice personality |
| Result aggregation | Per-platform results merged into overall status |
| Exponential backoff retries | Up to 3 retries with increasing intervals |
| Notion as state store | Single source of truth for content status and error tracking |

## Configuration & Credentials

The workflow files contain placeholder values that must be configured in your n8n instance before running:

| Placeholder | Purpose |
|---|---|
| `[YOUR_ANTHROPIC_API_KEY]` | Anthropic API key for Claude caption generation |
| `[YOUR_NOTION_CRED_ID]` | Notion API credential ID in n8n |
| `[YOUR_NOTION_CRED_NAME]` | Notion API credential name in n8n |
| `[YOUR_CONTENT_DB_ID]` | Notion content database ID |
| `[YOUR_ERROR_LOG_DB_ID]` | Notion error log database ID |

Slack webhook URLs and Blotato credentials must also be configured within the n8n instance.

**Never commit actual credentials or API keys to this repository.**

## Notion Database Schema

### Content Database

| Property | Type | Purpose |
|---|---|---|
| Platform | Select | Must be "Short Form" |
| Status | Select | Must be "Ready To Post" to be picked up |
| processing_started | Date | Lock field set during processing |
| Media File | File | Video/image to post |
| Caption | Text | Base caption text |
| Title / YouTube Title | Text | Title for YouTube uploads |

### Error Log Database

| Property | Type | Purpose |
|---|---|---|
| Retry Status | Select | "queued" for items awaiting retry |
| Next Retry At | Date | When to attempt next retry |
| Retry Count | Number | Current attempt count (max 3) |
| Master Page ID | Text | Reference to content page |
| Platform | Select | Which platform failed |
| Media URL | URL | CDN media URL |
| Caption Used | Text | Caption that was used |
| Content Title | Text | Content title |

## Development Guidelines

### Working with n8n Workflows

- Workflow files are JSON — validate with `python3 -m json.tool <file>.json` or `jq . <file>.json` before committing
- Keep node IDs stable when editing; changing IDs breaks internal references between nodes
- Test workflow changes in an n8n instance before committing
- Document any new nodes or significant logic changes in this file

### Commits & Version Control

- Keep commits small and focused with clear, descriptive messages
- Never commit secrets, credentials, or `.env` files
- When modifying workflow JSON, describe which nodes changed and why in the commit message

### Adding Application Code

If traditional application code (JavaScript, Python, etc.) is added to this project:

1. Choose a language/runtime and add the appropriate config (e.g., `package.json`, `pyproject.toml`)
2. Set up a linter and formatter from the start
3. Add a testing framework and write tests alongside new code
4. Configure CI/CD (e.g., GitHub Actions) for automated checks
5. Update this file with build/run/test commands

### Code Style (for n8n Code Nodes)

- n8n code nodes use JavaScript
- Keep code node logic minimal — prefer n8n's built-in nodes over custom code when possible
- Add inline comments for non-obvious business logic within code nodes

## Build / Test / Lint Commands

No traditional build system exists yet. Current validation:

```bash
# Validate JSON syntax
python3 -m json.tool main-workflow.json > /dev/null
python3 -m json.tool retry-workflow.json > /dev/null

# Or with jq
jq empty main-workflow.json
jq empty retry-workflow.json
```

## Updating This File

Update CLAUDE.md whenever:

- Workflow structure changes (new nodes, removed nodes, changed triggers)
- New integrations or external services are added
- Credential placeholders change
- A build system, test framework, or CI/CD pipeline is introduced
- Project structure changes significantly
- New conventions or architectural decisions are made
