# Workspace

My generic VS Code Workspace setup with AI, Git hooks and VS Code settings.

## Claude Code plugins

This repo doubles as a Claude Code plugin marketplace (`chrison`).

```text
/plugin marketplace add ChrisonSimtian/Workspace
/plugin install bdd-delivery@chrison
```

### `bdd-delivery`

A BDD/SAFe delivery workflow: write tickets whose acceptance criteria are Gherkin scenarios, then turn those scenarios into tests as the first half of a stacked PR pair.

| Skill | Does |
|---|---|
| `ticket` | Shapes work into a Story / Enabler / Bug, or an Epic + Story breakdown. Writes a reviewable markdown draft; pushes only on approval. |
| `tests-from-ticket` | Reads a ticket's Gherkin, classifies each scenario by what can actually reach it, scaffolds the tests it can honestly write. |
| `tracker-jira` | Jira mechanics — dedup search, hierarchy mapping, parent wiring, key stamping. |
| `tracker-github` | GitHub Issues mechanics — milestones, sub-issues, labels-as-types. |

**The rules know nothing about your tracker.** `ticket` produces a draft and delegates to an adapter, so the same conventions apply whether the work lands in Jira, GitHub Issues, or nowhere at all. Site specifics — tracker coordinates, label taxonomy, test framework and naming — come from [`.bdd-delivery.json`](plugins/bdd-delivery/config.md) in the consuming repo.

Three ideas in here earn their keep:

- **Enablers are team-owned.** If you can't honestly write `so that <business value>`, it isn't a Business Story a BA owes you — it's an Enabler Story. Classify it and write it yourself. That one rule unblocks most "nobody will write my technical ticket" complaints.
- **Four coverage checks on every ticket** — failure path, replay/idempotency, silence-on-success, deploy order. The recurring failure modes of message-driven systems, as a checklist rather than a habit.
- **`tests-from-ticket` refuses to fake tests it can't write.** Every scenario is classified unit / integration / infra / not-mechanisable, and the headline output is the list of scenarios with *no* automated coverage. A test that goes green against a fake while the real behaviour is untested is worse than no test.

Full conventions: [`plugins/bdd-delivery/skills/ticket/conventions.md`](plugins/bdd-delivery/skills/ticket/conventions.md).

#### Versioning

**Bump `version` in both `.claude-plugin/marketplace.json` and `plugins/bdd-delivery/.claude-plugin/plugin.json` on every content change.** There is no CI stamping it. Claude Code's `autoUpdate` compares versions, not content — so editing a skill without bumping leaves every installed copy silently stale, and `/plugin` cheerfully reports "already at the latest version" because by its own measure it is.

#### Status

Early. Exercised against one Jira site and a .NET/NUnit codebase; the GitHub adapter and the non-.NET test frameworks are written but lightly used. Expect the first run in a new environment to surface assumptions.

## Licence

MIT.
