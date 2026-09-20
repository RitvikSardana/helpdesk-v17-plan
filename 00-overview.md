# What v17 is

[Index](README.md) · next: [01-decisions.md](01-decisions.md)

Helpdesk v17 is Helpdesk's part of Frappe One. Four apps, one system: ERPNext, Helpdesk, CRM, HRMS, installed together and sharing their data.

Today Helpdesk is an island. It keeps its own customers, its own holiday lists, its own SLA engine, and syncs some of it to ERPNext by hand. A site running both apps maintains the same company twice. An agent answering a ticket cannot see the customer's orders. A salesperson cannot see their tickets.

v17 removes the duplication. Helpdesk stops owning records that ERPNext already owns, and starts reading ERPNext data directly.

## What changes for a Helpdesk user

- Customers are ERPNext customers. One list, one record, one place to edit.
- Holiday lists are ERPNext holiday lists, shared with HRMS and payroll.
- SLA runs on one engine that ERPNext owns, used by Helpdesk and CRM.
- The customer view shows warranty claims, maintenance visits and installation notes when they exist.
- Agents who are on leave in HRMS stop receiving assignments.

## Version and timing

Helpdesk renumbers to 17 to match ERPNext and the Framework. Its own version line ends.

ERPNext v17 ships about a month from now. The Helpdesk suite work lands later, roughly four months out. That number is a ballpark, not a commitment.

All of it lands on a long lived `v17` branch with all four apps installed.

## Scope

This plan covers Helpdesk only. CRM has its own owner and its own plan. Where Helpdesk needs something from CRM it is written here as a dependency, not as work.

Out of scope for v17:

| Not doing | Where it goes |
|---|---|
| Doctype renames, dropping the `HD` prefix | `08-future-scope.md` |
| Shared form builder and view settings across apps | `08-future-scope.md` |
| Warranty Claim moving into Helpdesk | `08-future-scope.md` |
| Helpdesk and CRM flows beyond the shared customer key | `08-future-scope.md` |
| A shared shell and app switcher | `08-future-scope.md` |
| Quality module consolidation | rejected, see `03-support-module-cleanup.md` |
| CRM internals | another owner |

## How to read this

`01-decisions.md` holds the settled calls. Everything after it is written on top of those and does not re-argue them.

Each section states what changes, why, the detail, what happens to existing data, what could go wrong, and what is still open. A section with no data impact says so rather than dropping the heading.

`09-open-questions.md` is the live list. When a question there is answered it moves to `01-decisions.md` and the section waiting on it gets written.

Audience: internal. Private repo.

---

[Decisions →](01-decisions.md)
