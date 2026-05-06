---
name: pr-review
description: Reviews pull requests and code changes in Ansible collections against project standards and common collection quality checks. Use when asked to review a PR, patch, diff, or set of code changes. Do not use for GitHub Issues or general Q&A unrelated to a changeset.
---

# Skill: PR review (Ansible collections)

## Purpose

Review pull requests and code changes in **Ansible collections** (modules,
plugins, module_utils, tests, docs, metadata) against repository rules and
general Ansible collection expectations.

## When to Invoke

Trigger when:

- The user asks to review a PR, patch, diff, or set of code changes
- Validating changes against project standards before merge

Do not trigger when:

- The task is only about GitHub Issues (not PRs or code changes)
- General Q&A, documentation lookup, or debugging unrelated to a changeset

## Inputs

- `target` (optional): PR number, branch name, commit hash, or file path.
  - If omitted, review the current working changes via `git diff HEAD`.

## Delegation and sources of truth

1. **Repository first**: Apply the repo's own `AGENTS.md`, `README`, and
   `.cursor/rules` (or equivalent) when present. They override generic checks.
2. **Global defaults**: Use the team's shared collection-wide conventions
   when the repo does not define a rule.
3. **External documentation**: For Ansible platform behavior, third-party
   library APIs, or tool flags not obvious from code, use parallel lookups,
   extract minimal relevant content, and cite sources.

## Approach

### Step 1 — Gather the changeset

Obtain the diff using the appropriate method:

- PR number provided → read changed files and their diffs (e.g. `gh pr diff`)
- Branch or commit reference → `git diff <base>..<ref>` or `git show <ref>`
- File path provided → read the file and review it in full
- No target → `git diff HEAD` for all current changes

Read every changed file completely before forming any judgment.

### Step 2 — Run review checks

Execute the checklist below. Where items overlap **Architecture**, **check
mode**, or **type/API** details, apply the repo's `AGENTS.md` / rules first;
use **docs-explorer** behavior for verifying external or official docs.

Collect findings per category.

### Step 3 — Report

Produce the structured report in **Output Format** below.

---

## Review checklist

### Collection metadata

- `galaxy.yml`: `version`, `description`, `tags`, `dependencies` are accurate
  and sensible for the change.
- `meta/runtime.yml`: `requires_ansible` minimum matches any new Ansible
  features used.
- Python dependencies: if the repo uses `requirements.txt` and/or
  `meta/ee-requirements.txt` (or EE images), new deps appear in the right
  place(s) and stay in sync.

### Module and plugin documentation

- Every public module/plugin option has `description`, `type`, and `required`
  or `default` as appropriate.
- `EXAMPLES` is present where required by project standards, valid YAML, and
  covers primary use cases.
- `RETURN` accurately describes keys returned (for modules that return data).
- `short_description` is concise and accurate.
- `author` is present and correctly formatted when the project expects it.

### Naming and style

- File and FQCN naming follow the collection's conventions (consistent prefixes,
  no gratuitous abbreviations).
- Python style matches project rules (PEP 8, line length, typing if required).

### Idempotency

- `changed` is `False` when no real change is made (modules and logical
  equivalents for plugins).
- Repeated runs with the same arguments do not cause spurious changes.

### check_mode

- Mutating operations respect check mode where applicable (modules and action
  plugins per Ansible patterns).

### Sensitive data

- Sensitive options use `no_log=True` where appropriate.
- Return values and logs do not echo secrets in plaintext.

### Error handling

- Modules use `module.fail_json(msg=...)` with actionable messages — avoid bare
  `raise` or `sys.exit()` for user-facing errors.
- Action plugins use the appropriate action-base failure APIs
  (e.g. `fail_json` / `AnsibleActionFail`) per project patterns.
- Read-only or informational modules may return partial results instead of
  failing hard when that matches project design.

### Shared code and doc fragments

- Shared logic lives in `plugins/module_utils/` (or collection-documented
  utils), not duplicated across plugins.
- Repeated connection or auth options use doc fragments / shared argument specs
  instead of copy-pasted parameters.

### Action plugins (when the collection uses them)

- Business logic and API calls live in action plugins where that is the
  project's pattern; modules stay thin if documented as such.
- Argument validation and check mode behavior align with Ansible expectations.

### Testing

- Sanity: run or recommend `ansible-test sanity` scoped to what changed (e.g.
  paths or `--docker` if the project uses it).
- Unit tests under `tests/unit/` (often `tests/unit/plugins/...`) for new or
  changed logic in `module_utils` or non-trivial code paths.
- Integration tests under `tests/integration/targets/...` when the collection
  uses them for behavior changes: cover happy path, idempotency (run twice), and
  `state: absent` (or equivalent) where applicable.
- Assertions and registrations follow the collection's established patterns.

### Backwards compatibility

- No removal or rename of public options without deprecation where required.
- Return value shape and types remain compatible or are explicitly called out as
  breaking with justification.
- Deprecations use Ansible's deprecation mechanisms when applicable.

### Changelog fragment

- User-visible behavior changes include a fragment under
  `changelogs/fragments/` when the project requires it; text is concise, past
  tense, and names the affected surface.

### Code quality

- No dead code, commented-out blocks, or stray debug output.
- No unnecessary feature flags or speculative abstractions.
- No obvious injection or unsafe handling of untrusted input; no hardcoded
  credentials.

---

## Output format

Structure the review as follows:

```
## PR Review: <target or "Current Changes">

### Summary
<One-paragraph overall assessment: scope of the change, general quality, primary concerns.>

### Findings

#### Blockers (must fix before merge)
- [CATEGORY] <File>:<line> — <description of the issue>

#### Warnings (should fix, not strictly blocking)
- [CATEGORY] <File>:<line> — <description of the issue>

#### Suggestions (optional improvements)
- [CATEGORY] <File>:<line> — <description of the issue>

### Checklist status
| Category | Status | Notes |
|---|---|---|
| Collection metadata | PASS / FAIL / N/A | ... |
| Module/plugin documentation | PASS / FAIL / N/A | ... |
| Naming and style | PASS / FAIL / N/A | ... |
| Architecture / shared code | PASS / FAIL / N/A | ... |
| Idempotency | PASS / FAIL / N/A | ... |
| check_mode | PASS / FAIL / N/A | ... |
| Sensitive data | PASS / FAIL / N/A | ... |
| Error handling | PASS / FAIL / N/A | ... |
| Action plugins | PASS / FAIL / N/A | ... |
| Testing | PASS / FAIL / N/A | ... |
| Backwards compatibility | PASS / FAIL / N/A | ... |
| Changelog fragment | PASS / FAIL / N/A | ... |
| Code quality | PASS / FAIL / N/A | ... |

### Verdict
APPROVE / REQUEST CHANGES / COMMENT

<One sentence justifying the verdict.>
```

Use `N/A` for categories that do not apply to the changeset (e.g. action plugins
for a docs-only PR). Be specific: always reference the file and line number when
citing a finding.
