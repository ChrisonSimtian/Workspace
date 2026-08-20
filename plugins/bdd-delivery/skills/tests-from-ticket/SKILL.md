---
name: tests-from-ticket
description: Turn a ticket's Gherkin acceptance criteria into tests — one test per scenario, named from the scenario, skipped so PR 1 of a stacked pair stays green. Classifies each scenario as unit / integration / infra / not-mechanisable rather than faking a test for criteria a unit test cannot reach. Trigger when the user says "tests from ticket", "scaffold the tests for <key>", "write PR 1", "turn the AC into tests", or names a ticket and asks for its tests.
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, Bash, PowerShell, Skill
argument-hint: [issue key | path to a ticket draft]
---

# Tests from ticket

The second link in the chain. `ticket` produces Gherkin; this turns it into PR 1 of a stacked pair.

```
ticket (Gherkin acceptance criteria)
        │
        ▼   this skill
PR 1 — one skipped test per scenario, red-by-design, CI green
        │
        ▼   implementation
PR 2 — deletes the skip markers and nothing else in a test file
```

**Read [`../ticket/conventions.md`](../ticket/conventions.md) § 8 first.** It owns the scenario→test-name mapping and the stacked-PR contract. This skill executes it; it does not redefine it.

## Step 0 — Load the site config

Read `.bdd-delivery.json` (repo root, else `.claude/`) for `tests.*` — framework, naming pattern, skip marker, glob, test-project pattern, and the path to any site test-conventions doc. Load `overlay` if declared.

Infer what's missing from the repo rather than asking: the test framework from the csproj/package.json/go.mod, the skip marker from the framework, the project layout from where existing tests live.

## Step 1 — Get the Gherkin

**From an issue key**: ask the tracker adapter (`tracker-jira` / `tracker-github`) to fetch it. Extract the fenced ```gherkin block.

**From a draft file** under `draftPath`: read it directly. Prefer this when it exists — same content, no round trip.

Parse out, per scenario: the name, the `Given`/`When`/`Then` steps, and any `Scenario Outline` + `Examples` table. Keep `Background:` separately — it becomes shared setup, not a test.

If the ticket has no Gherkin block, stop and say so. A sub-task legitimately has none — that is not a bug, there is just nothing here to do.

## Step 2 — Classify every scenario, before touching the repo

**This is the step that makes the skill honest, and it comes before any work.** Not every scenario becomes a unit test, and pretending otherwise is worse than writing nothing. A fabricated test that passes against a fake while the real behaviour is untested is a false green — it actively removes the pressure to test the thing properly.

| Class | Meaning | Action |
|---|---|---|
| **unit** | reachable with the normal test harness, doubles and in-memory fakes | write the test |
| **integration** | needs real infrastructure — a real database, broker, clock, or genuine concurrency | write it into the report, not the fixture; raise a follow-up if no integration suite exists |
| **infra** | satisfied by configuration — an alert rule, a pipeline setting, an IaC change | note it; there is no test in this repo to write |
| **not-mechanisable** | deploy-order and backwards-compatibility scenarios, "an operator can…", spike decisions | note it; verified by a human or by the deploy, and that is fine |

**If no scenario classifies as unit, stop here and report.** Skip straight to Step 7. Do not go looking for a test project you have no reason to touch — a spike whose every `Then` reads *the spike states…* has no system under test, and the correct output is zero tests plus the reason why. Emitting skipped stubs so the run looks productive is the exact failure this step exists to prevent: they show as coverage on a ticket that has none.

### What forces a scenario out of "unit"

Check each against the harness you actually have, not the one you wish you had:

- **In-memory and fake data stores usually don't enforce constraints.** Unique indexes, check constraints, cascades — commonly unenforced. Any scenario asserting "the constraint rejects this" is integration.
- **Optimistic concurrency needs a real store.** Row-version / ETag conflict scenarios can't fire against a fake that has no version column semantics.
- **"Never absent mid-transaction" needs two real connections.** Isolation behaviour is not simulable.
- **"An alert is raised" may not be code at all.** Check the ticket's Notes before assuming there's a collaborator to assert on — it may be a log-based alert rule, in which case it's infra.
- **Time and scheduling** need an injectable clock. If the code calls the system clock directly, the scenario is untestable until a seam exists — say so; don't add the seam yourself.

## Step 3 — Locate the test project

Only for the scenarios that survived Step 2 as unit.

Use `tests.projectPattern` if configured. Otherwise find the production project from the types named in Context, then locate its test project by searching for the nearest existing test that imports it.

Confirm with `Glob` rather than assuming — layouts vary between repos in the same organisation. If no test project exists, say so and ask before creating one; a new test project needs a manifest, a solution/workspace entry and a CI reference, which is a bigger change than scaffolding tests.

## Step 4 — Read a sibling before writing

**Mandatory. Never generate a fixture blind.** Read `tests.conventionsDoc` if configured, then the closest existing test in the target project, and match it: base class, setup shape, double/mock style, builder helpers, assertion library, naming.

What to look for, because these are what a generic generator gets wrong:

- The **base class or fixture harness** and what it provides for free.
- Whether the subject-under-test accessor is **cached or re-created per access** — a fixture that returns a fresh instance each read will silently break a test that touches it twice.
- Existing **builder helpers** (`AnOrder(...)`, `aUser()`, factories). Add to these rather than inlining setup.
- Assertion library and style.

**While you are in there, check which scenarios are already covered.** Read every existing test name against the scenario list before writing anything. A ticket written before its sibling shipped will restate behaviour that now exists; writing it again produces duplicate tests and hides the fact that the ticket has shrunk. Report already-covered scenarios with file and line, and write nothing for them.

## Step 5 — Write the tests

One test per unit-class scenario, named per `tests.naming`.

Rules for the generated body:

- **Every `Given`/`When`/`Then` line appears as a comment above the code implementing it.** That is the traceability — a reviewer diffs the test against the ticket by eye, with no tooling.
- **Write real assertions, not a bare fail.** The test must fail because the behaviour is missing, not because it is a stub. A stub proves nothing and gets deleted in PR 2 instead of passing.
- **The skip marker carries the ticket reference** in exactly the configured form, so PR 2's diff is greppable and the check in `conventions.md` § 8 works.
- `Scenario Outline` + `Examples` → one parameterised test, one case per row, values in the same column order.
- `Background:` → shared setup, appended to what exists rather than replacing it.
- If a needed production type or member does not exist yet, **reference it anyway** and say in the report that PR 1 will not compile until PR 2 adds it. Do not invent a placeholder type. Where PR 1 must compile standalone, note that the test has to move to PR 2 instead, and flag the departure from the stack.
- **A scenario that already passes is a finding, not a mistake.** Keep the test as a regression guard, leave the skip marker off, and say in the report why it is there — otherwise someone deletes it later as redundant.

## Step 6 — Verify the tests actually fail

Run the new tests with the skip markers temporarily removed, so "red" is a fact rather than an assumption. Three outcomes, each meaning something different:

- **Fails on the assertion** — correct. Restore the skip marker, done.
- **Passes** — the behaviour already exists, or the assertion is too weak to distinguish. Investigate before shipping; a scenario that passes before implementation is either already-satisfied (say so, it may shrink the ticket) or wrong.
- **Fails to compile** — expected when PR 2 adds the type. Record it; don't weaken the test to make it compile.

Restore every skip marker before reporting, and re-run to confirm the suite is green.

## Step 7 — Report

- Mapping table: scenario → test name → class (unit / integration / infra / not-mechanisable).
- **Scenarios with no automated coverage, and why.** Lead with this; it is the finding.
- Already-covered scenarios, with file and line.
- The verify result per test (fails-on-assertion / passes / does-not-compile).
- Anything suggesting the ticket is wrong — a scenario that passes already, one that can't be made falsifiable, one that needs splitting. The ticket gets fixed first, not the test bent to fit.

When Step 2 short-circuited, the report is just the mapping table and the reason. That is a complete, successful run — say so plainly rather than apologising for writing nothing.

## Notes

- **Don't touch production code.** This skill writes tests. If a scenario can't be tested without a seam that doesn't exist, that observation goes in the report and the seam is PR 2's job.
- **Don't test the test harness.** Fixtures, doubles and builders are exercised by their consumers.
- **Never weaken a scenario to make it testable.** If the Gherkin is untestable as written, the acceptance criteria are wrong — say so and fix the ticket.
- **Out of scope**: creating test projects, raising the PR, and building integration-test infrastructure that doesn't already exist.
