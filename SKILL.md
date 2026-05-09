---
name: documentation-operations-brief
description: Use when a "Documentation Operations Brief" routine execution issue is assigned, or when a Document Writer heartbeat fires with no other assigned work. Applies to any agent acting as a documentation function fallback. Covers queue scanning, vault hygiene, risk/dependency flagging, and brief publication to the Obsidian vault.
---

# Documentation Operations Brief

Produces a daily snapshot of documentation health when the Document Writer has no other assigned work. Saves the brief to the Obsidian vault and closes the execution issue.

## When to invoke

- A "Documentation Operations Brief" routine execution issue has been assigned (created by TEC-1127 or the board)
- Heartbeat fires with no other `todo` or `in_progress` issues assigned and the agent role is Document Writer
- Any agent acting as a documentation fallback when the above conditions apply

Do NOT produce a brief if higher-priority assigned issues exist — work those first.

## Checklist

### Step 1 — Inbox check

```bash
GET /api/agents/me/inbox-lite
```

If any `todo` or `in_progress` issue is assigned, **stop here** — skip the brief and work those issues instead.

### Step 2 — Paperclip queue snapshot

```bash
GET /api/companies/{companyId}/issues?assigneeAgentId={docWriterAgentId}&status=todo,in_progress,blocked,in_review
```

Count by status. Note any overdue or high-priority items. Also check for issues with titles containing: "document", "write", "draft", "spec", "runbook", "guide", "update docs".

### Step 3 — Vault scan (Obsidian)

Using the `obsidian-cli` skill or direct file access, scan:

- `_Inbox/` — unprocessed inbox items (any `.md` not moved to a project folder)
- `00_Meta/Governance/` — stale governance docs (last-modified > 30 days with `status: draft`)
- Recent project notes without a linked Paperclip issue (orphaned notes)

Count stale items and flag anything older than 14 days in `_Inbox/`.

### Step 4 — Produce the brief

Write a brief with these four sections:

**a) Queue Snapshot**
- Count of open doc-related issues by status (todo / in_progress / blocked / in_review)
- Flag any `blocked` items with no resolution date

**b) Vault Hygiene**
- Count of `_Inbox/` items (flag if > 5 unprocessed)
- Count of stale draft docs (flag if > 3 older than 30 days)
- Any orphaned notes not linked to an issue

**c) Risk / Dependencies**
- Overdue items: issues with `priority: high` sitting in `todo` for > 7 days
- Blocked issues with no `blockedByIssueIds` (free-text blockers only — these need linkage)
- Any doc deliverables blocking other agents (check `blocks` field on issues)

**d) Next Actions**
- Top 1–3 prioritised doc actions for the next heartbeat, based on queue and hygiene findings
- Each action should name a specific issue or vault path and the action required

### Step 5 — Save and publish

**Save to vault:**

```
00_Meta/Governance/Doc Ops Brief YYYY-MM-DD.md
```

Frontmatter template:
```yaml
---
title: Doc Ops Brief YYYY-MM-DD
tags: [governance, doc-ops, brief]
status: published
date: YYYY-MM-DD
created-by: agent/document-writer
---
```

**Post as completion comment** on the execution issue (full brief content in the comment body).

**Mark the issue done:**
```bash
PATCH /api/issues/{issueId}
{ "status": "done", "comment": "## Done\n\nDoc Ops Brief published to vault and posted above." }
```

## Fallback — vault unavailable

If the Obsidian vault is inaccessible (mount missing, CLI error, permission denied):

1. Skip Steps 3 and the Vault Hygiene section
2. Produce a minimal brief using Paperclip issue state only (Steps 2 and 4a/c/d)
3. Add a note at the top of the brief: `> ⚠️ Vault unavailable — hygiene section omitted. Vault access required for next run.`
4. Save the brief to the Paperclip issue document instead of the vault (`PUT /api/issues/{issueId}/documents/doc-ops-brief`)
5. Mark the issue done with a comment noting the vault was unavailable

## Thresholds that trigger flags

| Metric | Flag threshold |
|---|---|
| `_Inbox/` unprocessed items | > 5 |
| Draft docs older than 30 days | > 3 |
| High-priority `todo` issues sitting idle | > 7 days |
| Blocked issues with free-text-only blockers | Any |
| Issues with no next-action comment in > 14 days | Any |

## Common mistakes

- Producing the brief when higher-priority assigned work exists — always check inbox first
- Omitting the `created-by` frontmatter field (required for vault filing conventions)
- Posting a one-line completion comment without the brief content — the brief must be in the comment body
- Skipping the Paperclip queue scan and only scanning the vault — both are required when vault is available
