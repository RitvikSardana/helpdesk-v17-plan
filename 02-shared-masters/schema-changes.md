# Schema changes

[Index](../README.md) · [Shared masters](README.md)

## What this is

Every schema change in v17, in one list, so nobody has to read six sections to find out what a doctype looks like afterwards. The reasoning stays in the section each change comes from. This page is the register.

Changes are grouped by who owns the doctype, because that decides who writes the patch and when it runs. ERPNext's patches run before Helpdesk's, though not on every site by default. See [../05-upgrade.md](../05-upgrade.md). Nothing in v17 changes a Framework doctype.

## ERPNext gains fields

These are proposed as core ERPNext fields. Every one is useful to consumers other than Helpdesk, which is the test for going into core rather than being added as a custom field.

### `Customer`

| Field | Type | From | Note |
|---|---|---|---|
| `domain` | Data | `HD Customer.domain` | match an incoming email to a customer. CRM wants it too, `CRM Organization` has it |
| `country` | Data | `HD Customer.country` | shown in the Helpdesk customer dialog |

Both are contingent. If ERPNext takes them as core fields the Helpdesk patch asserts they exist. If it does not, the same patch creates them as Custom Fields and then migrates. Either way Helpdesk's patch runs the same check first. See [customer.md](customer.md).

One property change, not a field: `Customer.image` is `hidden: 1` today and has to be unhidden, since the Helpdesk customer view shows a logo.

### `Holiday List`

| Field | Type | From | Note |
|---|---|---|---|
| `description` | Data | `HD Service Holiday List.description` | |
| `recurring_holidays` | JSON | `HD Service Holiday List.recurring_holidays` | |

`Holiday` needs nothing. It is already a superset of `HD Holiday`. See [holiday-list.md](holiday-list.md).

### `Service Level Agreement`

The largest set. It takes Helpdesk's matching logic and the generic selector.

| Field | Type | From | Note |
|---|---|---|---|
| `rank` | Int | `HD Service Level Agreement.rank` | tie break when several agreements match |
| `description` | Data | same | |
| `condition_type` | Select | new | `Python` / `Filters`, the `Notification` pattern |
| `filters` | Code | `condition_json` | JSON |
| `filters_editor` | HTML | new | the core filter builder |
| `priority_doctype` | Link (DocType) | new | what the target selector points at |
| `default_status` | Dynamic Link | `default_ticket_status` | generalised off tickets |
| `reopen_status` | Dynamic Link | `ticket_reopen_status` | generalised off tickets |

`condition` already exists on both sides, same fieldname, same `PythonExpression` options. Nothing to add.

Two existing fields change type rather than being added:

| Field | Was | Becomes | Why |
|---|---|---|---|
| `default_priority` | Link to `Issue Priority` | Dynamic Link on `priority_doctype` | ERPNext cannot link to a Helpdesk master |
| `Service Level Priority.priority` | Link to `Issue Priority` | Dynamic Link | same, and the child carries `priority_doctype` down from the parent on save |

Name kept: `default_service_level_agreement`, not Helpdesk's `default_sla`. The doctype is ERPNext's.

See [sla.md](sla.md).

## ERPNext loses fields

Deprecated, not dropped. The column stays, the field is hidden and read only, existing values stay readable.

| Field | Why |
|---|---|
| `Service Level Agreement.pause_sla_on` | status category replaces it |
| `Service Level Agreement.sla_fulfilled_on` | same |

Both are child tables whose only consumer was `Issue`. `Issue` is deprecated and its clock never runs again, so nothing is left that needs them. An old agreement still shows what it was configured with.

## Helpdesk changes link targets

No new Helpdesk fields for the masters work. What changes is where the existing links point.

| Field | Was | Becomes |
|---|---|---|
| `HD Ticket.customer` | `HD Customer` | `Customer` |
| `HD Ticket.sla` | `HD Service Level Agreement` | `Service Level Agreement` |

This is the clean break in [../01-decisions.md](../01-decisions.md). No shim, release notes carry it.

## Helpdesk gains fields

Only the HRMS integration, all three in `HD Settings`. See [agent.md](agent.md).

| Field | Type | Note |
|---|---|---|
| leave integration enabled | Check | off by default |
| status to use on leave | Link (`HD Agent Status`) | must have category Unavailable |
| reset when leave ends | Check | on by default |

`HD Agent` gets nothing. `HD Agent Status.category` already exists with Active, Away and Unavailable.

## Doctypes deprecated

Read only, no create button, badge on the form, rows stay readable, API still answers. Nothing is deleted and no rows are migrated out. See [../01-decisions.md](../01-decisions.md).

| Doctype | Owner | Replaced by |
|---|---|---|
| `HD Customer` | Helpdesk | `Customer` |
| `HD Customer Member` | Helpdesk | depends on the contacts question |
| `HD Service Holiday List` | Helpdesk | `Holiday List` |
| `HD Holiday` | Helpdesk | `Holiday` |
| `HD Service Level Agreement` | Helpdesk | `Service Level Agreement` |
| `HD Service Day` | Helpdesk | `Service Day` |
| `HD Service Level Priority` | Helpdesk | `Service Level Priority` |
| `Pause SLA On Status` | ERPNext | status category |
| `SLA Fulfilled On Status` | ERPNext | status category |

`Issue`, `Issue Type` and `Issue Priority` are also deprecated. They belong to the Support cleanup, not to shared masters. See [../03-support-module-cleanup.md](../03-support-module-cleanup.md).

**A deprecated parent keeps its child tables.** `HD Service Day` and `HD Service Level Priority` are field for field identical to ERPNext's and the instinct is to delete them. They stay, because `HD Service Level Agreement` stays readable and a parent cannot render rows whose child doctype is gone. Deprecate the child alongside the parent, delete neither.

## Doctypes added

| Doctype | Owner | Why |
|---|---|---|
| merge log | migrator app | records what the migrator merged, and what it merged from. See [../04-migrator.md](../04-migrator.md) |

Nothing else. Every other change lands on a doctype that already exists.

## The app field

Some records need to know which app they belong to. Most do not, because they already point at a document and the app can be worked out from that.

### The rule

Store an `app` field only when the record is about the app itself and there is no document to derive the app from.

If the record already points at a document, derive: document type gives the module, module gives the app. One meta lookup.

Do not store what you can derive. A stored value goes stale when a doctype moves apps, and v17 is a release where doctypes move apps.

### The pattern already in core

Two doctypes already do this, so there is a house style rather than a new mechanism.

`User Invitation` has `app_name`, a Select validated against `frappe.get_active_apps()`. It drives the invite hook, the redirect path, which roles are allowed, and a permission query condition that limits rows to the apps a user may see. See `user_invitation.py:254`.

`Notification Log` has `app`, a Select whose options are filled client side the same way `Module Def.app_name` does it, alongside `document_type` and `source_doctype`.

### Needs a stored app field

No existing doctype gains one in v17. Both that need it already have it.

| Doctype | State |
|---|---|
| `User Invitation` | already has `app_name`. Helpdesk moves onto it instead of its own invite path |
| `Notification Log` | already has `app`. The notification migration sets it, and that work is already merged |
| The migrator's merge log | new doctype in a new app, so the field is part of its definition rather than a change |

Anything new in v17 that is app level with no document behind it follows the same rule. Nothing else in v17 is.

### Derive instead

`Comment`, `ToDo`, `Tag Link`, `File`, `Activity`, `Call Log`, `Webhook`, `Notification`, `Assignment Rule`, `Dashboard`, `Number Card`, `Service Level Agreement`.

Every one already carries a document type.

`Service Level Agreement` is the clearest case. `document_type` is finer grained than `app`, because two doctypes in the same app can want different agreements. See [sla.md](sla.md).

### Not the right tool for masters

`HD Ticket Priority` and `HD Ticket Type` are shared in the sense that more than one thing reads them. An app field on them would be wrong.

If a master ever needs splitting, scope it by document type the way SLA does. App is too coarse and it breaks the moment one app has two consumers with different needs.

## What does not change

Worth stating, because each of these was considered and rejected.

| Doctype | Why it stays |
|---|---|
| `HD Agent` | availability and the active flag are not HR concepts. See [agent.md](agent.md) |
| `HD Team` | `Sales Team` and `Employee Group` are not equivalents, `User Group` is one field with no consumers. See [team.md](team.md) |
| `HD Ticket Priority` | the Dynamic Link removes the reason to move it. See [sla.md](sla.md) |
| `HD Ticket Type` | nothing outside the Support module ever referenced `Issue Type`. See [../03-support-module-cleanup.md](../03-support-module-cleanup.md) |
| `Warranty Claim`, `Maintenance Visit`, `Installation Note` | not deprecated, not touched, surfaced as tabs in the Helpdesk customer view |

No doctype is renamed in v17. Doctype names are global and every Frappe app prefixes, so dropping `HD` is all or nothing. Renames are future scope. See [../08-future-scope.md](../08-future-scope.md).

## Order

The patches have to run in this order. Install order usually gives it, but not always, and the upgrade checks before running anything. See [../05-upgrade.md](../05-upgrade.md).

1. ERPNext adds its core fields and changes the two link fields to Dynamic Links.
2. Helpdesk asserts the contingent `Customer` fields exist, creating Custom Fields if they do not.
3. Helpdesk migrates data and repoints its links.
4. Helpdesk marks its doctypes deprecated.

Step 2 is the only one that branches, and it branches on whether ERPNext accepted `domain` and `country` into core.

## Risks

The contingent fields are the sharp edge. If ERPNext adds `domain` after Helpdesk has already created it as a Custom Field, the site ends up with both. The patch has to check the Custom Field as well as the core field, and remove its own if core has taken over.

Changing `Service Level Priority.priority` from Link to Dynamic Link rewrites a column's meaning on every existing ERPNext agreement. The patch has to fill `priority_doctype` with `Issue Priority` on those rows, otherwise the Dynamic Link has nothing to resolve against.

`Customer.image` being unhidden is a visible change to the ERPNext customer form for every site, including ones that never install Helpdesk.

## Open questions

Tracked in [09-open-questions.md](../09-open-questions.md).

- Contacts: child table or standard Dynamic Link. Decides whether `HD Customer Member` is deprecated or replaced. Critical.

---

[← Permissions](permissions.md) · [Support module cleanup →](../03-support-module-cleanup.md)
