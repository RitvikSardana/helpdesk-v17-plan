# Holiday list

[Index](../README.md) · [Shared masters](README.md)

## What changes

`HD Service Holiday List` folds into ERPNext's `Holiday List`. `HD Holiday` folds into `Holiday`.

Both Helpdesk doctypes are deprecated: read only, no create button, badge on the form, rows stay readable.

11 references move.

## Why

HRMS, payroll and the SLA all need to know which days do not count. A site should maintain one calendar of public holidays, not two.

## What Helpdesk brings

| Field | Type | Note |
|---|---|---|
| `description` | Data | not on ERPNext's list, can go in the core doctype |
| `recurring_holidays` | JSON | not on ERPNext's list, can go in the core doctype |

Both are additive and useful to every consumer, so ERPNext should take them as standard fields rather than Helpdesk adding them as custom fields.

`HD Holiday` brings nothing. `Holiday` already has everything it has, plus `is_half_day`.

## What Helpdesk gains

ERPNext's `Holiday List` is otherwise a superset.

| Field | What it does |
|---|---|
| `country` | pull that country's public holidays |
| `subdivision` | narrow it to a state or region |
| `get_local_holidays` | button that fills the table from the above |
| `is_half_day` | flag a date as a half day |

So Helpdesk users get automatic public holidays for free, which they do not have today.

## Not company scoped

`Holiday List` has no `company` field. It is global, and any number of companies can point at the same one.

The link runs the other way: `Company.default_holiday_list` nominates a default.

It is already shared inside ERPNext by CRM, Manufacturing, Projects, Setup and Support, which is the argument for Helpdesk joining rather than keeping its own.

## How it relates to working hours

They are two halves of one calculation and they live in different places.

- **Working hours** come from the agreement's `support_and_resolution` table. One row per weekday, each with a start and end time. That says which hours of a working day the clock runs.
- **Holiday list** says which whole days do not count at all, whatever the weekday row says.

`Holiday List` holds no times. `weekly_off` is not a rule, it is a button that generates a holiday row for every occurrence of that weekday in the range.

Known gap: `Holiday.is_half_day` has no hours attached, so there is no defined half. The SLA cannot do anything sensible with it and ignores it.

## Existing data

The default is to prefer what ERPNext already has, rather than pushing Helpdesk's lists into it.

1. If the site already has `Holiday List` records, use those. The agreement repoints at an ERPNext list and no new list is created.
2. If it has none, create a `Holiday List` for every `HD Service Holiday List`, copying holidays across.
3. Repoint `Service Level Agreement.holiday_list`. See [sla.md](sla.md).
4. Leave the Helpdesk rows in place, deprecated.

The reasoning: a site running ERPNext already maintains its real holiday calendar there, and it is the one HR and payroll use. Helpdesk's list is usually the copy.

No matching happens during the upgrade. If the automatic choice is wrong, an admin changes it in the migrator, which can also merge duplicate lists. See [../04-migrator.md](../04-migrator.md).

Open at build time: which ERPNext list to pick when the site has several. `Company.default_holiday_list` is the answer.

## Risks

Name collisions. A site running both apps may already have a `Holiday List` named the same as an `HD Service Holiday List`. The patch has to handle that without failing, which in practice means suffixing the new one and letting the migrator sort it out.

## Open questions

None.

---

[← Customer](customer.md) · [Agent →](agent.md)
