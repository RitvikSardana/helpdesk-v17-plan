# Agent

[Index](../README.md) · [Shared masters](README.md)

## What changes

Nothing structural. `HD Agent` stays in Helpdesk with no new fields.

What is new is behaviour: an agent who goes on leave in HRMS becomes unavailable in Helpdesk, and becomes available again when the leave ends.

## Why

Helpdesk needs its own agent record. It holds availability, the user image and the active flag, none of which are HR concepts.

But leave lives on `Employee`. Without a connection there is no way to know an agent is away, and tickets keep getting assigned to someone who is not there.

## Finding the employee

No link field. `HD Agent.user` and `Employee.user_id` both point at `User`, so the employee is found from the user.

`Employee.user_id` is not unique. One user can have several employee records, for example a rehire or one per company. The lookup filters to active employees, which is what ERPNext's own code does.

An agent with no employee record is unaffected. Everything below simply does not fire. That keeps external contractors working without an `Employee`.

## Leave and availability

`HD Agent Status` already has a `category` field with values Active, Away and Unavailable. `HD Agent.availability` links to it. No schema change needed anywhere.

- Employee goes on leave: the linked agent's availability is set to a status whose category is Unavailable.
- Leave ends: the agent goes back to their default status.

Assignment already skips unavailable agents. That behaviour does not change.

An agent going on leave never changes a ticket's due time. It only changes who gets assigned. See [sla.md](sla.md).

## Settings

In `HD Settings`, under the HRMS integration:

| Setting | Effect |
|---|---|
| Integration enabled | off, nothing below runs |
| Status to use on leave | which Unavailable status gets set |
| Reset when leave ends | on, the agent returns to their default status |

Off by default is the safe choice for existing sites, since availability is something teams manage by hand today.

## Shifts

Out of scope for the SLA clock.

HRMS `Shift Type` has one `start_time` and one `end_time` for all days, plus attendance machinery: check in windows, grace periods, thresholds, overtime. SLA needs one row per weekday with its own hours. The two do not line up, and shift is built for attendance rather than deadlines.

The SLA clock is always a property of the agreement, never of the individual agent.

## Existing data

None. No new fields, no migration.

## Risks

An agent who manually set themselves unavailable before going on leave gets reset to their default status when the leave ends, losing the manual state. Minor, and the reset setting can be turned off.

Leave that is cancelled or shortened after it starts has to move the agent back. Whatever fires on leave start has to fire on leave change too.

## Open questions

None.

---

[← Holiday list](holiday-list.md) · [Team →](team.md)
