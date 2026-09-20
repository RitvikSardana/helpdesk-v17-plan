# Helpdesk v17

Helpdesk's part of Frappe One. Internal plan, private repo.

Start with [00-overview.md](00-overview.md), then [01-decisions.md](01-decisions.md). Everything after those is written on top of them.

## Sections

| # | File | Covers | Status |
|---|---|---|---|
| 00 | [Overview](00-overview.md) | what v17 is, scope, out of scope | draft |
| 01 | [Decisions](01-decisions.md) | the settled calls, pinned | draft |
| 02 | [Shared masters](02-shared-masters/README.md) | customer, holiday list, agent, team, SLA, permissions | in progress |
| 03 | [Support module cleanup](03-support-module-cleanup.md) | issue trio deprecated, warranty and maintenance stay | draft |
| 04 | [Migrator](04-migrator.md) | the separate app, six step UI, matching, merge log | draft |
| 05 | [Upgrade](05-upgrade.md) | version lock, patch order, breaking changes | draft |
| 06 | [App integration](06-app-integration.md) | HRMS, CRM, moving between apps | draft |
| 07 | [Order of work](07-order-of-work.md) | tracks, critical path, what slips | draft |
| 08 | [Future scope](08-future-scope.md) | deferred on purpose | draft |
| 09 | [Open questions](09-open-questions.md) | undecided, with what each one blocks | draft |

## Appendix

- [Field mapping](appendix/field-mapping.md)
- [Reference counts](appendix/reference-counts.md)
- [Doctype inventory](appendix/doctype-inventory.md)

## How this is kept

[09-open-questions.md](09-open-questions.md) is the live list. When a question there is answered it moves to [01-decisions.md](01-decisions.md) and the section waiting on it gets written.
