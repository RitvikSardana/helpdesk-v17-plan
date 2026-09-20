# Team

[Index](../README.md) · [Shared masters](README.md)

## What changes

Nothing. `HD Team` stays exactly as it is, with no connection to ERPNext or HRMS.

## Why

There is no ERPNext equivalent worth merging into.

`Sales Team` is a child table about commission splits, not a work queue. `Employee Group` and `Department` are HR groupings tied to org structure, and a support team is not the same thing. A ticket team is a routing target with an assignment rule attached.

`User Group` was considered and rejected. It has one field, `user_group_members`, and nothing in Frappe, ERPNext, CRM or Helpdesk links to it. Moving would mean pushing `assignment_rule`, `ignore_restrictions` and `disabled` into a Frappe core doctype, migrating 31 files and every ticket's `agent_group`, to share a grouping concept that has no second consumer.

## Detail

`HD Team` keeps `team_name`, `assignment_rule`, `users`, `ignore_restrictions` and `disabled`.

`HD Team Member` stays as its child table.

## Existing data

None.

## Risks

None.

## Open questions

None.

---

[← Agent](agent.md) · [SLA →](sla.md)
