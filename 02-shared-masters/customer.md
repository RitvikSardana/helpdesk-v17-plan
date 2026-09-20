# Customer

[Index](../README.md) · [Shared masters](README.md)

## What changes

Tickets link to ERPNext `Customer`. `HD Customer` is deprecated: read only, no create button, badge on the form, rows stay readable, nothing links to them.

61 references move. This is the largest single change in v17 and everything in [../07-order-of-work.md](../07-order-of-work.md) track C waits on it.

## Why

A site running both apps keeps two customer lists and syncs them by hand. The same company sits in both under different names. An agent cannot see orders, a salesperson cannot see tickets.

One record removes the sync. ERPNext owns it because every other app can link to it.

## Field mapping

| `HD Customer` | `Customer` | Note |
|---|---|---|
| `customer_name` | `customer_name` | same |
| `customer_type` | `customer_type` | option lists match exactly |
| `image` | `image` | currently `hidden: 1` on Customer, needs unhiding |
| `mobile_no` | `mobile_no` | read only on both |
| `email_id` | `email_id` | read only on both |
| `primary_contact` | `customer_primary_contact` | renamed field, same meaning |
| `domain` | `domain` | new custom field (might go to ERPNext as a core field) |
| `country` | `country` | new custom field (might go to ERPNext as a core field) |
| `contacts` | undecided | child table, or `Contact` Dynamic Link. See [Contacts](#contacts) |
| `erpnext_customer` | none | dies, used as the migration key first |

Full list in [../appendix/field-mapping.md](../appendix/field-mapping.md).

## Custom fields

`Customer` gets two custom fields, added by a Helpdesk patch. Either may become a core ERPNext field instead, in which case the patch asserts it exists rather than creating it.

| Field | Type | Why |
|---|---|---|
| `domain` | Data | match an incoming email to a customer |
| `country` | Data | shown in the Helpdesk customer dialog |

ERPNext keeps country on `Address`, not on `Customer`. A customer with no address has no country. Putting it on `Customer` duplicates that. Deliberate trade for a simpler dialog.

CRM wants `domain` too, since `CRM Organization` has it. Whichever app's patch runs first creates the field and the second finds it already there. Helpdesk owns the definition because it migrates first.

## Contacts

`HD Customer` has a `contacts` child table, `HD Customer Member`. ERPNext does not work that way: a `Contact` links to a `Customer` through the standard Dynamic Link child table on `Contact` itself.

Not settled. Tracked as critical in [../09-open-questions.md](../09-open-questions.md).

Dropping the child table means one contact can belong to several customers, which Helpdesk cannot express today. Keeping it means two ways of saying the same thing.

## The customer dialogs

Helpdesk creates and edits customers through two dialogs. They keep their current fields by default: logo, name, customer type, country, domain, and a primary contact block with first name, last name, email and mobile.

The agent never sees ERPNext's customer form. `Customer` is a much heavier record, with a naming series, customer groups, territories, credit limits and accounts. None of that appears.

The layout is driven by `HD Field Layout`, so an admin can add ERPNext fields to either dialog if they want them.

Scope note: `HD Field Layout` currently lays out Helpdesk doctypes. It has to stop assuming the doctype belongs to Helpdesk.

## ERP tabs on the customer view

The Helpdesk customer view gains three read only tabs: `Warranty Claim`, `Maintenance Visit`, `Installation Note`. Each row deep links to the desk document.

A tab appears only if the customer has at least one record of that type. No configuration. A site with no ERP activity sees no extra tabs. Three count queries when the customer loads.

Build it as one configurable linked-documents tab, not three components. Config is doctype, filter field, columns, link target. A fourth tab is then a config entry.

Helpdesk grants no permission on those three doctypes. The tabs fetch through `get_list`, so a tab fills only for a user who already holds `Maintenance User` or `Sales User`. An agent without those roles never sees the tab at all.

Portal users never see them. See [permissions.md](permissions.md).

Sales and finance documents on this view are future scope.

## Existing data

The upgrade patch runs automatically and always succeeds. It does no matching.

1. For every `HD Customer` with `erpnext_customer` filled, link to that `Customer`. The old sync already verified it.
2. For every other `HD Customer`, create a `Customer` and copy the fields above.
3. Repoint `HD Ticket.customer`.
4. Move contacts across.
5. Leave `HD Customer` rows in place, deprecated.

On a site that ran both apps, the `Customer` table now holds duplicates: the real Netflix Inc that was always there, and a new Netflix that came from Helpdesk. Cleaning that up is optional and comes later, in [../04-migrator.md](../04-migrator.md).

## Risks

`Customer` is wide. An agent with read permission gets `credit_limits`, `accounts`, `tax_id`, `default_price_list` and `sales_team` along with the name. Accepted for v17 and handled as display rather than permission: Helpdesk renders a curated set of fields and a site adds more through the layout editor. See [../01-decisions.md](../01-decisions.md).

The contacts decision is unresolved and blocks the migration patch. It has to be answered before step 4 above can be written.

`Customer` has a controller that does real work on insert. Creating one per `HD Customer` on a large site is slower than creating an `HD Customer`. Worth measuring before the patch ships.

## Open questions

Tracked in [09-open-questions.md](../09-open-questions.md).

- Contacts: child table or standard Dynamic Link. Critical.
- None on the financial fields. Settled in [../01-decisions.md](../01-decisions.md).
- Company scoping on tickets, deferred.

---

[← Shared masters](README.md) · [Holiday list →](holiday-list.md)
