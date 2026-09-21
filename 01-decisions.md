# Decisions already made

[Index](README.md) · previous: [00-overview.md](00-overview.md) · next: [02-shared-masters](02-shared-masters/README.md)

Settled. The rest of the plan is written on top of these. They do not get re-argued in later sections.

## All four apps are always installed

ERPNext, Helpdesk, CRM and HRMS ship and run together. Helpdesk standalone is not a supported configuration in v17.

Consequence: shared masters can live in ERPNext without a thin masters app. The install matrix collapses to one.

**An existing standalone site cannot reach v17 and stay standalone.** It installs ERPNext, or it stays on the last 1.x. There is no third path, and moving the shared masters somewhere smaller than ERPNext to preserve one is rejected.

The ERPNext requirement is not what makes that hard. The Framework's own v17 breaking changes mean no site takes this upgrade casually, standalone or not. Requiring one more app is a small addition to a migration that was already a bench level event. See [05-upgrade.md](05-upgrade.md).

## Helpdesk renumbers to 17

Helpdesk's own version line ends and it matches ERPNext and the Framework.

## Versions are locked

Helpdesk v17 requires ERPNext v17 and Framework v17. The upgrade refuses to run on a mismatched bench.

Consequence: a Helpdesk patch can assume ERPNext v17 has already migrated. Patches run in `installed_apps` order, which is install order, so this holds on a fresh bench but not on every existing site. The upgrade refuses rather than running out of order. See [05-upgrade.md](05-upgrade.md).

## No renames

No doctype loses the `HD` prefix in v17. Not `HD Ticket`, not the masters that change apps.

Doctype names are global across a site. Taking `Ticket`, `Team`, `View`, `Agent` and `Settings` means Helpdesk owns those names forever, and every other Frappe app prefixes its doctypes for exactly that reason.

Renames go to `08-future-scope.md`.

## SLA lives in ERPNext

One SLA engine, owned by ERPNext, used by Helpdesk and CRM. Helpdesk's matching logic wins, ERPNext's generic scaffolding stays.

Consequence: after `Issue` is deprecated, ERPNext hosts the engine and runs nothing on it. Accepted.

## Deprecate, never delete

A doctype whose data has moved stays in place: read only, no create button in the UI, a deprecated badge on the form. Rows stay readable. The API still works, so integrations do not break.

This applies in both directions. ERPNext's `Issue` and its masters stay. Helpdesk's `HD Customer` and `HD Service Level Agreement` stay.

No data is migrated out of `Issue`. Existing issues sit there as an archive.

## The Support module shrinks, it does not go away

`Issue`, `Issue Type` and `Issue Priority` are deprecated. `Service Level Agreement` stays and gets rewritten. `Warranty Claim` stays exactly as it is. Maintenance is untouched.

## Migration never blocks the upgrade

The upgrade patch creates a `Customer` for every `HD Customer` with no matching at all. The site works the moment it finishes.

Deduplication is a separate app someone runs later, or never.

## A portal user reads nothing in ERPNext

Hard rule. Anything a customer sees that originates in ERPNext goes through a Helpdesk API that fetches it with elevated permission and returns only chosen fields.

Agents are different. Agent reads, Agent Manager writes, through ordinary role permissions on the ERPNext doctypes.

## Islands, not a shared shell

Each app keeps its own SPA and its own navigation. The connection between them is deep links.

An agent is on `/helpdesk/tickets/123`. They click a warranty claim in the customer view. The browser goes to `/app/warranty-claim/SER-WRN-0001`, which is desk: different sidebar, different styling, different shortcuts. Back is the browser back button. Helpdesk keeps its own notification bell and its own search, and so does every other app.

A shared shell and app switcher is future scope.

## Clean break on the API

`HD Ticket.customer` starts pointing at `Customer`. `HD Ticket.sla` starts pointing at `Service Level Agreement`. Anyone resolving those links gets a different record.

No compatibility shim. It goes in the release notes.

## No rollback

If an upgrade goes wrong the answer is restore from backup. Patches are not reversible.

## Financial fields on Customer are a display question

An agent reading `Customer` also reads `default_currency`, `credit_limit`, `payment_terms`, `tax_id` and the rest. No field on `Customer` sits above permission level zero today, and seven roles already read them.

No permission level is added, in ERPNext or in Helpdesk. Helpdesk decides what an agent sees, and it renders a curated set of `Customer` fields. A site that wants `credit_limit` in front of its agents adds it through the layout editor.

`HD Field Layout` already does this. It is a per doctype, per user layout store with a `document_type` link and an `apply_to_system` flag, used today for the agent home page. Pointing it at `Customer` needs no new doctype.

Being clear about what this is: display, not access. An agent with read permission on `Customer` can still reach every field over the API or through a report. The layout decides what Helpdesk renders, not what the role can read. Accepted, because agents are employees rather than portal users, and the untrusted case is covered by the portal rule above.

## The merge log belongs to the migrator app

Not Helpdesk, not ERPNext. The records being merged belong to ERPNext, the tool belongs to nobody in particular, and the log is the tool's. CRM's merges land in the same table for the same reason.

Consequence: the upgrade patch cannot write to it. A Helpdesk patch cannot insert into a doctype from an app that may not be installed, and making the migrator a hard dependency of the upgrade would contradict "migration never blocks the upgrade" above.

So the patch records what it created on its own side, on the already deprecated `HD Customer`, and the migrator reads that on first run to build its queue and its log. No cross app write, no install order dependency, and a site that adds the migrator months later still finds the list waiting. See [04-migrator.md](04-migrator.md).

## Field conflicts are shown, never resolved by default

Step 2 of the migrator copies `HD Customer` values onto the `Customer` a ticket now points at. Where both sides hold a value and the values differ, the migrator does not pick.

There is no default worth shipping. Helpdesk is the fresher source for support data, `domain` and `email_id` and `mobile_no`, because agents correct it while working a ticket. ERPNext is the fresher source for anything billing touched. A rule that is right for one field is wrong for the next.

So the tool finds the conflicts and hands them back. It knows both sides already, because it has to read them to copy them, which means it can list exactly which rows differ on which field and show the two values side by side. The user picks, per field, and can apply that pick to every row on that field in one action.

A site with no conflicts never sees the screen. See [04-migrator.md](04-migrator.md).

## CRM is another owner

CRM work is not in this plan. The one thing Helpdesk needs from CRM is that `CRM Organization` folds into `Customer`, giving both apps a shared key. That is confirmed and tracked as a dependency in `07-order-of-work.md`.

## Public commitment to keep in mind

On the forum thread about removing the Support module, Frappe said publicly that the module would not be deprecated until Helpdesk reaches parity with it, and named warranty claims as the gap.

That commitment is met by not deprecating anything it covers. `Warranty Claim` stays in ERPNext and Helpdesk surfaces it. `Issue` never had item or serial tracking, so deprecating it removes nothing the objection was about.

https://discuss.frappe.io/t/support-module-to-be-removed-in-v17/162745

---

[← Overview](00-overview.md) · [Shared masters →](02-shared-masters/README.md)
