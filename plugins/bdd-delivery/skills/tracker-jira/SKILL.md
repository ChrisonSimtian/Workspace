---
name: tracker-jira
description: Jira adapter for the bdd-delivery workflow. Pushes a ticket draft to Jira, searches for duplicates, maps the SAFe hierarchy onto Jira issue types, and stamps the created keys back into the draft. Called by the ticket skill; not usually invoked directly. Trigger when the user asks to push a draft to Jira, search Jira for an existing ticket, or reconcile a draft with its Jira issue.
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash, PowerShell, ToolSearch
argument-hint: [draft path | issue key]
---

# Jira adapter

Mechanics only. The ticket *rules* live in [`../ticket/conventions.md`](../ticket/conventions.md) and are none of this skill's business.

## Access

Prefer Atlassian MCP tools if the environment has them — search for `createJiraIssue`, `editJiraIssue`, `getJiraIssue`, `searchJiraIssuesUsingJql`, `getJiraProjectIssueTypesMetadata`, `getJiraIssueTypeMetaWithFields`, `createIssueLink` via `ToolSearch`. Otherwise use the REST API v3 with credentials from the environment.

Config comes from `.bdd-delivery.json` → `jira`:

| Key | Use |
|---|---|
| `cloudId` | site hostname or UUID |
| `projectKey` | default project |
| `acceptanceCriteriaField` | custom field id for AC, or `null` when Gherkin goes in the description |
| `parentField` | usually `parent` |
| `serviceDeskProjects` | **never write to these** |

## Discover the project's shape once, then memoise it

Don't guess field ids or issue-type names. On first use in a project:

```
getJiraProjectIssueTypesMetadata   → issue type ids and the hierarchy levels
getJiraIssueTypeMetaWithFields     → required fields, allowed values, whether an
                                     Acceptance Criteria custom field exists at all
```

Write what you learn into `.bdd-delivery.json` (or the site overlay) as a memoised table — issue type ids, required fields, the AC field or its absence, the priority default. Re-discovering this on every run is waste, and guessing it produces silent field-drops.

**Many Jira projects have no Acceptance Criteria field.** That is normal, not a misconfiguration. When `acceptanceCriteriaField` is `null`, the Gherkin block goes in the description and that is the intended behaviour.

**Parent wiring:** modern Jira uses the `parent` field for every level, including Epic→Story. The legacy `customfield_10014` "Epic Link" still exists in older projects and is a common source of "the parent didn't stick". Check which one the project's create metadata actually accepts.

## Never write to a customer-facing project

Jira Service Management projects put the reporter on the other side of the ticket. A comment there can reach a customer, and some MCP tools default to a public comment with no parameter to make it internal.

- Refuse to create or comment on any issue in `serviceDeskProjects`.
- If a comment is genuinely needed, draft the body and ask the user to paste it with the internal-note toggle on.
- This applies even when the request is explicit. Say why, offer the draft.

## Push

1. **Dedup first.** `searchJiraIssuesUsingJql` with `project = <key> AND text ~ "<key terms>" AND created >= -365d`. Report near-matches and let the caller decide before creating anything.
2. **Epic first**, if the draft has one. Capture the returned key.
3. **Stories next**, each with `parent: <epicKey>`.
4. **Links last** — Blocks / Relates / Duplicate between new and existing issues.

```
createJiraIssue
  cloudId:        <jira.cloudId>
  projectKey:     <jira.projectKey>
  issueTypeName:  <from hierarchy config>
  summary:        <summary line>
  contentFormat:  markdown
  description:    <everything above the tracker:end marker>
  parent:         <epic key, stories only>
  additional_fields:
    labels:   [...]
    priority: {"name": "Medium"}
```

**Verify the first issue of a run.** Fetch it back and check the fenced Gherkin block survived. Markdown→ADF conversion is not always clean — a mangled block shows up as literal wiki markup (`h2.`) or a flattened code fence. If it came through wrong, switch to `contentFormat: adf` with an explicit `codeBlock` node rather than creating five more broken tickets.

**Attribution.** If the site requires an AI-authorship disclosure on written-on-behalf content, append it as the last line of the description. Substitute the model actually running — never copy a literal version string out of a doc, because it goes stale the moment the model rolls.

## Stamp back

Update the draft in place: header table `**Issue**` → `[KEY](https://<site>/browse/KEY)`, `**Status**` → `PUSHED <date>`, the structure table's Issue column, and each story heading.

## Reconcile

When asked to reconcile a draft with Jira, fetch each issue and diff the pushed region against the draft. Report drift in both directions — a description edited in the Jira UI is not wrong, it just means the draft is stale, and the draft is usually the one to update.

## Editing an existing issue

- **Don't rewrite a description wholesale.** Preserve what's there and change only what was asked; someone else may own the wording.
- Acceptance criteria added to an existing ticket follow the same Gherkin rules as a new one — no lower bar for a retrofit.
- `editJiraIssue` replaces the whole field, so send the full description including the parts you're keeping.
- Only stamp an authorship disclosure when you materially authored body content — not when you touched a label.
