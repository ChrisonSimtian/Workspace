---
name: ticket
description: Draft tickets in SAFe/BDD shape with Gherkin acceptance criteria, review them as a version-controlled markdown draft, then push to a tracker on approval. Two modes — a single Story/Bug/Enabler from an idea, incident, PR or chat thread; or an Epic + Story breakdown from a plan doc. Tracker-agnostic; hands off to a tracker adapter skill. Trigger when the user says "ticket", "write a ticket", "raise a ticket for this", "break this down into tickets", "turn this plan into an epic", "acceptance criteria", or "push the tickets".
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash, PowerShell, Skill
argument-hint: [idea | issue key | PR url | plan path | "push <draft>"]
---

# Ticket

Turn a piece of work into a ticket whose acceptance criteria are BDD scenarios, so the same scenarios drive the tests and the tests gate the PR.

```
conversation / plan / incident
        │
        ▼   this skill, draft mode
<draftPath>/<slug>.md          ← reviewable, diffable, version-controlled
        │
        ▼   this skill, push mode → tracker adapter skill
tracker issue (Epic → Story), Gherkin in the body
        │
        ▼   PR 1   (tests-from-ticket skill)
skipped tests, one per Scenario
        │
        ▼   PR 2
implementation — only skip markers removed from test files
```

**Read [`conventions.md`](./conventions.md) before drafting anything.** It holds the ticket anatomy, the Gherkin rules, the SAFe hierarchy mapping and the Definition of Ready. Don't write scenarios from memory — those rules are the whole point of the skill.

[`templates.md`](./templates.md) has the two draft-file shapes. [`../../config.md`](../../config.md) documents the site config.

## Step 0 — Load the site config

Read `.bdd-delivery.json` (repo root, else `.claude/`). It names the tracker, the draft path, the label taxonomy, the hierarchy names and the test conventions.

If it declares an `overlay`, load that too — it carries site-only rules this skill must not hardcode. **The overlay wins on anything it covers.**

No config file is fine. Fall back to `conventions.md` defaults and ask about the tracker only if a push is actually requested.

## Modes

Pick from the argument; ask via `AskUserQuestion` only if genuinely ambiguous.

| Mode | Trigger | Output |
|------|---------|--------|
| **story** | an idea, a bug, an incident, a PR url, a chat link, "raise a ticket for X" | one Story / Enabler Story / Bug in a new draft file |
| **breakdown** | a path to a plan doc, "break this down", "turn this into an epic" | one Epic + N Stories in a single draft file |
| **push** | "push", "send it", plus a draft path | tracker issues created via the adapter, keys stamped back into the draft |

## Step 1 — Gather the source material

The ticket is only as good as the evidence behind it. Before writing:

- **Read the actual code.** Name real files, real types, real line references. A ticket that says "the handler" instead of the class name is not finished.
- **Pull the numbers.** Message counts, exception counts, dates, incident references. "N failures over three days, against a baseline of zero" beats "this happens a lot", and it is the difference between a ticket that gets prioritised and one that doesn't.
- **Find the prior art.** Search the tracker for an existing ticket before creating a new one — ask the adapter to search. If something close exists, say so and offer to extend it. Duplicate tickets are worse than no ticket.
- **PR url** → `gh pr view <url> --json title,body,files`.
- **Chat thread** → read it if a tool is available; otherwise ask for a paste.

## Step 2 — Classify

Two questions, in this order. Both are in `conventions.md` in full.

**Business Story or Enabler?** Try to write the `so that <business value>` clause. If you have to invent the value, it's an Enabler — classify it and write it yourself rather than waiting on a BA. Then pick the type: Exploration (spike), Architecture, Infrastructure, or Compliance.

**What level?** Map SAFe's levels onto the names in `hierarchy`. Don't invent a level the tracker can't hold.

## Step 3 — Draft the file

Write to `<draftPath>/<YYYY-MM-DD>-<slug>.md` using the matching template.

The draft is a **superset** of what goes to the tracker. Everything under `## Refinement` is a working aid and is stripped at push time. Everything above it is pushed verbatim.

Then run the Definition of Ready checks against your own draft and record the result in `## Refinement`. Be honest — a draft that fails INVEST should say so and say which letter.

## Step 4 — Show it and stop

Paste the full draft back. **Do not push.** A tracker is shared, and a badly-shaped ticket is public.

Lead with anything needing a decision — an ambiguous key, an unverified assumption, a duplicate you found, a scenario you couldn't make falsifiable. Those go in `## Open questions` in the ticket body if the reviewer needs them too, and in `## Refinement` if they're only for the author.

## Step 5 — Push on approval

Only after explicit approval. Delegate to the adapter named by `tracker`:

```
Skill: tracker-jira      # or tracker-github
```

Pass it the draft path. The adapter owns field mapping, hierarchy names, parent wiring, and the search/dedup query. **This skill does not call a tracker API directly** — if you find yourself reaching for one, the adapter is missing something and that's where the fix belongs.

Order matters and the adapter enforces it: Epic first, capture its key; Stories next, parented to it; links last.

Two rules the adapter must honour, restated here because they are easy to lose:

- **Never write to a service-desk / customer-facing project** (`jira.serviceDeskProjects`, or a GitHub repo with public issues where the reporter is a customer). A comment there can reach a customer.
- **Verify the first issue of a run.** Fetch it back and check the Gherkin block survived the markdown conversion. If it came through mangled, fix the encoding before creating five more.

Then stamp the keys back: update the draft's header table and each story heading with its key and url, and set `**Status:** PUSHED <YYYY-MM-DD>`.

## Step 6 — Report

- Every key created, with url.
- Anything you deliberately did **not** create and why (duplicate found, blocked on a decision).
- The next action: normally "PR 1 is the tests" — hand off to `tests-from-ticket`.

## Notes

- **One ticket per PR is the goal, not one per commit.** If the work is genuinely one PR, it's one story. Resist sharding a story into sub-tasks that each need their own branch.
- **A spike delivers a decision, never code.** If an Exploration enabler's scenarios describe a working implementation, reclassify it.
- **Attribution.** If your environment requires an AI-authorship disclosure on written-on-behalf content, the adapter appends it. Configure it there, not here.
- **Out of scope for this skill**: estimating, sprint assignment, transitioning issues, and anything in a customer-facing project.
