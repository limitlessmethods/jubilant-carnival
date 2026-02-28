# Pro Short-Form Auto-Poster v2 — Documentation

Complete setup guide, capabilities reference, and workflow architecture for the automated short-form social media posting system built on n8n.

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Prerequisites](#prerequisites)
4. [External Services Setup](#external-services-setup)
5. [Notion Database Setup](#notion-database-setup)
6. [Credential Configuration](#credential-configuration)
7. [Placeholder Reference](#placeholder-reference)
8. [Import & Deployment](#import--deployment)
9. [Main Workflow — Node-by-Node Reference](#main-workflow--node-by-node-reference)
10. [Retry Workflow — Node-by-Node Reference](#retry-workflow--node-by-node-reference)
11. [Platform Capabilities & Limits](#platform-capabilities--limits)
12. [Content Lifecycle & Status Flow](#content-lifecycle--status-flow)
13. [Error Handling & Retry Strategy](#error-handling--retry-strategy)
14. [AI Caption Enrichment](#ai-caption-enrichment)
15. [Slack Notifications](#slack-notifications)
16. [Testing & Validation](#testing--validation)
17. [Customization Guide](#customization-guide)
18. [Troubleshooting](#troubleshooting)
19. [File Reference](#file-reference)

---

## Overview

**Pro Short-Form Auto-Poster v2** is a production-grade n8n automation system that:

- Pulls content queued in a Notion database
- Optionally enriches captions using Claude AI for each platform
- Uploads media via Blotato
- Sequentially posts to 8 social media platforms with staggered delays
- Logs every result per-platform to a Notion error log database
- Calculates aggregate posting status and updates the master content page
- Sends Slack alerts on failures
- Automatically retries failed posts with exponential backoff (separate workflow)

### Supported Platforms

| # | Platform | Account | Blotato Platform String |
|---|----------|---------|------------------------|
| 1 | TikTok | thelimitlessjess | `tiktok` |
| 2 | Instagram | thelimitlessjess | `instagram` |
| 3 | YouTube Shorts | Lead And Launch | `youtube` |
| 4 | Threads | thelimitlessjess | `threads` |
| 5 | LinkedIn | (configured) | `linkedin` |
| 6 | X / Twitter | limitlessjess_ | `x` |
| 7 | Facebook | (page configured) | `facebook` |
| 8 | Pinterest | (board configured) | `pinterest` |

### Files

| File | Purpose |
|------|---------|
| `main-workflow.json` | Primary posting workflow (21 nodes) |
| `retry-workflow.json` | Automatic retry handler (22 nodes) |

---

## System Architecture

```
                          MAIN WORKFLOW
                          ============

  [Schedule Trigger]          09:00 UTC & 16:00 UTC daily
         |
  [Query Notion]              Get up to 5 "Ready To Post" items (FIFO)
         |
  [IF - Has Content] ------> FALSE: End (no content)
         |
       TRUE
         |
  [Set Processing Lock]      Write timestamp to prevent duplicates
         |
  [Content Validation]
     /         \
  VALID       INVALID -----> [Update Notion: Validation Error]
    |                              |
    |                        [Slack: Validation Failure]
    |
  [Claude Caption Enrichment] (optional, graceful fallback)
         |
  [Parse Enrichment]
         |
  [Update Enriched Captions]  Write AI captions back to Notion
         |
  [Blotato Media Upload]
         |
  [IF - Upload Success]
     /         \
  SUCCESS     FAIL ---------> [Mark All Platforms Failed] --> Merge
    |
    |  SEQUENTIAL POSTING CHAIN (with staggered delays)
    |
  [Post TikTok] -> [Set Result] -> [Wait 45s]
  [Post Instagram] -> [Set Result] -> [Wait 45s]
  [Post YouTube] -> [Set Result] -> [Wait 30s]
  [Post Threads] -> [Set Result] -> [Wait 30s]
  [Post LinkedIn] -> [Set Result] -> [Wait 30s]
  [Post X/Twitter] -> [Set Result] -> [Wait 30s]
  [Post Facebook] -> [Set Result] -> [Wait 30s]
  [Post Pinterest] -> [Set Result]
    |
  [Merge All Results]         Collect 8 platform results
         |
  [Prepare Log Entries]       Format for Notion error log
         |
  [Create Platform Logs]      Write 8 rows to Error Log DB
         |
  [Aggregate Status]          Calculate Posted / Partial / Failed
         |
  [Update Master Page]        Set final status, clear lock
         |
  [IF - Has Failures]
     /         \
  TRUE        FALSE: End
    |
  [Slack: Failure Notification]


                        RETRY WORKFLOW
                        ==============

  [Schedule Trigger]          Every 30 minutes
         |
  [Query Error Log]           queued + next_retry_at <= now + count < 3
         |
  [IF - Has Retries] -------> FALSE: End
         |
  [SplitInBatches]            Process one at a time
         |
  [Extract Retry Data]        Parse Notion properties
         |
  [Re-upload Media]
         |
  [IF - Re-upload Success]
     /         \
  SUCCESS     FAIL ---------> [Handle Re-upload Failure]
    |                              |
  [Platform Router]          [Update Error Log]
    |  (Switch to correct          |
    |   platform only)       [IF - Permanent Failure]
    |                           /         \
  [Retry Post]              TRUE        FALSE --> Loop
    |                         |
  [Calculate Retry Result]  [Slack: Permanent Failure]
         |                         |
  [Update Error Log Entry]       Loop
         |
  [Prepare Master Update]
         |
  [Fetch All Platform Statuses]
         |
  [Recalculate Overall Status]
         |
  [Update Master Page]
         |
  [IF - Permanent Failure]
     /         \
  TRUE        FALSE --> Loop
    |
  [Slack: Permanent Failure Alert]
         |
       Loop
```

---

## Prerequisites

### Required Accounts & Services

| Service | Purpose | Required |
|---------|---------|----------|
| **n8n** | Workflow automation engine (self-hosted or cloud) | Yes |
| **Notion** | Content database + error log database | Yes |
| **Blotato** | Unified social media posting API | Yes |
| **Slack** | Failure notifications via incoming webhook | Yes |
| **Anthropic API** | Claude AI caption enrichment | Optional |

### n8n Requirements

- n8n version **1.20+** (for node type version compatibility)
- Blotato community node installed: `@blotato/n8n-nodes-blotato`
- Sufficient execution timeout (recommended: 15 minutes for the main workflow due to staggered delays totaling ~4.5 minutes)

### Installing the Blotato Community Node

1. In n8n, go to **Settings** > **Community Nodes**
2. Click **Install a community node**
3. Enter: `@blotato/n8n-nodes-blotato`
4. Click **Install**
5. Restart n8n if prompted

---

## External Services Setup

### Blotato

1. Create an account at [blotato.com](https://blotato.com)
2. Connect all 8 social media accounts in the Blotato dashboard
3. Note each **Account ID** from the dashboard — you will need these for the placeholders
4. For Pinterest: note the **Board ID** you want to post to
5. For Facebook: note the **Page ID** you want to post to
6. Generate an API key in Blotato settings

### Slack Incoming Webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create a new app (or use existing)
2. Navigate to **Incoming Webhooks** > toggle **Activate**
3. Click **Add New Webhook to Workspace**
4. Select the channel for notifications
5. Copy the webhook URL (format: `https://hooks.slack.com/services/T.../B.../...`)

### Anthropic API (Optional)

1. Sign up at [console.anthropic.com](https://console.anthropic.com)
2. Create an API key under **API Keys**
3. Ensure your account has credits/billing configured
4. The workflow uses `claude-sonnet-4-20250514` with a 60-second timeout

---

## Notion Database Setup

You need **two** Notion databases. Share both with your Notion API integration.

### Content Database

This is your content queue. Create a database with these properties:

| Property Name | Type | Options / Notes |
|---------------|------|-----------------|
| `Title` | Title | Content title (default title column) |
| `Platform` | Select | Options: `Short Form`, `Long Form`, etc. |
| `Status` | Select | Options: `Ready To Post`, `Posted`, `Partial - ...`, `Failed`, `Validation Error` |
| `property_media_file` | Files & media | The video/image file to post |
| `property_caption` | Rich text | Master caption used as fallback |
| `youtube_title` | Rich text | Title for YouTube Shorts |
| `processing_started` | Date | Lock timestamp (set automatically) |
| `posted_at` | Date | When posting completed (set automatically) |
| `platform_results` | Rich text | JSON summary of results (set automatically) |
| `ai_enriched` | Checkbox | Whether Claude enrichment has run |
| `validation_error` | Rich text | Validation issue details (set automatically) |
| `property_tiktok_caption` | Rich text | Platform-specific caption |
| `property_instagram_caption` | Rich text | Platform-specific caption |
| `property_youtube_caption` | Rich text | Platform-specific caption |
| `property_threads_caption` | Rich text | Platform-specific caption |
| `property_linkedin_caption` | Rich text | Platform-specific caption |
| `property_twitter_caption` | Rich text | Platform-specific caption |
| `property_facebook_caption` | Rich text | Platform-specific caption |
| `property_pinterest_caption` | Rich text | Platform-specific caption |

### Error Log Database

This tracks every platform posting attempt. Create a database with these properties:

| Property Name | Type | Options / Notes |
|---------------|------|-----------------|
| `Title` | Title | Auto-format: `PLATFORM - STATUS - Content Title` |
| `Master Page ID` | Rich text | Links back to content DB page ID |
| `Platform` | Select | Options: `tiktok`, `instagram`, `youtube`, `threads`, `linkedin`, `twitter`, `facebook`, `pinterest` |
| `Status` | Select | Options: `success`, `failed` |
| `Error Message` | Rich text | Error details when failed |
| `Content Title` | Rich text | Content title for reference |
| `Media URL` | URL | The media file URL used |
| `Caption Used` | Rich text | Caption sent to platform (truncated to 200 chars) |
| `Retry Count` | Number | 0-3, incremented on each retry |
| `Retry Status` | Select | Options: `N/A`, `queued`, `succeeded`, `permanent_failure` |
| `Next Retry At` | Date | When to attempt next retry |
| `Workflow Run ID` | Rich text | n8n execution ID for tracing |
| `Timestamp` | Date | Include time, when the log was created |

### Getting Database IDs

1. Open each Notion database as a full page
2. The URL format is: `https://notion.so/your-workspace/DATABASE_ID?v=VIEW_ID`
3. The 32-character hex string before `?v=` is your database ID
4. You can also use it with dashes (Notion accepts both formats)

---

## Credential Configuration

### In n8n: Create Credentials

#### Notion API Credential

1. Go to **Settings** > **Credentials** > **Add Credential**
2. Select **Notion API**
3. Enter your Notion integration token (starts with `ntn_` or `secret_`)
4. Save and note the credential **ID** and **Name** shown in n8n

#### Blotato API Credential

1. Go to **Settings** > **Credentials** > **Add Credential**
2. Select **Blotato API** (from community node)
3. Enter your Blotato API key
4. Save and note the credential **ID** and **Name**

### Inline Credentials (in the JSON files)

These are passed directly as parameters, not through n8n's credential system:

- **Slack Webhook URL** — used in HTTP Request nodes
- **Anthropic API Key** — sent as `x-api-key` header

---

## Placeholder Reference

Search and replace these placeholders in **both** JSON files before importing:

### Credentials

| Placeholder | Replace With | Used In |
|-------------|-------------|---------|
| `[YOUR_NOTION_CRED_ID]` | n8n Notion credential ID | All Notion nodes |
| `[YOUR_NOTION_CRED_NAME]` | n8n Notion credential name | All Notion nodes |
| `[YOUR_BLOTATO_CRED_ID]` | n8n Blotato credential ID | All Blotato nodes |
| `[YOUR_BLOTATO_CRED_NAME]` | n8n Blotato credential name | All Blotato nodes |
| `[YOUR_SLACK_WEBHOOK_URL]` | Slack incoming webhook URL | HTTP Request nodes |
| `[YOUR_ANTHROPIC_API_KEY]` | Anthropic API key | Claude enrichment node |

### Database IDs

| Placeholder | Replace With |
|-------------|-------------|
| `[YOUR_CONTENT_DB_ID]` | Notion content database ID |
| `[YOUR_ERROR_LOG_DB_ID]` | Notion error log database ID |

### Platform Account IDs

| Placeholder | Replace With |
|-------------|-------------|
| `[TIKTOK_ACCOUNT_ID]` | Blotato TikTok account ID |
| `[INSTAGRAM_ACCOUNT_ID]` | Blotato Instagram account ID |
| `[YOUTUBE_ACCOUNT_ID]` | Blotato YouTube account ID |
| `[THREADS_ACCOUNT_ID]` | Blotato Threads account ID |
| `[LINKEDIN_ACCOUNT_ID]` | Blotato LinkedIn account ID |
| `[TWITTER_ACCOUNT_ID]` | Blotato X/Twitter account ID |
| `[FACEBOOK_ACCOUNT_ID]` | Blotato Facebook account ID |
| `[FACEBOOK_PAGE_ID]` | Facebook page ID |
| `[PINTEREST_ACCOUNT_ID]` | Blotato Pinterest account ID |
| `[PINTEREST_BOARD_ID]` | Pinterest board ID |

---

## Import & Deployment

### Step 1: Replace All Placeholders

Open both `main-workflow.json` and `retry-workflow.json` in a text editor. Use find-and-replace to substitute every `[PLACEHOLDER]` with your actual values. There are 18 distinct placeholders to replace (see table above).

### Step 2: Import Workflows

1. Open your n8n instance
2. Go to **Workflows** in the sidebar
3. Click the **"..."** menu (top-right) > **Import from File**
4. Select `main-workflow.json` > click **Import**
5. Repeat for `retry-workflow.json`

### Step 3: Verify Credential Bindings

After import, open each workflow and check every Notion and Blotato node:
- If you see a warning icon, click the node and re-select the correct credential from the dropdown
- This happens when credential IDs in the JSON don't match your n8n instance

### Step 4: Test with a Single Item

1. Create one test entry in your Content Database:
   - Set `Platform` to `Short Form`
   - Set `Status` to `Ready To Post`
   - Add a media file
   - Add a caption
   - Add a youtube_title
2. In the main workflow, click **Test Workflow** (manual trigger)
3. Watch the execution log node by node
4. Verify results appear in the Error Log Database
5. Verify the master page status was updated

### Step 5: Activate

1. Open the main workflow > click **Save** > toggle **Active** (top-right)
2. Open the retry workflow > click **Save** > toggle **Active**
3. Both workflows now run on their schedules

---

## Main Workflow — Node-by-Node Reference

### Node 1: Schedule Trigger

| Setting | Value |
|---------|-------|
| Type | `n8n-nodes-base.scheduleTrigger` v1.2 |
| Trigger 1 | 16:00 UTC daily |
| Trigger 2 | 09:00 UTC daily |

Two trigger slots allow morning and afternoon posting windows. Modify the hours to match your audience's peak engagement times.

### Node 2: Query Notion

| Setting | Value |
|---------|-------|
| Type | `n8n-nodes-base.notion` v2.2 |
| Operation | Get All (Database Pages) |
| Database | Content Database |
| Limit | 5 items per run |
| Sort | Created date ascending (FIFO) |

**Filter logic:**
```
Platform = "Short Form"
AND Status = "Ready To Post"
AND (processing_started is empty OR processing_started < 2 hours ago)
```

The stale lock cleanup (2-hour threshold) prevents items from being permanently locked if a previous execution crashed.

### Node 3: IF - Has Content

Gates the workflow. If the Notion query returned no items, the workflow ends cleanly without errors.

### Node 4: Set Processing Lock

Updates each Notion page's `processing_started` property to the current timestamp. This acts as a distributed lock to prevent overlapping executions from processing the same content.

### Node 5: Content Validation

JavaScript code node that validates each content item:

| Check | Validation |
|-------|-----------|
| Media file | `property_media_file[0]` exists and starts with `http` |
| Caption | `property_caption` is non-empty |
| YouTube title | `youtube_title` or `title` exists |
| Caption length | Warns if > 2200 chars (Instagram limit) or > 4000 (TikTok limit) |

Outputs two branches:
- **Output 0 (Valid):** continues to enrichment
- **Output 1 (Invalid):** routes to error handler

### Node 6: Invalid Content Handler

Two nodes handle invalid content:
1. **Update Invalid to Notion** — sets `Status` to `Validation Error`, writes `validation_error` details, clears the processing lock
2. **Slack - Validation Failure** — sends a webhook alert with the title and specific validation errors

### Node 7: Claude Caption Enrichment

| Setting | Value |
|---------|-------|
| Type | HTTP Request (to Anthropic API) |
| Model | `claude-sonnet-4-20250514` |
| Timeout | 60 seconds |
| continueOnFail | `true` |

Sends the master caption to Claude with a brand voice system prompt and requests platform-optimized versions for all 8 platforms. If the API call fails, the workflow continues with the master caption as fallback.

**Brand voice configuration (in system prompt):**
- Storyteller: 30%
- Opinionator: 25%
- Fact Presenter: 20%
- Frameworker: 20%
- F-Bomber: 5%
- Style: Casual, conversational, data-driven, action-oriented
- Uses present tense time markers, specific numbers
- Transitions with "Anyway" or "My point is"

### Node 8: Parse Enrichment

JavaScript code node that extracts the JSON response from Claude's output. Falls back to the master caption for any platform where parsing fails.

### Node 9: Update Enriched Captions

Writes all 8 platform-specific captions back to the Notion content page and sets `ai_enriched` to `true`. Has `continueOnFail: true` so a Notion API issue doesn't block posting.

### Node 10: Blotato Media Upload

Uploads the media file URL to Blotato's CDN. The returned `mediaId` is used by all subsequent platform posting nodes.

### Node 11: IF - Upload Success

Routes to the posting chain on success, or to the "Mark All Platforms Failed" handler on failure.

### Nodes 12-27: Sequential Posting Chain

Eight platform posts, each followed by a result capture node and a wait node:

| Order | Platform | Wait After | Caption Source |
|-------|----------|-----------|----------------|
| 1 | TikTok | 45 seconds | `tiktok_caption` -> `property_tiktok_caption` -> `property_caption` |
| 2 | Instagram | 45 seconds | `instagram_caption` -> `property_instagram_caption` -> `property_caption` |
| 3 | YouTube | 30 seconds | `youtube_caption` -> `property_youtube_caption` -> `property_caption` |
| 4 | Threads | 30 seconds | `threads_caption` -> `property_threads_caption` -> `property_caption` |
| 5 | LinkedIn | 30 seconds | `linkedin_caption` -> `property_linkedin_caption` -> `property_caption` |
| 6 | X/Twitter | 30 seconds | `twitter_caption` -> `property_twitter_caption` -> `property_caption` |
| 7 | Facebook | 30 seconds | `facebook_caption` -> `property_facebook_caption` -> `property_caption` |
| 8 | Pinterest | (none) | `pinterest_caption` -> `property_pinterest_caption` -> `property_caption` |

**Key behaviors:**
- Every platform posting node has `continueOnFail: true` — a failure on one platform does not stop the others
- Each Set Result node captures: platform name, status, error message, master page ID, content title, media URL, caption used, and workflow execution ID
- Caption resolution uses a three-level fallback: AI-enriched caption > pre-existing platform caption > master caption
- YouTube requires `postCreateYoutubeOptionTitle` (falls back to content title or "Short")
- Facebook requires `facebookPageId`
- Pinterest requires `pinterestBoardId`

**Total stagger time per content item:** ~4 minutes 15 seconds

### Node 28: Merge All Results

Collects all 8 Set Result outputs (plus the upload-fail path) into a single array using append mode.

### Node 29: Prepare Log Entries

JavaScript code node that formats each result for Notion insertion, including:
- Composing the log title: `PLATFORM - STATUS - Content Title`
- Setting `retry_status` to `queued` for failures, `N/A` for successes
- Setting `next_retry_at` to 30 minutes from now for failures

### Node 30: Create Platform Logs

Creates one Notion page per platform result in the Error Log Database with all fields populated.

### Node 31: Aggregate Status Calculator

JavaScript code node that analyzes all 8 results:

| Condition | Calculated Status |
|-----------|------------------|
| All 8 succeeded | `Posted` |
| Some succeeded, some failed | `Partial - [failed platforms] failed` |
| All 8 failed | `Failed` |

Also builds a `platform_results` JSON object for the master page.

### Node 32: Update Master Notion Page

Updates the original content page with:
- `Status`: the calculated status
- `posted_at`: current timestamp
- `platform_results`: JSON summary
- `processing_started`: cleared (releases the lock)

### Node 33: IF - Has Failures / Slack Notification

Only fires if any platform failed. Sends a Slack Block Kit message with:
- Content title
- Calculated status
- Failed platforms list
- Error details per platform
- Note that retries are queued
- Link to the Notion content page

---

## Retry Workflow — Node-by-Node Reference

### Node 1: Schedule Trigger

Runs every **30 minutes** to check for queued retries.

### Node 2: Query Error Log

Queries the Error Log Database with filter:
```
Retry Status = "queued"
AND Next Retry At <= now()
AND Retry Count < 3
```

Limit: 10 items per run, sorted by `Next Retry At` ascending (oldest retries first).

### Node 3: IF - Has Retries

Gates the workflow when no retries are pending.

### Node 4: Process One at a Time (SplitInBatches)

Processes failed items one at a time to avoid overwhelming the Blotato API. Loops back after each item is fully processed.

### Node 5: Extract Retry Data

JavaScript code node that parses Notion's nested property format into clean flat fields: `master_page_id`, `platform`, `media_url`, `caption_used`, `retry_count`, `error_log_id`, `content_title`.

### Node 6: Re-upload Media

Re-uploads the media file to Blotato since media IDs may expire. Has `continueOnFail: true`.

### Node 7: IF - Re-upload Success

Routes to the platform router on success, or to the re-upload failure handler.

### Node 8: Platform Router (Switch)

Routes to the correct platform posting node based on the `platform` field. Uses 8 outputs, one per platform.

### Nodes 9-16: Platform Retry Posts

One Blotato posting node per platform. Each uses:
- The same caption from the original attempt
- The fresh `mediaId` from re-upload
- Platform-specific parameters (YouTube title, Facebook page ID, Pinterest board ID)
- `continueOnFail: true`

### Node 17: Calculate Retry Result

Determines the outcome and next steps:

| Condition | Action |
|-----------|--------|
| Post succeeded | `retry_status = "succeeded"` |
| Post failed, retry_count < 3 | `retry_status = "queued"`, schedule next retry with backoff |
| Post failed, retry_count = 3 | `retry_status = "permanent_failure"` |

**Exponential backoff schedule:**

| Retry # | Delay |
|---------|-------|
| 1 | 30 minutes |
| 2 | 2 hours |
| 3 | 8 hours |

### Node 18: Update Error Log Entry

Writes the retry result back to the error log entry: updated retry count, retry status, next retry time, and error message.

### Nodes 19-20: Recalculate Master Status

1. Fetches all error log entries for the same master page ID
2. Recalculates the overall status across all platforms
3. If all platforms have now succeeded (including via retries): sets status to `Posted`

### Node 21: Update Master Page

Writes the recalculated status and `platform_results` JSON back to the content database.

### Node 22: Permanent Failure Alert

If a retry reaches `retry_count = 3`, sends a Slack Block Kit message:
- Header: "Permanent Posting Failure - Manual Action Required"
- Platform name and content title
- Last error message
- Link to Notion page

### Re-upload Failure Path

If media re-upload fails, a separate path:
1. Calculates the retry result (same backoff logic)
2. Updates the error log entry
3. Checks for permanent failure
4. Sends Slack alert if permanent
5. Loops back to process the next item

---

## Platform Capabilities & Limits

| Platform | Max Caption Length | Media Types | Special Parameters |
|----------|-------------------|-------------|-------------------|
| TikTok | ~4,000 chars | Video | — |
| Instagram | 2,200 chars | Video, Image | — |
| YouTube Shorts | 5,000 chars (description) | Video (< 60s) | `postCreateYoutubeOptionTitle` (required) |
| Threads | 500 chars | Video, Image | — |
| LinkedIn | 3,000 chars | Video, Image | — |
| X / Twitter | 280 chars (text), ~4,000 (with media) | Video, Image | — |
| Facebook | 63,206 chars | Video, Image | `facebookPageId` (required) |
| Pinterest | 500 chars (description) | Video, Image | `pinterestBoardId` (required) |

---

## Content Lifecycle & Status Flow

```
                    Content Status Flow
                    ===================

  [Draft] (manual)
      |
      v
  [Ready To Post] (manual — triggers pickup by workflow)
      |
      +--- validation fails ---> [Validation Error]
      |
      +--- processing_started set (lock acquired)
      |
      +--- all succeed --------> [Posted]
      |
      +--- some succeed -------> [Partial - twitter, pinterest failed]
      |                               |
      |                          retries run...
      |                               |
      |                          all eventually succeed --> [Posted]
      |                               |
      |                          some permanently fail --> stays Partial
      |
      +--- all fail ------------> [Failed]
                                      |
                                 retries run...
                                      |
                                 some succeed --> [Partial - ...]
                                      |
                                 all succeed --> [Posted]
                                      |
                                 all permanently fail --> stays [Failed]
```

### Lock Mechanism

- `processing_started` is set to `now()` when a content item is picked up
- Stale locks (older than 2 hours) are automatically eligible for re-processing
- Lock is cleared when the master page is updated at the end of the workflow
- Prevents duplicate posting from overlapping trigger executions

---

## Error Handling & Retry Strategy

### Failure Categories

| Category | Handling |
|----------|---------|
| Content validation failure | Status set to "Validation Error", Slack alert, no retries |
| Media upload failure | All 8 platforms marked as failed, retries queued |
| Individual platform failure | Other platforms continue, failed one is retried |
| Claude enrichment failure | Graceful fallback to master caption |
| Notion update failure | `continueOnFail` prevents workflow halt |

### Retry Behavior

- **Maximum retries:** 3 per platform per content item
- **Retry check interval:** Every 30 minutes
- **Backoff schedule:** 30 min → 2 hours → 8 hours
- **Items per retry run:** Up to 10
- **Processing:** One at a time (SplitInBatches)

### Permanent Failure

After 3 failed retry attempts:
- Error log entry marked as `permanent_failure`
- Slack alert sent with "Manual Action Required"
- No further automatic retries
- Manual intervention needed: fix the issue, then set `Retry Status` back to `queued` and `Retry Count` to 0 in Notion

---

## AI Caption Enrichment

### How It Works

1. Content validation passes, item has `ai_enriched` unchecked
2. The master caption and content title are sent to Claude via the Anthropic API
3. Claude returns a JSON object with 8 platform-optimized captions
4. Captions are parsed and written back to the Notion content page
5. `ai_enriched` is set to `true`

### Disabling Enrichment

To skip Claude enrichment entirely:
1. Open the main workflow in n8n
2. Click the **Claude Caption Enrichment** node
3. Click **Deactivate** (or disconnect the node)
4. Connect the **Content Validation** valid output directly to **Blotato Media Upload**

The posting chain will use `property_[platform]_caption` if pre-populated, or fall back to `property_caption`.

### Cost Estimation

Each enrichment call uses approximately 500-1,500 tokens. At `claude-sonnet-4-20250514` pricing, this is roughly $0.01-0.03 per content item.

---

## Slack Notifications

### Notification Types

| Trigger | Channel Message |
|---------|----------------|
| Validation failure | Warning with content title and specific errors |
| Platform posting failure | Alert with failed platforms list, error details, retry status |
| Permanent retry failure | Critical alert with "Manual Action Required" and last error |
| Upload permanent failure | Critical alert for media upload exhausting retries |

### Message Format

All notifications use Slack Block Kit for rich formatting with:
- Header blocks with status icons
- Section blocks with field pairs
- Links to the Notion content page

---

## Testing & Validation

### Pre-Deployment Checklist

- [ ] All 18 placeholders replaced in both JSON files
- [ ] Notion integration has access to both databases
- [ ] All Content Database properties exist with correct types
- [ ] All Error Log Database properties exist with correct types
- [ ] Blotato API credential is valid and all 8 accounts connected
- [ ] Slack webhook URL is valid and posts to correct channel
- [ ] Anthropic API key is valid (if using enrichment)
- [ ] Blotato community node is installed in n8n

### Testing Procedure

1. **Test validation:** Create a content item missing the media file. Run the main workflow manually. Verify it routes to validation error and Slack alert fires.

2. **Test single platform:** Temporarily disconnect all platform nodes except TikTok. Create a valid content item. Run manually. Verify TikTok post appears and Notion logs are created.

3. **Test full chain:** Reconnect all platforms. Run with a real content item. Monitor each node's output in the execution log.

4. **Test retry:** Manually create an error log entry with `Retry Status = queued`, `Next Retry At` in the past, `Retry Count = 0`. Run the retry workflow manually.

5. **Test permanent failure:** Set `Retry Count = 2` on an error log entry with `Retry Status = queued`. Run the retry workflow. Verify Slack permanent failure alert.

---

## Customization Guide

### Changing Post Times

Edit the **Schedule Trigger** node in `main-workflow.json`:
```json
"interval": [
  { "triggerAtHour": 16 },  // Change to desired UTC hour
  { "triggerAtHour": 9 }    // Change or remove second slot
]
```

### Changing Stagger Delays

Edit the Wait nodes. Each has an `amount` and `unit` parameter:
```json
{ "amount": 45, "unit": "seconds" }
```

### Changing Platform Order

Rearrange the connections in the `connections` object. The chain flows through Wait nodes, so reorder the `Wait After [Platform]` → `Post to [Next Platform]` connections.

### Adding a New Platform

1. Add a new Blotato posting node after the last Wait node
2. Add a Set Result node after it
3. Connect the Set Result to the Merge node (increment `numberInputs`)
4. Update the Content Validation code if the platform has special requirements
5. Add the platform to the Claude enrichment system prompt
6. Add the platform-specific caption field to Notion

### Removing a Platform

1. Disconnect the platform's posting node
2. Remove the posting node, its Set Result node, and the preceding Wait node
3. Reconnect the chain (previous Set Result → next Wait or next Post)
4. Reduce `numberInputs` on the Merge node

### Adjusting Retry Backoff

Edit the `backoffMinutes` array in the **Calculate Retry Result** code node:
```javascript
const backoffMinutes = [30, 120, 480];  // retry 1, 2, 3
```

### Changing Max Retries

Edit the filter in **Query Error Log** (change `less_than: 3`) and the condition in **Calculate Retry Result** (change `currentRetryCount >= 3`).

---

## Troubleshooting

### Content not being picked up

- Verify `Platform` is exactly `Short Form` (case-sensitive)
- Verify `Status` is exactly `Ready To Post`
- Check if `processing_started` is set and less than 2 hours old (indicating another execution is active)
- Verify the Notion integration has access to the database

### Media upload fails

- Ensure the file URL in `property_media_file` is publicly accessible
- Check Blotato dashboard for API rate limits or account issues
- Verify the file format is supported by all target platforms

### Claude enrichment returns master caption for everything

- Check Anthropic API key validity
- Check the execution log for the Claude enrichment node's response
- Ensure the response contains valid JSON (sometimes the model wraps it in markdown code blocks — the parser handles this with regex)

### Retries not running

- Verify the retry workflow is **Active**
- Check that `Retry Status` is `queued` (not `N/A` or `permanent_failure`)
- Check that `Next Retry At` is in the past
- Check that `Retry Count` is less than 3

### Slack notifications not arriving

- Test the webhook URL directly with curl: `curl -X POST -H 'Content-type: application/json' --data '{"text":"test"}' YOUR_WEBHOOK_URL`
- Check that the Slack app hasn't been removed from the channel

### Duplicate posts

- Check if two executions ran simultaneously (overlapping schedule triggers)
- Verify the processing lock is working: `processing_started` should be set immediately
- If stale locks are being cleared too aggressively, increase the 2-hour threshold in the Query Notion filter

---

## File Reference

```
/home/user/jubilant-carnival/
├── CLAUDE.md                 # AI assistant guidance
├── README.md                 # Project description
├── WORKFLOW-DOCS.md          # This documentation file
├── main-workflow.json        # Primary posting workflow (n8n import)
└── retry-workflow.json       # Retry handler workflow (n8n import)
```

### Node Type Versions Used

| Node Type | Version |
|-----------|---------|
| `n8n-nodes-base.scheduleTrigger` | 1.2 |
| `n8n-nodes-base.notion` | 2.2 |
| `@blotato/n8n-nodes-blotato.blotato` | 2 |
| `n8n-nodes-base.set` | 3.4 |
| `n8n-nodes-base.code` | 2 |
| `n8n-nodes-base.httpRequest` | 4.2 |
| `n8n-nodes-base.merge` | 3 |
| `n8n-nodes-base.wait` | 1.1 |
| `n8n-nodes-base.if` | 2 |
| `n8n-nodes-base.switch` | 3 |
| `n8n-nodes-base.splitInBatches` | 3 |
| `n8n-nodes-base.noOp` | 1 |
