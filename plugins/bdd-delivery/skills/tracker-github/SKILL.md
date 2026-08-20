---
name: tracker-github
description: GitHub Issues adapter for the bdd-delivery workflow. Pushes a ticket draft to GitHub Issues via the gh CLI, searches for duplicates, maps the SAFe hierarchy onto milestones and sub-issues, and stamps the created numbers back into the draft. Called by the ticket skill; not usually invoked directly. Trigger when the user asks to push a draft to GitHub Issues, or to raise a GitHub issue for a piece of work.
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash, PowerShell
argument-hint: [draft path | issue number]
---

# GitHub Issues adapter

Mechanics only. The ticket *rules* live in [`../ticket/conventions.md`](../ticket/conventions.md) and are none of this skill's business.

Uses the `gh` CLI throughout. Config comes from `.bdd-delivery.json` → `github`:

| Key | Use |
|---|---|
| `repo` | `owner/name`; defaults to the current repo |
| `typeLabels` | map ticket type → label, since GitHub has no issue-type field |
| `useMilestoneAsEpic` | model the Feature level as a milestone rather than a tracking issue |

## Check the repo is private before writing internal detail

**This is the one that bites.** GitHub Issues on a public repo are world-readable, including every incident reference, hostname, customer count and internal path that makes a good ticket good.

```sh
gh repo view <repo> --json visibility,hasIssuesEnabled
```

- **Public repo** → say so and stop. Ask whether to sanitise the body, target a private repo, or keep the ticket as a draft file only. Do not push an evidence-rich internal ticket to a public tracker on the assumption it was intended.
- **Issues disabled** → they can be enabled with `gh api -X PATCH repos/<repo> -f has_issues=true`, but that is a repo setting change. Ask first, and say what you changed.

## Hierarchy

GitHub has no native issue type and, until recently, no native hierarchy. Map it:

| SAFe level | GitHub |
|---|---|
| Portfolio Epic | milestone, or a tracking issue with a task list |
| Feature | milestone (`useMilestoneAsEpic: true`) or a tracking issue |
| Story | an issue |
| Sub-task | a native sub-issue, or a task-list checkbox on the parent |

Native sub-issues exist and are the better option where available (`gh api` on the sub-issues endpoint). Task lists are the portable fallback and still render a progress bar on the parent. Pick one per repo and record it — mixing them makes the hierarchy unreadable.

Since there's no type field, the type lives in a label from `typeLabels` plus the SAFe enabler labels from `conventions.md` § 7.

## Push

1. **Dedup first.**
   ```sh
   gh issue list --repo <repo> --state all --search "<key terms>" --limit 20 --json number,title,state
   ```
   Report near-matches and let the caller decide before creating anything.
2. **Milestone or tracking issue first**, if the draft has an epic. Capture its number.
3. **Stories next**, each linked to the parent.
4. **Cross-references last** — a comment naming the related issue is enough; GitHub backlinks it automatically.

```sh
gh issue create --repo <repo> \
  --title "<summary line>" \
  --body-file <tmp file holding everything above the tracker:end marker> \
  --label "<type label>" --label "<area>" --label "enabler" \
  --milestone "<epic>"
```

**Always `--body-file`, never `--body`.** Gherkin blocks contain backticks, quotes and newlines that a shell-quoted argument will mangle. Write the body region to a temp file and pass the path.

**The Gherkin survives natively** — a ```gherkin fence renders as a highlighted code block with no conversion step. This is the one place GitHub Issues is straightforwardly better than a rich-text tracker, so there is no post-push verification to do beyond eyeballing the first one.

**Attribution.** If the environment requires an AI-authorship disclosure on written-on-behalf content, append it as the last line of the body, naming the model actually running.

## Creating a milestone

```sh
gh api -X POST repos/<repo>/milestones -f title="<name>" -f state=open -f description="<one-line scope>"
```

A milestone with no description is how epics stop matching their contents — give it the same *outcome, not tasks* treatment a tracking issue would get.

## Stamp back

Update the draft in place: header table `**Issue**` → `[#123](https://github.com/<repo>/issues/123)`, `**Status**` → `PUSHED <date>`, the structure table's Issue column, and each story heading.

## Editing an existing issue

- `gh issue edit <n> --body-file <path>` replaces the whole body, so include the parts you're keeping.
- Don't rewrite someone else's wording; change only what was asked.
- `gh issue comment` for anything additive — an edit that buries the original text loses the thread's history.
