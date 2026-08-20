# Ticket conventions — SAFe + BDD

The rules a ticket has to satisfy before it goes to a tracker. Load this before drafting.

Nothing here knows which tracker you use. Tracker mechanics live in the adapter skills (`tracker-jira`, `tracker-github`); site specifics live in `.bdd-delivery.json` — see [`../../config.md`](../../config.md).

Sources: [SAFe BDD](https://framework.scaledagile.com/behavior-driven-development/) (login-gated), [Dan North's original](https://dannorth.net/blog/introducing-bdd/).

> BDD is "a test-first, Agile Testing practice that provides Built-In Quality by defining (and potentially automating) tests before or as part of specifying system behavior… a collaborative process that creates a shared understanding of requirements between the business and the Agile Teams."

Two things follow from that sentence and they're the ones people skip:

- **Test-first means the scenarios exist before the code**, not written up afterwards to describe what was built. A ticket refined after the branch is cut is documentation, not BDD.
- **Collaborative means more than one head.** See § 5.

---

## 1. Who owns which ticket

The problem this solves: *"our BAs don't write low-level technical tickets."* In SAFe they're not supposed to.

| SAFe work item | Owner | Example |
|---|---|---|
| **Business Story** | PO / BA | "customer can skip a delivery from the app" |
| **Enabler Story — Exploration** | the team | a spike: can we trigger off change-tracking on a vendor-owned table? |
| **Enabler Story — Architecture** | the team / System Architect | consolidate two copies of a provider into one package |
| **Enabler Story — Infrastructure** | the team | move IP allowlists to service tags; migrate a service to a new runtime |
| **Enabler Story — Compliance** | the team | remove a deprecated SDK; rotate a leaked key |

**Decision rule.** Try to write the clause `so that <business value>`. If you can write it honestly, it's a Business Story and the BA owns the wording — go get it. If you have to invent the value, it's an **Enabler**. Write it yourself, label it, stop waiting.

Enablers are first-class SAFe backlog items with their own capacity allocation. "It's just tech debt so it doesn't get a ticket" is the anti-pattern this framing exists to kill.

---

## 2. Hierarchy

SAFe's levels, and how they land in a tracker:

| SAFe | Typical tracker equivalent | When |
|---|---|---|
| Epic (portfolio) | Jira Initiative · GitHub milestone or tracking issue | multi-quarter, multiple teams. Rare. |
| Capability | often not modelled | fold upward |
| Feature | Jira Epic · GitHub tracking issue | a coherent outcome, weeks of work, several stories |
| Story | Jira Story · GitHub issue | one PR-sized change with its own acceptance criteria |
| — | Jira Sub-task · GitHub task-list item | a mechanical slice of one story with no AC of its own |

**Most trackers don't model every level.** Map down and record the mapping in config (`hierarchy`) rather than inventing levels the tracker can't hold. Where a level is missing, fold into the one above and say so in the ticket.

A **Bug** still gets Gherkin: one scenario reproducing the defect (currently failing), plus the scenarios that must stay green.

---

## 3. Ticket anatomy

### Summary line

Imperative, specific, ≤ 80 chars, names the subject. It should read as the PR title.

- ✅ `Replace delete-and-reinsert with an idempotent upsert for orders`
- ✅ `Alert when the inbound feed cannot ingest a row`
- ❌ `Sync improvements` — which improvement?
- ❌ `Fix bug in handler` — which handler, which bug?

### Body sections, in order

| Section | Business Story | Enabler | Bug |
|---|---|---|---|
| **Value** — user-voice `As a … I want … so that …` | required | **omit** | omit |
| **Why now** — the pain, with evidence | optional | **required** | required |
| **Context** — what exists today, real paths and types | required | required | required |
| **Acceptance criteria** — one Gherkin `Feature:` | required | required | required |
| **Out of scope** — the adjacent thing you're not doing | if ambiguous | if ambiguous | if ambiguous |
| **Notes** — traps, sequencing, decisions to record | optional | optional | optional |
| **Open questions** — blockers a reviewer must answer | if any | if any | if any |

**Do not put user-voice on an enabler.** "As a developer I want a version column so that I can detect conflicts" is theatre. State it technically and justify it under **Why now**.

**Why now** must carry evidence: a count, a date, an incident reference, a PR number, a production symptom. "It's messy" is not evidence. This is the section that gets the ticket prioritised.

---

## 4. Gherkin rules

### Shape

1. **One `Feature:` per ticket.** Name it after the capability being guaranteed, not the change being made — `Feature: Idempotent upsert of orders`, not `Feature: Change the handler`.
2. **3–8 scenarios.** Fewer than 3 and it's probably a sub-task. More than 8 and the story is too big — split it and say so.
3. **`Background:` only when 3+ scenarios share an identical `Given`.** Otherwise it hides context.
4. **`Scenario Outline` + `Examples` for table-driven variation.** Never five near-identical scenarios.

### Grammar

5. **Scenario names are declarative statements of the guaranteed behaviour**, readable aloud in refinement.
   - ✅ `Scenario: A stale message is ignored`
   - ❌ `Scenario: Test stale message handling` — it's not a test name
   - ❌ `Scenario: Should ignore stale messages` — see rule 6
6. **Present indicative. No "should", "will", "must".** `Then the row is updated`, not `Then the row should be updated`. "Should" invites argument about whether it did.
7. **One `When` per scenario.** Two `When`s means two scenarios, or your `Given` is under-specified.
8. **`Given` = state that already holds. `When` = the single event under test. `Then` = an outcome an observer could check without reading the source.**
9. **`And` / `But` continue the previous keyword.** Never open a scenario with `And`.
10. **No pronouns.** Name the thing every time. "it" in a Then clause is where ambiguity hides.

### Content

11. **Declarative, not imperative.** Describe the guarantee, not the click path or the method call. No UI steps. No HTTP verbs unless the API contract *is* the behaviour under test.
12. **Specification by example — be concrete.** `Given rows exist for items A, B and C` beats `Given some rows exist`. Concrete examples are what expose the disagreement in refinement.
13. **Every scenario must be falsifiable.** You must be able to state what a failing run looks like. If you can't, it's a wish.
14. **Non-functional criteria carry a number or don't exist.** `Then the p95 stays under 200 ms`, not `Then it is fast`.

### The four coverage checks

These are the recurring failure modes of message-driven and distributed systems. Walk the list on every ticket and include the ones that apply:

| Check | Applies when | Scenario to write |
|---|---|---|
| **Failure path** | always | what happens when it breaks — who finds out, and how |
| **Replay / idempotency** | queues, topics, timers, webhooks, anything at-least-once | processing the same message twice has the same effect as once |
| **Silence on success** | anything that alerts | `Given a healthy run / Then no alert is raised` — otherwise you build a pager that cries wolf |
| **Deploy order** | package changes, schema migrations, contract changes | old consumer + new package (or new schema + old build) still works |

Missing the replay scenario is how you end up doing dead-letter archaeology. Missing "silence on success" is how alerting gets muted.

### Two legitimate exceptions

15. **Architecture enablers may name types and members.** For a purely structural change the code shape *is* the observable behaviour:
    ```gherkin
    Then the entity configuration maps the column as a concurrency token
    ```
    Don't force fake user-facing language onto it. Business Stories get no such licence.

    **A structural scenario may drop the `When`.** Rule 7 assumes there is an event under test. A claim about the shape of the code or the schema has no event, and inventing one produces padding — `When the schema is inspected`, `When the change is reviewed`, `When the code is read`. Those are not behaviour, they are a `Then` wearing a costume. Write the assertion directly:
    ```gherkin
    Scenario: The unique constraint enforces the contract
      Given the inbox table
      Then a unique index covers subscriber, event type and business key where the row is valid
    ```
    Where a real trigger *does* exist, name it — `When the model is built` is a genuine event with a genuine failure mode (the build throws), so it stays. The test is whether the `When` could fail on its own. If it can't, cut it.

16. **Exploration enablers (spikes) assert a decision, not an implementation.**
    ```gherkin
    Then the spike outputs a go/no-go with reasoning
    And if go, a follow-up story with the migration scope
    And if no-go, the alternative with its cost
    ```
    A spike's Definition of Done is a documented decision plus a follow-up ticket. If a spike's scenarios describe working code, it isn't a spike — reclassify it.

### Reject on sight

| Anti-pattern | Why |
|---|---|
| `Then the code is refactored` | not observable |
| `Then the tests pass` | circular |
| `Then the developer can …` | the developer isn't a runtime actor — unless the deliverable genuinely is a dev tool |
| `Given the system is working` | noise |
| A scenario that restates the summary | says nothing |
| `Then it works as expected` | expected by whom, checked how |

---

## 5. Three Amigos

BDD is collaborative by definition. Before a ticket is Ready, three perspectives have seen the scenarios:

| Seat | Normally | On an enabler |
|---|---|---|
| **Business** | BA / PO | whoever owns the operational pain — support, ops, the on-call who got paged |
| **Development** | the engineer | same |
| **Test** | QA | same |

Record in the draft's `## Refinement` block: which seats have been covered, and the single question to put to each empty seat. Claude counts as **none** of the three.

**There are exactly three seats, and they hold roles, not names.**

- **Don't add a fourth.** The Three Amigos is three *perspectives*, not a stakeholder list. A standards owner, an architect, a security reviewer — all legitimate reviewers, none of them a fourth amigo. Adding a row for them is a category error that quietly turns the check into a sign-off queue.
- **Don't write people's names in the seats.** Names go stale the moment someone changes role, and they turn a required perspective into one person's opinion — which is exactly what makes the seat skippable when that person is on leave. Record the seat and whether it's covered.
- One person can hold two seats. That's normal on a small team, and it's still worth recording as two, because the two questions are different.

---

## 6. Definition of Ready

Check the draft against this and record the result. Failing is fine; hiding the failure isn't.

**INVEST**

- **I**ndependent — can ship without another unmerged story? If not, name the dependency explicitly.
- **N**egotiable — describes the *what*, leaves room on the *how*. A ticket that prescribes the implementation line by line has skipped refinement.
- **V**aluable — Value (business) or Why now (enabler) stands up to "so what?"
- **E**stimable — could the team size it today? If a fact is missing, that's an Open question, and possibly a spike.
- **S**mall — one PR's worth. 3–8 scenarios is the proxy.
- **T**estable — every scenario is falsifiable.

**Plus**

- No duplicate exists (you searched the tracker, and say so).
- Every claim in Context has a source: file path, PR number, incident reference, or a query someone else can re-run.
- Every open question is either answered or explicitly deferred with an owner.
- Blocked-on items are named. A ticket that can't start yet says why on its face.

---

## 7. Labels

Labels are a filter, not a description — three or four, not nine. Lowercase-kebab. No new label without a second ticket that would also use it.

The taxonomy is site-specific and comes from config (`labels.area`, `labels.kind`, `labels.always`). Two conventions worth keeping wherever you are:

- **A team label on everything the team owns**, so the team's work is one query.
- **SAFe enabler labels**: `enabler` plus one of `enabler-exploration`, `enabler-architecture`, `enabler-infrastructure`, `enabler-compliance`.

The enabler labels matter more than they look. The classification is worthless in a description alone — SAFe wants it *filterable*, because capacity allocation for enabler work is the argument you have to win with numbers.

---

## 8. Handing off to tests

The scenarios are the test plan. That's the payoff.

The `tests-from-ticket` skill executes this section — it reads a ticket's Gherkin, classifies each scenario by what can actually reach it, and scaffolds the tests. Use it rather than transcribing by hand.

**Mapping**

| Gherkin | Test |
|---|---|
| `Feature:` | the test fixture / suite |
| `Scenario:` | one test |
| `Scenario Outline` + `Examples` | one parameterised case per row |
| `Given` | arrange |
| `When` | act |
| `Then` | assert |

**Test names come from the scenario, mechanically.** The pattern is site-specific (config: `tests.naming`) — but whatever it is, apply it without thinking, and if the resulting name doesn't read back as the scenario, rename the test. Two that work:

| Pattern | `Scenario: A stale message is ignored` becomes |
|---|---|
| `Given<given>_Should<then>` | `GivenAStaleMessage_ShouldNotModifyTheRow` |
| `<subject>_<condition>_<outcome>` | `Upsert_StaleMessage_IsIgnored` |

If the house pattern uses "should" in test names while rule 6 bans it in Gherkin, that is not a contradiction: a scenario asserts what the system *does*, a test name is a claim about what the code under test should do. Don't "fix" it in either direction.

**Stacked PRs**

- **PR 1 — tests.** One test per scenario, each skipped/ignored with the ticket reference as the reason. Reviewers review *the specification*, and CI stays green so PR 1 is mergeable on its own.
- **PR 2 — implementation.** Removes the skip markers; the tests go green.

**The rule that makes the stack worth doing: in PR 2, the only permitted change to a test file is deleting a skip marker.** Anything else — a changed assertion, a renamed test, a loosened expectation — means the acceptance criteria were wrong. Update the ticket, amend PR 1, then continue. Without this the tests just get bent to fit whatever got built, and the whole exercise is theatre.

Stated that precisely because it's mechanically checkable. With `tests.skipMarker` = `Ignore` and `tests.glob` = `**/*Test*.cs`:

```powershell
git diff <pr1-base>..HEAD -- '**/*Test*.cs' | Select-String '^[+-]' | Where-Object { $_ -notmatch '^\-\s*\[Ignore' }
```

Empty output means PR 2 is clean.

**On executable Gherkin.** Cucumber, SpecFlow/Reqnroll and friends bind `.feature` files directly to step definitions. That is a real option and this workflow deliberately does not assume it: Gherkin stays in the ticket, your existing test framework stays in the repo, and the scenario→test-name mapping is the link. Adopting a binding framework is a separate, deliberate decision — not a side effect of writing acceptance criteria well.
