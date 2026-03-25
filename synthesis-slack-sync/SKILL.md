---
name: synthesis-slack-sync
description: "Slack channel sync protocol for AI-assisted workflows. Reads channels and threads via Slack MCP, saves to local transcript files, and updates daily action plans. Handles mid-day re-syncs with thread staleness detection. Use when asked to: slack sync, sync from slack, check slack, read channels, sync messages, sync transcripts, what's new on slack."
license: "CC0-1.0"
depends_on:
  - synthesis-daily-rituals
  - synthesis-project-management
metadata:
  author: "Rajiv Pant"
  version: "1.0.0"
  source_repo: "github.com/rajivpant/synthesis-skills"
  source_type: "public"
---

# Synthesis Slack Sync

A protocol for syncing Slack channels and threads to local transcript files using Slack MCP. Designed for AI-assisted workflows where an agent reads Slack on behalf of a user, saves transcripts locally, and updates a daily action plan.

This skill provides the **protocol** — the sync methodology, thread re-reading discipline, transcript format, and action plan update rules. A per-project **config file** (`.claude/slack-sync.yaml`) provides the specifics: which channels, which paths, which DMs.

## Configuration

Create `.claude/slack-sync.yaml` in each project that uses this skill:

```yaml
# .claude/slack-sync.yaml — Slack sync configuration
#
# channels: Slack channels to monitor. All channel types supported.
# dm_channels: Active DM conversations to check (optional).
# transcripts_path: Where transcript files are saved (relative to ai-knowledge repo root).
# action_plan_path: Where daily action plans live (relative to ai-knowledge repo root).
# transcript_prefix: Filename prefix for transcript files (default: "slack-channels").
# ai_knowledge_repo: Absolute path to the ai-knowledge repo where transcripts are stored.

ai_knowledge_repo: ~/projects/my-projects/ai-knowledge/ai-knowledge-rajiv

transcripts_path: projects/_transcripts/
action_plan_path: projects/_daily-plans/
transcript_prefix: slack-channels

channels:
  - id: C0AGZCHGUAK
    name: mmc-product-growth-squad
    type: private_channel
  - id: C0AKDAQN34G
    name: tech-csa-pull-requests
    type: private_channel
  # Add more channels as needed

dm_channels: []
  # - id: U0AH9M2FYUQ
  #   name: Emil Penalo
```

If the config file is missing, the skill should warn and ask the user to create one.

---

## Prerequisites

- **Slack MCP must be connected and authenticated.** If any Slack tool call fails with an auth error, stop and instruct the user to re-authenticate: `claude mcp auth slack`, then restart the IDE/CLI.
- **Local transcript file must exist or be created.** The skill creates today's file if it doesn't exist.

---

## Sync Protocol

Every sync — whether day-start, mid-day, or day-end — follows the same five steps. No shortcuts, no skipped steps.

### Step 1: Read channels for new top-level messages

For each channel in the config:

```
slack_read_channel(channel_id, oldest=LAST_SYNC_TIMESTAMP, limit=30)
```

- On **first sync of the day** (no transcript file exists yet): omit `oldest` or use midnight timestamp.
- On **subsequent syncs**: use the timestamp of the last sync recorded in the transcript file.
- Note the **reply count** on every message that has threads. These will be re-read in Step 2.

### Step 2: Re-read ALL threads with replies from today

**This is the most important step. It is the step that gets skipped and causes missed messages.**

For every message in today's transcript that shows a thread (reply count > 0):

```
slack_read_thread(channel_id, message_ts=PARENT_TS)
```

Rules:
- **Never use the `oldest` parameter on thread reads.** It causes missed replies. Read the full thread every time.
- **Compare the reply count and latest reply timestamp** against what's in the local transcript.
- **If new replies exist**, append them to the transcript.
- **If the user sent a message** in a thread, it does NOT appear as a new channel-level message. The only way to detect it is to re-read the thread. If this step is skipped, the action plan shows drafts as "unsent" when the user already sent them.

**Mechanical check:** Before reporting "no new messages" for any sync, verify that every thread TS in today's transcript was re-read and reply counts match.

### Step 3: Check DMs

For each DM channel in the config:

```
slack_read_channel(channel_id=USER_ID, oldest=LAST_SYNC_TIMESTAMP, limit=20)
```

Only check DMs that have active conversations. Don't read every DM — the config file specifies which ones are active.

### Step 4: Save to local transcripts

**This step is not optional. Never skip it, even if "nothing changed."**

- Append all new messages and thread replies to the local transcript file.
- Record the sync time in the file (e.g., `## Mid-day sync (~14:30 EDT)`).
- File naming convention: `{transcript_prefix}-YYYY-MM-DD.md` (e.g., `slack-channels-2026-03-25.md`).
- If the file doesn't exist, create it with the standard header.

### Step 5: Update action plan

- **Mark sent messages as SENT** with timestamps. Cross-reference messages the user sent against draft messages in the action plan.
- **Update waiting-on-others** table with any new information from thread replies.
- **Note new action items** or signals worth responding to in the "Things to Know" section.
- **Do NOT remove content** from the action plan — it is append-only (mark done, don't delete).

---

## Transcript File Format

```markdown
# Slack Transcript — [Day], [Month] [Date], [Year]

Last synced: ~HH:MM TZ

---

## #channel-name (CHANNEL_ID)

### [Author Name] — HH:MM TZ (TS: [unix_timestamp])
[Message content]
**Thread ([N] replies):**
- [Reply Author] HH:MM (TS: [unix_timestamp]): "[reply text]"
- [Reply Author] HH:MM (TS: [unix_timestamp]): "[reply text]"
**Reactions:** [emoji_name] ([count])

---

## Mid-day sync (~HH:MM TZ)

### #channel-name

#### [Context heading] (TS: [unix_timestamp]) — [N] replies
- [New reply details]

---
```

Key rules:
- **Always record the TS (Unix timestamp)** for every significant message. TSs are the key to re-reading threads later.
- **Note reply counts** so the next sync can detect new replies.
- **Separate sync sessions** with a horizontal rule and a timestamp header.

---

## Date Verification

Before writing or naming any dated file, cross-check the date against at least two independent signals:

1. Slack Unix timestamps (convert with `date -r TS`)
2. Day-of-week clues in message content
3. User statements about the current day

The `currentDate` system value is a snapshot from session start. If a session crosses midnight, all subsequent dates will be wrong.

---

## Following Continuing Conversations

When a channel message references or continues an earlier discussion (broadcast replies, "also sent to channel," or topic continuations):

1. Grep local transcripts for the topic/keywords to find the parent message and its TS.
2. Read the parent thread via MCP using that TS — this surfaces all new replies.
3. Update the local transcript with any new thread replies.

**Do NOT search Slack MCP repeatedly.** Local transcripts are the source of truth for historical context. The point of syncing is to avoid depending on the MCP API for lookups.

---

## Error Handling

- **Slack MCP auth failure:** Stop immediately. Instruct user to run `claude mcp auth slack` and restart.
- **Channel not found:** The channel ID may have changed or the bot may have been removed. Warn and skip.
- **Rate limiting:** If Slack returns rate limit errors, wait and retry. Do not skip channels.
- **Empty channel:** Record "No new messages" in the transcript. Do not silently skip.

---

## When This Skill Runs

This skill is invoked:
- **By the user** typing `/slack-sync` or "sync from Slack" or similar
- **By `synthesis-daily-rituals`** during Day-Start (Step 2: Sync), Mid-Day Sync, and Day-End (Step 1: Transcript Sync)
- **Before drafting any Slack reply** — the daily-rituals skill requires re-reading the actual thread before drafting, to avoid stale-information replies

---

## Why Each Step Matters

These steps were developed through real incidents, not theory:

- **Step 2 (thread re-reading):** On 2026-03-24, a mid-day sync skipped thread re-reads. The action plan showed a draft as "unsent" when the user had already sent it hours earlier. The agent proposed sending it again, which would have been a duplicate message.
- **Step 4 (save to local):** Transcripts are the persistence layer. Without them, every sync starts from scratch, re-reading entire channels. With them, syncs are incremental and fast.
- **Step 5 (action plan update):** The action plan is the user's dashboard. If it shows stale information (unsent drafts that were sent, unresolved items that were resolved), the user makes wrong decisions.
