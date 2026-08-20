# Draft file templates

Two shapes. Both live at `<draftPath>/<YYYY-MM-DD>-<slug>.md`.

**The `## Refinement` block is never pushed.** Everything above the `<!-- tracker:end -->` marker is the issue body, verbatim. Everything below is a working aid.

---

## A. Single ticket

```markdown
# <Summary line — imperative, ≤80 chars, reads as the PR title>

| | |
|---|---|
| **Status** | DRAFT |
| **Type** | Story · Business \| Enabler (Architecture) \| Bug |
| **Parent** | <parent key, or —> |
| **Labels** | `<team>`, `<area>`, `enabler`, `enabler-architecture` |
| **Repos** | <repo>, <repo> |
| **Issue** | <stamped at push> |

<!-- tracker:start -->

## Value            ← business stories only; delete for enablers

As a <role>, I want <capability>, so that <business value>.

## Why now          ← enablers and bugs; required, must carry evidence

<The pain, with numbers, dates, incident references, PR numbers.>

## Context

<What exists today. Real file paths, real type names. Where the behaviour lives
and why it's wrong. Links to incidents, PRs, prior tickets, plan docs.>

## Acceptance criteria

```gherkin
Feature: <the capability being guaranteed>

  Scenario: <declarative statement of behaviour>
    Given <state that already holds>
    When <the single triggering event>
    Then <observable outcome>
    And <observable outcome>

  Scenario: <failure path — what happens when it breaks>
    ...

  Scenario: <replay / idempotency, if it touches messaging>
    ...

  Scenario: <a healthy run is quiet, if it alerts>
    ...

  Scenario: <deploy order, if it ships a package or a migration>
    ...
```

## Out of scope

- <the adjacent thing a reader would assume is included, and isn't>

## Notes

- <traps, sequencing, decisions to record, things to verify before estimating>

## Open questions

1. <blocker a reviewer has to answer, with who should answer it>

<!-- tracker:end -->

## Refinement            ← NOT pushed

**Three Amigos** — three seats, no more, roles not names

| Seat | Covered | Question for the seat |
|---|---|---|
| Business | no | <the one question> |
| Development | yes | — |
| Test | no | <the one question> |

**Definition of Ready**

| | |
|---|---|
| Independent | ✅ / ⚠️ depends on <X> |
| Negotiable | ✅ |
| Valuable | ✅ |
| Estimable | ⚠️ blocked on <fact> |
| Small | ✅ 6 scenarios |
| Testable | ✅ |
| No duplicate | ✅ searched `<query>`, nothing since <date> |
| Claims sourced | ✅ |

**Verdict:** Ready / Not ready — <one line>
```

---

## B. Epic + story breakdown

Same rules per story; one file for the set.

```markdown
# <Epic name> — epic & story breakdown

| | |
|---|---|
| **Status** | DRAFT |
| **Source** | [`<plan>.md`](<path>) |
| **Issue** | <stamped at push> |

## Structure at a glance

| # | Story | Type | Issue |
|---|---|---|---|
| 1 | <summary> | Enabler (Architecture) | — |
| 2 | <summary> | Enabler (Exploration) | — |

**Judgement calls worth challenging**

- <the split you made and the argument against it>

**Dependencies:** <S2 → S1, and what a cruder S1-free version would look like>

<!-- tracker:start:epic -->

# Epic — <name>

**Type:** Epic · **Labels:** ...

## Description

<The outcome, not the tasks. What is true when this epic is done that isn't now.>

## Out of scope

- <adjacent work, with a pointer to where it does live>

<!-- tracker:end:epic -->

<!-- tracker:start:story-1 -->

## Story 1 — <summary>

**Type:** Story · **Epic:** <name> · **Labels:** ...

### Why now

### Context

### Acceptance criteria

```gherkin
Feature: ...
```

### Notes

<!-- tracker:end:story-1 -->

...

## Open decisions before we push

1. <the thing the author has to call>

## Refinement            ← NOT pushed

<amigos + DoR table per story, or one table if they share a verdict>
```

---

## Stamping keys back

At push time, edit in place — don't rewrite the file:

- header table `**Issue**` → `[KEY](url)`
- `**Status**` → `PUSHED <YYYY-MM-DD>`
- structure table `Issue` column → the key per row
- each story heading → `## Story 1 — <summary> · [KEY](url)`

The draft stays in the repo afterwards. It's the diffable history of how the ticket was worded, which the tracker doesn't give you.
