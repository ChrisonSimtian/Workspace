# Site configuration

Everything site-specific lives in one file so the skills stay portable. Put it at `.bdd-delivery.json` in the repo root, or `.claude/bdd-delivery.json` if you prefer to keep the root clean. The skills look for both, root first.

Nothing here is required. Every key has a fallback, and a repo with no config file still works — you just get asked more questions and generic defaults.

```json
{
  "tracker": "jira",
  "draftPath": "docs/tickets",
  "overlay": "acme-tools:acme-ticket",

  "hierarchy": {
    "portfolioEpic": "Initiative",
    "feature": "Epic",
    "story": "Story",
    "subTask": "Sub-task"
  },

  "labels": {
    "always": ["my-team"],
    "area": ["api", "worker", "infra"],
    "kind": ["reliability", "spike", "security", "tech-debt"],
    "enabler": true
  },

  "tests": {
    "framework": "nunit",
    "naming": "Given{given}_Should{then}",
    "skipMarker": "[Ignore(\"{ticket} — implementation lands in PR 2\")]",
    "glob": "**/*Test*.cs",
    "projectPattern": "src/Tests/{layer}/{project}.Test",
    "conventionsDoc": ".claude/docs/test-conventions.md"
  },

  "jira": {
    "cloudId": "your-site.atlassian.net",
    "projectKey": "ABC",
    "acceptanceCriteriaField": null,
    "parentField": "parent",
    "serviceDeskProjects": ["SUPPORT"]
  },

  "github": {
    "repo": "owner/name",
    "typeLabels": { "story": "enhancement", "bug": "bug", "enabler": "enabler" },
    "useMilestoneAsEpic": true
  }
}
```

## Keys

| Key | Meaning | Fallback |
|---|---|---|
| `tracker` | which adapter to call: `jira`, `github`, or `none` for draft-only | ask |
| `draftPath` | where reviewable ticket drafts are written | `docs/tickets` |
| `overlay` | a skill or doc carrying site-only rules the generic layer must not hardcode — load it after `conventions.md` | none |
| `hierarchy.*` | what each SAFe level is called in your tracker. Omit a level your tracker doesn't model | see `conventions.md` § 2 |
| `labels.always` | applied to every ticket, e.g. a team label | none |
| `labels.area` / `labels.kind` | the permitted taxonomy — the skill won't invent labels outside it | none, and it will say so |
| `labels.enabler` | emit `enabler` + `enabler-<type>` labels | `true` |
| `tests.framework` | `nunit`, `xunit`, `mstest`, `jest`, `vitest`, `pytest`, `go` | inferred from the repo |
| `tests.naming` | scenario→test-name pattern. `{given}` and `{then}` are PascalCased scenario fragments; `{scenario}` is the whole thing | `<subject>_<condition>_<outcome>` |
| `tests.skipMarker` | the attribute/call that makes a test skip. `{ticket}` is substituted | inferred from framework |
| `tests.glob` | which paths count as test files for the PR 2 check | inferred from framework |
| `tests.projectPattern` | how to find the test project for a given production project | discovered by search |
| `tests.conventionsDoc` | site test-style doc — base class, builders, gotchas | none |
| `jira.acceptanceCriteriaField` | custom field id, or `null` when the project has none and Gherkin goes in the description | `null` |
| `jira.serviceDeskProjects` | projects where a public comment reaches a customer. **Never write to these.** | none |
| `github.useMilestoneAsEpic` | model the Feature level as a milestone rather than a tracking issue | `false` |

## Overlay vs floor — what to duplicate on purpose

An `overlay` keeps site detail in one place, which is the right default: one source of truth, no drift. But it is a *plugin*, and a plugin can fail to load — stale cache, not installed yet, a marketplace that hasn't refreshed. If the overlay carries everything, a cache miss leaves the skill with no idea where the tracker is or which projects are dangerous.

So duplicate a **floor** into this file deliberately:

- **Anything safety-critical.** `jira.serviceDeskProjects` above all — a rule that stops you writing to a customer-facing project must not depend on a cache refresh. Fail safe, not fail open.
- **Anything without which the skill simply cannot function.** Tracker coordinates, project key, the hierarchy names.

Leave in the overlay:

- The full label taxonomy and the reasoning behind it.
- Fixture-harness detail, test-stack gotchas, house naming conventions.
- Anything that is guidance rather than a guard.

The overlay still wins wherever the two overlap. The floor exists so that a degraded run is *narrower*, not *unsafe* — you lose the label taxonomy and the fixture conventions, you do not lose the prohibition on writing to a support project.

## Why the tracker is a seam

The ticket rules — enabler classification, Gherkin grammar, the coverage checks, Definition of Ready — are the same whether the ticket ends up in Jira, GitHub Issues, Linear or a markdown file. Only three things change: where the acceptance criteria go, what the hierarchy is called, and how a parent is set.

So `ticket` never calls a tracker API. It produces a draft, and hands off to whichever adapter `tracker` names. That means:

- The same conventions apply across repos that use different trackers.
- A new tracker is a new adapter skill, not a fork of the rules.
- `tracker: "none"` is a legitimate setting. The draft file is useful on its own.
