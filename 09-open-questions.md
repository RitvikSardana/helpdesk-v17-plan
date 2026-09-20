# Open questions

[Index](README.md) · previous: [08-future-scope.md](08-future-scope.md)

The live list. Every question raised in the other sections lands here. When one is answered it moves to [01-decisions.md](01-decisions.md) and the section waiting on it gets written.

Critical means a section cannot be finished until it is answered. The rest can be decided while the work is in flight.

## Critical

### Contacts on Customer: child table or Dynamic Link

`HD Customer` holds its members in a `contacts` child table. ERPNext links a `Contact` to a `Customer` through `Dynamic Link`. One of the two survives the merge.

The child table is what Helpdesk's permission query reads today. The Dynamic Link is what every other Frappe app reads, including the automatic role assignment below.

Blocks: the field mapping in [02-shared-masters/customer.md](02-shared-masters/customer.md), the migration patch, and the migrator's contact step.

**This is the one on the deadline.** It decides a field on `Customer`, which decides what goes into ERPNext v17, which ships about a month out. Every other question in this file can be answered while work is in flight. This one gates the start of it. See [07-order-of-work.md](07-order-of-work.md).

### Helpdesk contacts silently gaining the ERPNext Customer role

`HD Customer` and `Customer` are two different roles today. Helpdesk owns the first, with `desk_access = 0` and `home_page = /helpdesk`. ERPNext owns the second, and it carries nine portal routes from `erpnext/hooks.py`.

[One portal role or two](#one-portal-role-or-two) below asks whether to merge them on purpose. **This question is about them merging by accident.**

`set_default_role` in `erpnext/portal/utils.py` walks a new user's `Contact` links and grants the role from the link target:

```python
if link.link_doctype == "Customer" and "Customer" not in roles:
    doc.add_roles("Customer")
```

It runs on user insert, with no admin action and nothing to opt out of.

So if the contacts question above is answered "Dynamic Link", every helpdesk portal contact acquires the ERPNext `Customer` role the moment their user record is created. Nobody chose it, and it contradicts the portal rule in [01-decisions.md](01-decisions.md). It is the merge the question below is carefully weighing, arriving through the back door.

Three ways out: Helpdesk suppresses the assignment, the portal rule is softened, or the contacts question is answered the other way. Which means this is not really a separate decision. It is the contacts question seen from the permission side, and answering one answers both.

Blocks: [02-shared-masters/permissions.md](02-shared-masters/permissions.md) and the contacts question.

## Open

### One portal role or two

Today Helpdesk owns `HD Customer` and `HD Customer Manager`, both `desk_access = 0`. ERPNext owns `Customer`. The question is whether Helpdesk's portal user should just hold `Customer`.

This is the deliberate version of the merge. The accidental version is [Helpdesk contacts silently gaining the ERPNext Customer role](#helpdesk-contacts-silently-gaining-the-erpnext-customer-role) above, and it can happen whatever is decided here.

Sharing it buys one thing: an ERP customer contact gets the helpdesk portal without a second invite.

Sharing it costs four:

- The role carries nine portal pages from `erpnext/hooks.py`: `/quotations`, `/orders`, `/invoices`, `/shipments`, `/project`, `/timesheets`, `/material-requests`, `/addresses`, `/issues`. Each is scoped by a permission query, so it is not a cross-customer leak, but it is an ERPNext sales portal appearing for someone who wanted to raise a ticket. `/issues` would be a dead page after the Support cleanup.
- The automatic assignment above becomes an enrolment path into Helpdesk that Helpdesk does not control.
- If `Portal Settings.default_role` also became `Customer`, `create_customer_or_supplier` would insert a `Customer` record and a `Contact` for every website user on session creation. One sales master row per self-serve ticket raiser.
- `setup_customer_role` sets `home_page` and `desk_access` on the roles it owns. Applied to `Customer`, every ERP portal customer lands on `/helpdesk`, and uninstalling Helpdesk cannot put that back.

It is also only half a merge. There is no ERPNext counterpart to `HD Customer Manager`.

Sharing becomes safe only if ERPNext splits `Customer` into identity and portal access, which are one role today. That is an ERPNext change, not one Helpdesk can make from its side.

Current lean: keep them separate, and let the invite flow grant both when an admin wants someone on both portals.

Related: the ERPNext website portal is not deprecated. Its pages were last touched in March 2026 and carry no deprecation markers. The e-commerce half left for the `webshop` app in v14; the party pages stayed.

### The Issue email flow

Support routes inbound mail to `Issue` through an Email Account. After the deprecation that creates rows nobody reads.

Leaning: repoint it in a patch so mail creates an `HD Ticket`. Open because the repoint rewrites an existing site's mail configuration.

Blocks nothing. Belongs in [03-support-module-cleanup.md](03-support-module-cleanup.md).

### Which side wins a field conflict

Step 2 of the migrator copies `HD Customer` values onto the `Customer` a ticket now points at. Where both hold a value and they differ, somebody picks.

There is no obvious default. Helpdesk is the fresher source for support data, `domain` and `email_id` and `mobile_no`, because agents correct it while working a ticket. ERPNext is the fresher source for anything billing touched. A per field default is possible, a global one is not.

Open: a global default with a per field override, a per field default shipped with the tool, or no default and every conflict is shown.

Blocks nothing now. Has to be answered before [04-migrator.md](04-migrator.md) ships.

## Deferred, with a trigger

These are decided for v17. They reopen only if the trigger fires.

### Company on a ticket

`HD Ticket` gets no `company` field in v17.

Reopens when a multi-company site needs tickets filtered or reported by company.

### Item and serial number on a ticket

`HD Ticket` gets no `item_code` or `serial_no`. `Issue` never had them either, so nothing is lost in the deprecation. `Warranty Claim` keeps them and is not being deprecated.

Reopens when the customer view's warranty tab is not enough and agents need to raise a ticket against a serial directly.

## Build time

Not blocking. Decided when the code is written.

- The SLA status category contract: whether it standardises on one field name and value set across document types, or carries a per consumer map. From [02-shared-masters/sla.md](02-shared-masters/sla.md).
- Where the per document type SLA registry lives. From the same section.
- How the merge log outlives an uninstall of the migrator app: the app is never really uninstallable, or the log is exported on uninstall. Ownership is settled in [01-decisions.md](01-decisions.md); this is the mechanism. From [04-migrator.md](04-migrator.md).
- How to warn someone editing a `Holiday List` that it now reaches HRMS payroll. Shared records mean shared consequences and the person editing an SLA calendar has no reason to expect it. From [06-app-integration.md](06-app-integration.md).

---

[← Future scope](08-future-scope.md)
