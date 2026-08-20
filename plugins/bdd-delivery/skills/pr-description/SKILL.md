---
name: pr-description
description: Draft a pull request description from the branch diff and its ticket's Gherkin, classify the PR as the test half or the implementation half of a stacked pair, and mechanically check that an implementation PR only removes skip markers from test files. Trigger when the user says "PR description", "write the PR", "raise the PR", "describe this PR", "PR body", or asks to open a pull request for the current branch.
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash, PowerShell, Skill
argument-hint: [ticket key | PR number | nothing, to use the current branch]
---

# PR description

The last link in the chain. `ticket` produced the Gherkin, `tests-from-ticket` turned it into PR 1; this describes whichever PR you're raising and enforces the contract between them.

```
ticket (Gherkin acceptance criteria)
        │
        ▼   tests-from-ticket
PR 1 — skipped tests, one per Scenario     ← this skill describes it
        │
        ▼   implementation
PR 2 — skip markers removed, nothing else  ← this skill checks that claim
```

**Read [`../ticket/conventions.md`](../ticket/conventions.md) § 8** for the stacked-PR contract. This skill enforces it; it does not restate it.

## Step 0 — Load the site config

Read `.bdd-delivery.json` for `tests.glob`, `tests.skipMarker`, `pr.*` and `overlay`. Load the overlay if declared — PR labels, required preflight commands and any attribution requirement are site-specific and live there.

## Step 1 — Resolve the ticket

In order of preference:

1. **The branch name.** `feat/MFB-1234-thing`, `MFB-1234/thing`, `1234-thing` — extract the key against the project prefix in config.
2. **An argument.** The user named a key or a PR number.
3. **The commit messages** on the branch.
4. **Ask.** Don't invent a reference, and don't raise a PR with no ticket silently — if there genuinely isn't one, say so and let the user decide.

With the key, fetch the ticket through the tracker adapter and extract its ```gherkin block. You need the scenario names; you don't need the rest.

## Step 2 — Classify the PR, and check the contract

Get the diff against the base branch, then decide which half of the stack this is:

| Diff contains | Shape | Description says |
|---|---|---|
| test files only | **PR 1 — specification** | these tests are expected to fail until the implementation PR lands |
| production files, and the only test-file change is deleting skip markers | **PR 2 — implementation** | which scenarios go green |
| production files, no test files, no prior test PR | **single PR** | scenario coverage, and why it wasn't stacked |
| production files **and** substantive test changes | **contract violation** — see below |

### The mechanical check

For a PR 2, the only permitted change to a test file is deleting a skip marker. That is checkable, so check it rather than asserting it:

```bash
git diff "$BASE...HEAD" -- '**/*Test*.cs' \
  | grep -E '^[+-]' \
  | grep -Ev '^(\+\+\+|---)' \
  | grep -Ev '^-[[:space:]]*\[Ignore'
```

```powershell
git diff "$base...HEAD" -- '**/*Test*.cs' |
  Select-String '^[+-]' |
  Where-Object { $_ -notmatch '^(\+\+\+|---)' -and $_ -notmatch '^-\s*\[Ignore' }
```

Substitute `tests.glob` and `tests.skipMarker` from config. **Empty output means the PR is clean.**

The two `grep -Ev` filters both matter: the first drops the `+++ b/…` / `--- a/…` file headers, which are not content and would otherwise report a violation on every single PR 2.

**When the check finds something, do not quietly write it into the description as though it were normal.** Report it, name the files and lines, and say what `conventions.md` § 8 says: a changed assertion, a renamed test or a loosened expectation means the acceptance criteria were wrong. The fix is to update the ticket and amend PR 1 — not to describe the drift nicely. Offer to draft the description anyway if the user decides the change is legitimate, but make them decide.

## Step 3 — Read recent merged PRs before writing

**Mandatory, same principle as the rest of this plugin: match what's there.** Read the last three to five merged PRs in the target repository and copy their shape.

```bash
gh pr list --state merged --limit 5 --json number,title,body
```

Look for:

- **Section headings actually in use.** Repos converge on their own — `## What` / `## Why`, or `#### What does this PR do?`, or a problem/fix pair. Use theirs, not a generic template.
- **How the ticket is referenced** — a bare key, a link, a link plus the ticket title, `Fixes …`.
- **Whether there's a reviewer-orientation section.** Some repos have a "where should the reviewer start" convention, which is worth more than any other section and is easy to miss.
- **How out-of-scope work is called out.** "Deliberately not in this PR" phrasing is common and load-bearing.
- **Whether siblings are cross-referenced** by PR number.

If `pr.template` is configured, that wins over inference.

## Step 4 — Draft

Whatever the house shape, the content owes the reader four things:

1. **The ticket**, linked, with its title — so a reviewer knows the intent without leaving the page.
2. **What changed and why**, in prose. Not a file list; the diff is right there.
3. **Where to start reading.** Name the entry-point file or type. This is the single highest-value sentence in a PR description.
4. **The scenario mapping** — which acceptance-criteria scenarios this PR satisfies, by name, and which it deliberately doesn't.

Then, by shape:

- **PR 1** — state that the tests are expected to fail, that they carry skip markers so CI stays green, and that the PR is a review of the *specification*. List any scenario that got no test and why, from `tests-from-ticket`'s classification.
- **PR 2** — list the scenarios that go green, and state the result of the mechanical check explicitly. A PR 2 that says "test diff clean: skip markers only" has earned the reviewer's trust cheaply.
- **Single PR** — say why it wasn't stacked. "Behaviour already covered by existing tests" is a fine reason; "forgot" is worth knowing too.

Add anything from `pr.preflight` that the site requires — a version bump, a changelog entry, a generated-client refresh. Run them if they're safe to run and report the result; don't claim a preflight passed without running it.

**Never describe a change the diff doesn't contain.** The most common failure here is writing the description from the ticket rather than the diff, so it describes intended work rather than delivered work.

## Step 5 — Show it and stop

Paste the draft. **Do not raise or edit the PR yet.** Lead with the contract-check result if it failed, and with anything the ticket claims that this PR doesn't deliver.

## Step 6 — Raise or update on approval

```bash
gh pr create  --title "<title>" --body-file <tmp>  [--label … --draft]
gh pr edit <n> --body-file <tmp>
```

**Always `--body-file`, never `--body`.** Bodies contain backticks, quotes and newlines that shell quoting mangles.

Titles follow the repo's existing convention — read them in step 3 rather than assuming Conventional Commits. Apply labels from the overlay. If the site requires an authorship disclosure, append it as the last line.

## Step 7 — Report

- The PR url.
- The contract-check result.
- Scenarios covered, and scenarios deliberately not covered.
- Any preflight that failed or that you didn't run.

## Notes

- **This skill describes; it does not review.** If the repo has a review skill, that's a separate pass and a separate voice.
- **Don't touch the code to make the description truer.** If the diff and the ticket disagree, that's the finding.
- **A draft PR is a legitimate output.** If the contract check failed or a preflight didn't pass, raising as a draft and saying why beats either blocking or pretending.
- **Out of scope**: merging, requesting reviewers, and rewriting commit history.
