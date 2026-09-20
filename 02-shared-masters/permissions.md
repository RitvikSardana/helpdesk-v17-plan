# Permissions

[Index](../README.md) · [Shared masters](README.md)

## What changes

Once tickets link to `Customer` and the customer view links to ERP documents, Helpdesk users can reach ERPNext data for the first time.

Two rules, and they are not the same rule.

## A portal user reads nothing in ERPNext

Hard rule. No role permission on any ERPNext doctype, ever.

Anything a customer sees that originates in ERPNext goes through a Helpdesk whitelisted method that fetches it with elevated permission and returns only chosen fields. The customer name on their ticket comes back as a string, not as a readable `Customer` record.

The ERP tabs on the customer view are agent only. A portal user never sees them.

### Helpdesk and ERPNext keep separate portal roles

```
HD Customer          Helpdesk portal role, desk_access = 0
HD Customer Manager  Helpdesk portal role, desk_access = 0
Customer             ERPNext portal role
```

`HD Customer` gates the Helpdesk portal and does not change.

Merging onto ERPNext's `Customer` role is not an option. That role carries the ERPNext website portal, which is exactly what the hard rule forbids.

Naming oddity: the role `HD Customer` and the doctype `HD Customer` share a name, the doctype is being deprecated, and the role is unrelated to it and stays. No renames in v17, so it stays confusing. Recorded in [../08-future-scope.md](../08-future-scope.md).

### ERPNext hands out the Customer role by itself

`erpnext/portal/utils.py:16` adds the `Customer` role to a user whenever a `Contact` of theirs is linked to a `Customer`.

```python
if link.link_doctype == "Customer" and "Customer" not in roles:
    doc.add_roles("Customer")
```

After v17, Helpdesk portal contacts are linked to `Customer`. So ERPNext gives them the `Customer` role on its own, and the ERPNext website portal comes with it. Nobody in Helpdesk decides this and nobody sees it happen.

The role works in two layers, and the second one is the problem.

Doctype permissions are thin. `select`, `report`, `print`, `email`, `export` and `share` on `Customer Group` and `Territory`. Nothing on `Customer` itself.

Portal menu items are not thin. `erpnext/hooks.py` ships nine pages gated on this role:

```
/quotations         Quotation
/orders             Sales Order
/invoices           Sales Invoice
/shipments          Delivery Note
/project            Project
/timesheets         Timesheet
/material-requests  Material Request
/addresses          Address
/issues             Issue
```

Each page lists that customer's own documents, scoped by permission query conditions, so it is not a cross customer leak. It is an entire ERPNext customer portal appearing for someone who only wanted to raise a support ticket.

`/issues` points at a deprecated doctype, so it would show a dead page as well.

Unresolved, but the likely answer is that Helpdesk suppresses that assignment for its own portal users rather than softening the rule. Tracked in [../09-open-questions.md](../09-open-questions.md) as critical.

## Customer is the only doctype Helpdesk grants

| Doctype | Agent | Agent Manager |
|---|---|---|
| `Customer` | read | read, write, create |

Every ticket shows its customer, and the dialogs create and edit them. That cannot depend on a site admin remembering to grant an ERPNext role, so Helpdesk grants it on install.

## The three ERP tabs grant nothing

`Warranty Claim`, `Maintenance Visit` and `Installation Note` get no Helpdesk permission rows at all.

The tabs fetch through `get_list`, which already enforces permissions. So a tab fills for a user who already has the ERPNext role, and stays empty for one who does not. Since a tab only renders when it has rows, an agent without those roles never sees it. No empty state, no error.

| Doctype | Who sees it today |
|---|---|
| `Warranty Claim` | `Maintenance User` |
| `Maintenance Visit` | `Maintenance User` |
| `Installation Note` | `Sales User` |

An agent who also holds `Maintenance User` sees warranty claims and maintenance visits. One who also holds `Sales User` sees installation notes. Nobody has to configure Helpdesk for that to work.

The site admin decides who gets those roles, which is the right place for the decision. It also means Helpdesk never writes permission rows onto ERPNext doctypes and so cannot widen access by accident.

Deep links work for exactly the people who can read the document, because it is the same permission check.

For reference, the current Support module agent role is `Support Team`. It exists only on `Issue` and goes away with it.

## Three things to know

**Roles are global, not per app.** An agent with read on `Customer` can also open `Customer` in desk and see every field on it. There is no way to give someone read inside Helpdesk only.

**Helpdesk defines the `Customer` permission rows, not ERPNext.** If ERPNext shipped them it would reference Helpdesk roles that do not exist on an ERPNext only site. Helpdesk adds them on install.

There is already a precedent. `update_agent_role_permissions` and `add_agent_manager_permissions` in `helpdesk/setup/install.py` add `File`, `Contact`, `Email Account`, `Communication`, `User Invitation`, `Role` and `Assignment Rule`. The ERPNext doctypes go in the same place.

**Agents already have desk access.** The `Agent` role turns on the desk UI flags in `helpdesk/setup/install.py:177` and never sets `desk_access = 0`, so the deep links to `/app/warranty-claim/...` open. The portal roles `HD Customer` and `HD Customer Manager` set `desk_access = 0` explicitly at line 225, which is what keeps the hard rule above true.

## What an agent can see on Customer

`Customer` is wide. Read permission includes fields Helpdesk has no use for.

```
credit_limits          accounts              tax_id
default_price_list     payment_terms         sales_team
loyalty_program        default_commission_rate
tax_withholding_category
```

Seven roles already read all of them: Sales User, Sales Manager, Sales Master Manager, Stock User, Stock Manager, Accounts User and Accounts Manager.

No field on `Customer` sits above permission level zero. The doctype defines permlevel 1 rules for Sales User and Sales Master Manager, but no field is assigned to permlevel 1, so those rules do nothing. Every role with read gets every field.

So a Stock User can already read a credit limit and a tax ID. Adding Agent is one more role on a list, not a new class of exposure.

Accepted for v17, and treated as a display question rather than a permission one. See [../01-decisions.md](../01-decisions.md).

Helpdesk renders a curated set of `Customer` fields, and a site that wants `credit_limit` in front of its agents adds it through the layout editor. `HD Field Layout` already stores per doctype layouts and needs no new doctype to point at `Customer`.

No permission level is added. Moving the financial fields to permlevel 1 would tighten them for the seven roles that read them today, which makes it an ERPNext change with a blast radius well outside Helpdesk.

Tracked in [../09-open-questions.md](../09-open-questions.md).

## Existing data

None. Permission rules are added on install.

## Risks

This is the largest new attack surface in v17, and the failure is silent. A missing permission query condition does not throw, it returns rows.

Every whitelisted method that serves ERPNext data to the portal has to be checked by hand. There is no framework level guarantee.

## Verification

Before release, for each ERPNext doctype reachable from Helpdesk:

- log in as a portal user and confirm no direct read path exists, through the API as well as the UI
- log in as an agent and confirm read works and write does not
- log in as an agent manager and confirm write works
- check every Helpdesk whitelisted method that returns ERPNext data for what it leaks in its response

The existing QA logins cover the first three. The fourth is a code review, not a test.

## Open questions

Tracked in [09-open-questions.md](../09-open-questions.md).

- **Critical.** Whether Helpdesk suppresses the automatic `Customer` role assignment in `erpnext/portal/utils.py`, or the portal rule gets softened.
- Whether Helpdesk's portal role merges with ERPNext's `Customer` role or stays separate.
- None on the financial fields. Settled in [../01-decisions.md](../01-decisions.md).

---

[← SLA](sla.md) · [Schema changes →](schema-changes.md)
