---
name: jira-cli-read
description: Read Jira tickets, issues, sprints, and epics using the jira CLI. Use when the user asks to view a Jira ticket, list issues, check sprint status, browse epics, query backlog, or look up any Jira issue key.
---

# Jira Read (CLI)

Read-only Jira access via the `jira` CLI (ankitpokhrel/jira-cli).

## Prerequisites

- Binary: `jira` must be installed and available on `$PATH`
  ([ankitpokhrel/jira-cli](https://github.com/ankitpokhrel/jira-cli)).
- Config must exist (`jira init` already run). Default config lives at
  `~/.config/.jira/.config.yml`.

## Commands

Always add `--plain` so output is machine-readable (no interactive TUI).

### View a single ticket

```bash
jira issue view ISSUE-123 --plain
```

Include recent comments:

```bash
jira issue view ISSUE-123 --comments 5 --plain
```

### List issues

Default project issues:

```bash
jira issue list --plain --no-truncate
```

Filtered by assignee and status:

```bash
jira issue list -a "me" -s "In Progress" --plain
```

Specific columns only:

```bash
jira issue list --plain --columns key,summary,status,assignee,priority
```

Raw JQL query:

```bash
jira issue list -q "assignee = currentUser() AND status != Done" --plain
```

### Sprint issues

Current sprint:

```bash
jira sprint list --current --plain
```

Previous or next sprint:

```bash
jira sprint list --prev --plain
jira sprint list --next --plain
```

### Epics

```bash
jira epic list --plain
```

## Common flags

| Flag | Purpose |
|------|---------|
| `-p PROJECT` | Override the default project |
| `--paginate N` or `--paginate FROM:LIMIT` | Limit results (max 100) |
| `--plain` | Plain text output (required for non-interactive use) |
| `--no-truncate` | Show all columns in plain mode |
| `--columns col1,col2` | Pick specific columns |
| `--raw` | Raw JSON response from Jira API |

## Output handling

After running a command, summarize the key fields for the user:
**Key**, **Summary**, **Status**, **Assignee**, **Priority**.

When the Jira instance URL is known from the config, link directly to
the ticket: `https://<instance>/browse/ISSUE-123`.
