# Shared masters

Helpdesk stops owning records that ERPNext already owns. This is the core of v17 and the critical path for everything else.

Back to [the index](../README.md).

| File | Covers |
|---|---|
| [customer.md](customer.md) | `HD Customer` folds into `Customer`. The big one, 61 references. |
| [holiday-list.md](holiday-list.md) | `HD Service Holiday List` folds into `Holiday List`. |
| [agent.md](agent.md) | `HD Agent` stays unchanged. Leave drives availability, employee found via `User`. |
| [team.md](team.md) | `HD Team` stays as it is. |
| [sla.md](sla.md) | One SLA engine in ERPNext, used by Helpdesk and CRM. |
| [permissions.md](permissions.md) | Who can read what once the masters are shared. |
| [schema-changes.md](schema-changes.md) | Every schema change in one list, including the `app` field rule. |

Two masters that were in the original commonification plan are not here. `HD Ticket Priority` and `HD Ticket Type` stay in Helpdesk. See [sla.md](sla.md) for why priority no longer has to move, and [../03-support-module-cleanup.md](../03-support-module-cleanup.md) for why type never did.

---

[← Decisions](../01-decisions.md) · [Customer →](customer.md)
