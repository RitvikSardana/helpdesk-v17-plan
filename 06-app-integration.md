# App integration

[Index](README.md) · previous: [05-upgrade.md](05-upgrade.md) · next: [07-order-of-work.md](07-order-of-work.md)

## What changes

Helpdesk stops bridging to other apps and starts sharing their records.

Every connection Helpdesk has to another app today is a sync: two records kept in step by hooks, with a settings toggle and a button to repair the drift. v17 deletes all of that and replaces it with one record, read directly.

## The three relationships are not the same shape

Worth separating before the detail, because they get discussed as one thing and they are not.

| App | What v17 does | Size |
|---|---|---|
| ERPNext | Helpdesk gives up `Customer`, `Holiday List` and the SLA engine | the whole plan |
| HRMS | Helpdesk reads one thing, whether an agent is on leave | one behaviour |
| CRM | CRM folds its own duplicates into the same ERPNext records | a peer, not a dependency |

ERPNext is where the work is. HRMS is a feature. CRM is another app doing the same migration on its own schedule, and Helpdesk's only interest in it is that they land on the same records.

## ERPNext

### The bridge is deleted

`ERPNext HD Settings`, the `Customer` doc_events, the User Permission and DocShare mirrors, and the Sync Customers button all go. They exist to keep two customer masters in step, and v17 has one.

The file list and the two things that outlive the sync are in [04-migrator.md](04-migrator.md), because the migrator is the last thing that needs them.

### What replaces it

Nothing. That is the point.

There is no v17 integration layer between Helpdesk and ERPNext. `HD Ticket.customer` is a link to `Customer` and Helpdesk reads it like any other link field. An integration is what you build when two apps own the same concept. One owner needs no integration.

### What Helpdesk reads from ERPNext

```
Customer                   the ticket's customer
Contact                    who raised it
Holiday List               the SLA calendar
Service Level Agreement    the SLA itself
Warranty Claim             read only tab on the customer view
Maintenance Visit          read only tab
Maintenance Schedule       read only tab
Installation Note          read only tab
```

The first four are load bearing. The last four are display, and they deep link into the desk document rather than rendering it. See [02-shared-masters/customer.md](02-shared-masters/customer.md) and [03-support-module-cleanup.md](03-support-module-cleanup.md).

## HRMS

### One read, no shared record

Helpdesk does not give HRMS anything and does not take a record from it. It asks one question: is this agent on leave.

`HD Agent` stays in Helpdesk with no new fields. There is no link field to `Employee`. `HD Agent.user` and `Employee.user_id` both point at `User`, so the employee is found through the user, filtered to active employees. An agent with no employee record is simply unaffected, which keeps contractors working. See [02-shared-masters/agent.md](02-shared-masters/agent.md).

### Off by default

Three settings in `HD Settings`, and the first one gates the other two:

| Setting | Default |
|---|---|
| Leave integration enabled | off |
| Status to use on leave | must have category Unavailable |
| Reset when leave ends | on |

Off is the right default because availability is something teams manage by hand today, and turning it on without being asked would start overwriting their choices.

### The one place HRMS and Helpdesk share a record

`Holiday List`. Helpdesk's SLA calendar and HRMS payroll read the same rows, which is the argument for moving it to ERPNext rather than keeping `HD Service Holiday List`. See [02-shared-masters/holiday-list.md](02-shared-masters/holiday-list.md).

That also means a holiday list edit now has consequences in two apps. Nobody owns that warning yet.

### Shifts stay out

`Shift Type` has one start and end time for all days plus attendance machinery. The SLA clock needs one row per weekday with its own hours. They do not line up, and shift is built for attendance rather than deadlines. The SLA clock stays a property of the agreement, never of the agent.

## CRM

### The same duplication, the same fix

CRM carries its own copies of what ERPNext already owns, exactly as Helpdesk does:

```
CRM Organization               Customer
CRM Holiday List               Holiday List
CRM Holiday                    Holiday
CRM Service Level Agreement    Service Level Agreement
CRM Service Day                Service Day
CRM Service Level Priority     Service Level Priority
```

And its own bridge to keep them in step: `ERPNext CRM Settings` with an `enabled` toggle, `crm/integrations/erpnext/user_permission.py` mirroring permissions, `create_customer_in_erpnext` on a deal status change.

Two apps built the same workaround for the same missing thing. v17 removes the reason both of them existed.

### CRM is a peer, not a dependency

Helpdesk does not need CRM installed and CRM does not need Helpdesk. Today they do not reference each other at all: no doctype link, no import, no hook. The only mentions of one in the other are translation strings and built assets.

So nothing in this plan waits on CRM. The migrator ships the two CRM pairs as defaults because the tool is shared, not because Helpdesk is blocked. See [04-migrator.md](04-migrator.md).

### What they do share, once both have moved

Two records, and they are the whole of the v17 integration:

**`Customer`.** An agent opening a ticket and a salesperson opening a deal are looking at the same company record. That is the shared customer key from [00-overview.md](00-overview.md), and it is what makes "an agent can see the customer's orders" true.

**`Service Level Agreement`.** One engine, two apps. Which means the status contract has to work for a ticket and a deal without either app hardcoding the other's statuses. That is already a build time question. See [09-open-questions.md](09-open-questions.md).

### CRM's remote mode goes away

`ERPNext CRM Settings` can point at an ERPNext on a different site and talk to it over the API: `is_erpnext_in_different_site`, `erpnext_site_url`, `api_key`, `api_secret`. CRM is removing it.

That settles something the rest of this plan rests on rather than states. **Frappe One is one site with four apps.** A remote ERPNext cannot share a `Customer` row, so sharing records and talking to another site over HTTP were never going to coexist, and the remote configuration had no path to v17 in any case.

Nothing changes for Helpdesk, which never had a remote mode. It is written down because it is a real breaking change for whoever is running that setup, and because the one site assumption is now a decision instead of an assumption.

## Moving between apps

In scope for v17: **deep links**. A customer view in Helpdesk lists warranty claims and maintenance visits and links out to the desk form. A ticket links to its customer.

Out of scope, and in [08-future-scope.md](08-future-scope.md):

```
a shared shell and app switcher
shared form builder and view settings across apps
Helpdesk and CRM flows beyond the shared customer key
```

The line is: v17 makes the records shared, not the interface. Linking out to another app's form is enough to prove the data is shared, and it is the version that ships in four months rather than twelve.

## Existing data

The HRMS integration has none. No new fields, no migration, and it is off until someone turns it on.

The ERPNext bridge's data is the two link fields, `HD Customer.erpnext_customer` and `Customer.hd_customer`. They survive the upgrade because tier 1 and the migrator read them. See [05-upgrade.md](05-upgrade.md).

The User Permission and DocShare rows the mirrors created are the part that needs care. Deleting the mirror is easy. Making sure those rows end up pointing at `Customer` rather than at a deprecated `HD Customer` is a migration, and it belongs to [02-shared-masters/permissions.md](02-shared-masters/permissions.md).

## Risks

**Deleting the mirrors before the permission rows move.** The mirror is what currently keeps a customer's portal access working on both masters. Remove it first and access depends on whichever master the rows happen to sit on. Order matters, and it is an order across two sections.

**A holiday list edit now reaches payroll.** Shared records mean shared consequences, and the person editing an SLA calendar has no reason to expect that. This needs saying in the UI, not just in release notes.

**Assuming CRM has moved.** Helpdesk's plan is written as though `Customer` is the one customer master. Until CRM ships its own migration, a site with both still has `CRM Organization` sitting alongside. Nothing breaks, but "one customer list" is not true yet on that site.

**The leave integration overwriting a manual status.** An agent who set themselves unavailable before going on leave is reset to their default when it ends. Minor, and the reset setting turns it off.

## Open questions

Tracked in [09-open-questions.md](09-open-questions.md).

- The SLA status category contract, which has to serve a ticket and a deal.
- Where the SLA registry lives.

---

[← Upgrade](05-upgrade.md) · [Order of work →](07-order-of-work.md)
