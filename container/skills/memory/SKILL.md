---
name: memory
description: Date-based persistent memory using Cloudflare R2. Loads MEMORY.md and yesterday's daily summary at session start. Saves daily summaries automatically at midnight KST. Use when the user asks about memory, past conversations, or runs /memory.
---

# Memory System (R2-based)

This NanoClaw instance stores persistent memory in Cloudflare R2, organized by date.

## R2 File Structure

```
memory/
  MEMORY.md               — Permanent identity, preferences, and settings
  daily/
    YYYY-MM-DD.md         — Daily conversation summaries (auto-saved at midnight KST)
  weekly/
    YYYY-WNN.md           — Weekly rollups (auto-saved every Monday 00:05 KST)
  monthly/
    YYYY-MM.md            — Monthly rollups (auto-saved on 1st of each month 00:10 KST)
uploads/
  {timestamp}-{filename}  — Files received from Discord attachments
```

## Loading Memory at Session Start

At the start of every new conversation:

1. Call `mcp__nanoclaw__r2_read` with key `memory/MEMORY.md`
2. Call `mcp__nanoclaw__r2_read` with key `memory/daily/{yesterday-KST}.md` (skip if not found)

Yesterday KST = today's date in Asia/Seoul timezone minus 1 day, format `YYYY-MM-DD`.

Use the loaded content to recall past context, preferences, and ongoing tasks.

## Saving a Daily Summary

When asked to save today's summary, or when the daily cron task fires:

1. Get today's date in KST (Asia/Seoul), format `YYYY-MM-DD`
2. Compile a summary of the day's conversation using this format:

```markdown
# Daily Summary: YYYY-MM-DD

## 주요 작업/결정사항
- ...

## 논의된 주제
- ...

## 진행 중인 작업
- ...

## 내일 할 일
- ...
```

3. Save with:
   ```
   mcp__nanoclaw__r2_upload(key="memory/daily/YYYY-MM-DD.md", content=<summary>)
   ```

## Updating MEMORY.md

When identity, preferences, or important settings change:

```
mcp__nanoclaw__r2_upload(key="memory/MEMORY.md", content=<new content>)
```

Recommended MEMORY.md structure:

```markdown
# NanoClaw Memory

## Identity
- Instance name, group affiliation, etc.

## Capabilities
- List of active integrations

## Preferences
- Response language, style, etc.

## R2 Structure
- Brief description of file layout

## Last Updated
- ISO timestamp
```

## Viewing Past Memory

To read a specific past summary:

```
mcp__nanoclaw__r2_read(key="memory/daily/YYYY-MM-DD.md")
```

To list all saved files:

```
mcp__nanoclaw__r2_list(prefix="memory/")
```

## Sharing Files

To generate a temporary download link for any R2 file:

```
mcp__nanoclaw__r2_share(key="uploads/filename.png", ttl_seconds=86400)
```

## Automatic Schedule Tasks

These tasks should be registered once per instance using `mcp__nanoclaw__schedule_task`:

| Task | Cron | Context | Action |
|------|------|---------|--------|
| Daily summary | `0 0 * * *` | `group` | Summarize today's conversation → `memory/daily/YYYY-MM-DD.md` |
| Weekly rollup | `5 0 * * 1` | `isolated` | Compact 7 daily summaries → `memory/weekly/YYYY-WNN.md` |
| Monthly rollup | `10 0 1 * *` | `isolated` | Compact weekly summaries → `memory/monthly/YYYY-MM.md` |

Use `mcp__nanoclaw__list_tasks` to check if they are already registered before creating duplicates.

## Initial Setup (New Instance)

To set up the memory system on a fresh NanoClaw instance:

1. Create `memory/MEMORY.md` in R2:
   ```
   mcp__nanoclaw__r2_upload(key="memory/MEMORY.md", content=<initial content>)
   ```

2. Register the 3 schedule tasks above.

3. Add memory-loading instructions to `/workspace/group/CLAUDE.md` so they persist across sessions. The file should reference this skill's session-start steps.

4. Optionally save a summary of the setup conversation as today's daily entry.
