# Upgrade

[Index](README.md) · previous: [04-migrator.md](04-migrator.md) · next: [06-app-integration.md](06-app-integration.md)

## Helpdesk does not own this upgrade

Start here, because the rest of the section only makes sense once this is clear.

v17 is a bench level event. The Framework goes to 17 and ERPNext goes to 17, each with its own breaking changes and its own migration story, both larger than Helpdesk's. By the time Helpdesk's patches run, a long migration has already happened in two other apps.

So this section is not "how a Helpdesk site upgrades". Helpdesk cannot describe that, and should not try. It is three things Helpdesk owns:

1. **Its own patches**, and where they sit in the run.
2. **Its own breaking changes**, for release notes.
3. **What it refuses**, so its patches never run against a bench that is not ready.

Everything else belongs to the Framework and ERPNext. In particular Helpdesk does not decide, document or promise how a production site gets to v17. In place `bench update` is the path on a development bench. What a live site does is the suite's call, not Helpdesk's, and writing a procedure here would be inventing one.

That makes the ordering and the refusal in this section **more** important than they would be for a self contained app, not less. Helpdesk's patches run inside someone else's migration, on a schema two other apps have just rewritten. The only thing standing between that and a corrupted site is a check that runs first.

## What changes

When the run finishes, every ticket points at an ERPNext `Customer` and an ERPNext `Service Level Agreement`.

Helpdesk's patches never ask a question and never stop halfway. Anything that needs a human is the migrator's job, afterwards. See [04-migrator.md](04-migrator.md).

## The version lock

Helpdesk v17 requires ERPNext v17 and Framework v17. A mismatched bench is refused rather than migrated.

`required_apps` in `helpdesk/hooks.py` gains `erpnext`, alongside the `telephony` already there. That makes `install_app` install ERPNext first on a fresh bench, recursively, before Helpdesk.

CRM and HRMS are not added. Helpdesk does not depend on either, and requiring an app it does not use would be wrong even though the release ships all four. The four app assumption in [01-decisions.md](01-decisions.md) is about what the release is tested as, not about what Helpdesk declares.

The version itself is checked in `before_migrate`, reading `frappe.utils.get_installed_apps_info()`, which carries each app's version and branch. Wrong major version on ERPNext or the Framework and the migration refuses with a message naming what it found.

Refusing is the point. A half upgraded bench is worse than one that will not start, because the patches would run against a schema that has not been prepared.

## Patch order is install order, and that is a problem

This is the sharpest thing in this section and it is easy to miss.

Patches run app by app, in the order the apps sit in `installed_apps`:

```python
for app in frappe.get_installed_apps():
    patches.extend(get_patches_from_app(app, patch_type=patch_type))
```

`frappe/modules/patch_handler.py:117`

And `installed_apps` is a **global stored in the database, appended to as apps are installed**. `add_to_installed_apps` appends. Nothing sorts it.

So on a fresh install the order is right, because `required_apps` makes ERPNext go in first.

**On an existing site it may not be.** A site that installed Helpdesk in 2024 and added ERPNext in 2025 holds `["frappe", "helpdesk", "erpnext"]`. Adding `required_apps` to Helpdesk does not reorder that list. On that site Helpdesk's v17 patches run **before** ERPNext's, and Helpdesk's migration would look for `Customer.domain` and the Dynamic Link conversion before ERPNext has made them.

### The fix already exists in the Framework

`update_installed_apps_order` is a whitelisted method on `Installed Applications` that rewrites the global. An admin can reorder apps from that page, and the Framework forces `frappe` to stay first.

So Helpdesk's `before_migrate` check does two things:

1. Refuse if ERPNext or the Framework is on the wrong version.
2. Refuse if `erpnext` does not come before `helpdesk` in `installed_apps`, with a message pointing at Installed Applications.

Both are read only checks that run before a single patch. Neither reorders anything by itself, because silently rewriting which app's patches run first on someone's site is not a thing to do without being asked.

## The order the patches run in

```
1  ERPNext   adds domain and country to Customer, if they are going into core
2  ERPNext   adds description and recurring_holidays to Holiday List
3  ERPNext   adds the new Service Level Agreement fields
4  ERPNext   converts default_priority and Service Level Priority.priority
             to Dynamic Links, backfilling priority_doctype with Issue Priority
5  ERPNext   removes the three generic reads of track_service_level_agreement
6  ERPNext   deprecates Issue, Issue Type, Issue Priority and their reports
7  Helpdesk  asserts the Customer fields exist, creating Custom Fields if not
8  Helpdesk  links each HD Customer to a Customer through erpnext_customer
             or an exact name match, creating one only when neither applies;
             one Holiday List per HD Service Holiday List unless ERPNext
             already has lists; one Service Level Agreement per HD SLA
9  Helpdesk  repoints HD Ticket.customer, HD Ticket.sla and the
             User Invitation.customer custom field
10 Helpdesk  deprecates its own replaced doctypes
```

Step 5 is not optional and it is not cosmetic. `track_service_level_agreement` defaults to off and gates the engine's entry point, so a site that never used `Issue` would upgrade onto an SLA engine that silently finds no agreement. See [03-support-module-cleanup.md](03-support-module-cleanup.md).

Step 7 is the only one that branches, on whether ERPNext accepted `domain` and `country` into core. See [02-shared-masters/schema-changes.md](02-shared-masters/schema-changes.md).

### Step 8 branches on whether the ERPNext integration was on

The easiest thing in the plan to misread, so be exact.

Helpdesk already ships a two way sync between `HD Customer` and `Customer`, gated by `ERPNext HD Settings.enabled`. When it is on, `HDCustomer.after_insert` creates the `Customer`, `Customer.after_insert` creates the `HD Customer`, and `set_links` writes both link fields:

```
HD Customer.erpnext_customer = <Customer>
Customer.hd_customer         = <HD Customer>
```

So the patch is not looking at one situation. It is looking at three.

**Integration on.** Every row created since the toggle is already paired, verified, in both directions. The patch reads `erpnext_customer` and repoints. No matching, no creating, no duplicates, no human. There is no customer migration on this site.

**Integration off, ERPNext installed.** No links exist. Two customer lists that grew separately. This is where all the work is.

**No ERPNext.** Nothing to point at. See [Upgrading a standalone Helpdesk](#upgrading-a-standalone-helpdesk).

The toggle alone does not pair anything retroactively, but there is a button that does. `sync_all_customers` in `helpdesk/integrations/erpnext/api.py:59` links every unlinked row that has an exact name match on the other side and creates the counterpart for the rest, in both directions. It is exposed as **Sync Customers** in the integration settings and gated by `in_sync()`.

So the real question per site is not whether the toggle was on. It is whether that button was ever pressed. `in_sync()` answers it in two counts:

```python
if frappe.db.count("HD Customer", {"erpnext_customer": ["is", "not set"]}): return False
if frappe.db.count("Customer", {"hd_customer": ["is", "not set"]}): return False
```

A site that is `in_sync` needs nothing from step 8 but the repoint.

### Three tiers, none of them a guess

Per `HD Customer`, in order:

| Signal | Patch does |
|---|---|
| `erpnext_customer` is set | link to it, the site already verified this |
| `customer_name` matches a `Customer` exactly | link to it |
| neither | create one |

Tier 2 is not matching. It is the same identity test `Customer.get_customer_name` applies one line before it decides to suffix. Without it the patch would insert `Netflix - 1` next to an existing `Netflix`, a duplicate the upgrade invented for no reason.

How much each tier carries depends on the site:

| Site | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| No ERPNext | nothing | nothing | everything |
| Integration off | nothing | the overlap | the rest |
| Integration on, never synced | rows touched since the toggle | most of the tail | few |
| Integration on and `in_sync` | everything | nothing | nothing |

The patch never scores, never merges and never asks. It always succeeds.

### Reuse the sync, do not rewrite it

Tiers 2 and 3 are `sync_all_customers`. Same rule, same two directions, already shipped and already exercised by the button. The patch should call the existing logic rather than grow a second implementation that drifts from it.

Two things to know before relying on it. It creates with `customer_name` and `image` only, so it has the same field gap as the patch. And its `elif not frappe.db.exists(...)` skips a row whose name is already taken by a linked record on the other side, so `in_sync()` can still read false after a run. Neither blocks reuse; both need handling.

### Linking is not copying

Tier 3 copies the `HD Customer`'s field values, because the record it creates is empty. Tiers 1 and 2 link to a `Customer` that already exists and holds its own values, so the data does not move by itself. Two kinds of field, two answers.

**Standard Helpdesk fields with a home on `Customer`.** `domain` and `country`, and whatever else [02-shared-masters/schema-changes.md](02-shared-masters/schema-changes.md) lands. The patch copies these onto a record it **creates**, because it is inserting anyway and the alternative is a bare name. It copies nothing onto a record it **linked to**. That record holds its own values, and deciding whose value wins is a question rather than a copy.

**Custom fields somebody added to `HD Customer`.** The patch does nothing. There is no field on `Customer` to write into, and it cannot invent one: it does not know which custom fields should come across, what they should be called, or what type they should be. That is the whole of the migrator's step 2. See [04-migrator.md](04-migrator.md).

There is a second reason to keep field work out of the patch, and it applies to created records too. `rename_doc(merge=True)` deletes the record that loses. Anything the patch wrote onto a `Customer` the migrator later merges away is destroyed with it. So field data moves once, in the migrator, after the merge decisions are settled.

Nothing is lost in the meantime. `HD Customer` is deprecated, not deleted, so the rows stay readable and the migrator can still reach every value. See [03-support-module-cleanup.md](03-support-module-cleanup.md).

| | Patch, during the upgrade | Migrator, whenever |
|---|---|---|
| Trust an existing `erpnext_customer` mapping | yes | already done |
| Link on an exact name match | yes | already done |
| Create a `Customer` when neither applies | yes | no |
| Repoint `HD Ticket.customer` and `.sla` | yes, changes the link target | yes, through `rename_doc` |
| Score names against ERPNext's list | no | yes |
| Merge two records into one | no | yes |
| Copy Helpdesk field values into a `Customer` it created | yes | n/a |
| Copy them into a `Customer` it linked to | no | yes, including conflicts |
| Map a field that has no obvious home | no | yes |
| Ask a human anything | never | every merge |

Holiday lists and agreements have no tier 1, because there is no link field to read. Holiday lists have their own rule instead: if ERPNext already has `Holiday List` records the patch uses those and creates none. See [02-shared-masters/holiday-list.md](02-shared-masters/holiday-list.md).

### Tier 3 creates duplicates, and that is the correct trade

`HD Customer` Acme Corp against `Customer` Acme Corporation is not an exact match, so the patch creates a second `Customer`. Two records for one company, visible in ERPNext's customer list and its reports.

That is deliberate. The alternatives are worse:

| Instead | Why not |
|---|---|
| Run the fuzzy matcher in the patch and auto link above a cutoff | silent wrong merges, in a patch, with no undo |
| Leave unmatched tickets with `customer` empty | data loss |
| Refuse to upgrade until the migrator has run | breaks "migration never blocks the upgrade" in [01-decisions.md](01-decisions.md) |

What is worth fixing is not the duplicate, it is the review set. Without help the migrator opens on a site where every `Customer` looks equally suspect.

**So the patch marks every `Customer` it creates.** The mark goes on the `HD Customer` side, which Helpdesk owns and is deprecating anyway. It does not go in the merge log, because the log belongs to the migrator app and the patch cannot write into an app that may not be installed. See [01-decisions.md](01-decisions.md).

The migrator turns those marks into its queue on first run, whenever that is. So the queue is the patch's own creations rather than the whole customer table, and when it is empty there is nothing left to merge.

Field mapping is separate and survives an empty queue. It is decided once for the pair and applies to every row, linked or created. A site with the integration on and a custom field on `HD Customer` has no merges and still has a step 2.

### Tickets get repointed twice

Once by the patch, once per merge. Different operations, and only the first is Helpdesk's code.

**Step 9, the patch.** Changes `HD Ticket.customer` from a link into `HD Customer` to a link into `Customer`, and writes the new name. Helpdesk writes this.

**Every merge, afterwards.** `rename_doc("Customer", "Acme Corp", "Acme Corporation", merge=True)` repoints every reference to the surviving name, and `HD Ticket.customer` is one of them. The framework does this. The migrator collects the decision and calls it; Helpdesk writes no reference walker. See [04-migrator.md](04-migrator.md).

So a ticket on a site that ran both apps without the integration ends up pointing at `HD Customer` Acme Corp, then `Customer` Acme Corp, then `Customer` Acme Corporation. All three are correct at the time.

A ticket on an integration on site is repointed once and never touched again.

### Whether step 9 changes the value depends on the site

Not always a link target change with the same name on both sides.

```
HD Customer          autoname: field:customer_name, and customer_name is unique
Customer             named by Selling Settings.cust_master_name
                     default "Customer Name", so named by customer_name too
```

So on a default site the linked or created `Customer` carries the same name and step 9 rewrites the field without changing the string.

Tier 2 is what keeps that true. Without it `Customer.get_customer_name` would suffix on collision and `Netflix` would become `Netflix - 1`, changing the value on exactly the sites where both lists hold the same company under the same name. Linking instead of creating removes that case.

One case still changes the value: **a site that set `cust_master_name` to `Naming Series`.** Then every created `Customer` is named `CUST-2026-00001` and every ticket's customer value becomes an opaque code. Nothing is broken, but anything outside Helpdesk that matched on the customer string stops matching. Tier 1 and tier 2 rows are unaffected, since nothing is created for them.

This is the concrete form of the API break in [Breaking changes](#breaking-changes), and it is worth naming in the release notes rather than leaving people to find it.

## What the upgrade does not do

No scoring. No merging. No questions.

Where a `Customer` is already linked, or carries exactly the same name, the patch uses it. Where nothing matches exactly it creates, including for companies that are obviously the same as a `Customer` already there under another name. A site that ran both apps without the integration comes out of the upgrade with duplicates, on purpose. A site that had the integration on comes out with none.

The alternative is an upgrade that pauses on a customer it cannot resolve, which means an upgrade that can leave a site half migrated. That is worse than duplicates, which are visible, harmless, logged and fixable whenever someone has time.

No `Issue` row becomes an `HD Ticket`. No deprecated row is deleted.

## Breaking changes

Helpdesk's own. The Framework's and ERPNext's are theirs to publish, and there are more of them.

A clean break, announced in the release notes, with no compatibility shim.

| What | Was | Becomes |
|---|---|---|
| `HD Ticket.customer` | Link to `HD Customer` | Link to `Customer` |
| `HD Ticket.sla` | Link to `HD Service Level Agreement` | Link to `Service Level Agreement` |
| `User Invitation.customer` | Link to `HD Customer` | Link to `Customer` |

`User Invitation.customer` is a Custom Field from `helpdesk/patches/add_customer_field_in_user_invitation.py`. It is easy to miss and it is live: `HD Customer.unlink_invitations` writes to it. See [appendix/reference-counts.md](appendix/reference-counts.md).

Anything reading those fields over the REST API gets a name from a different doctype. On a site that ran both apps without the integration the value may also change, because Acme Corp from Helpdesk and Acme Corporation from ERPNext are two records until the migrator merges them.

What this hits, in rough order of how likely it is to bite:

```
REST integrations filtering or resolving ticket.customer
Webhooks with a condition on customer
Server scripts and client scripts
Custom reports and dashboards
Custom fields and property setters on HD Customer
Saved list views with a customer filter
```

A shim was considered and rejected. Keeping `HD Customer` readable and keeping `HD Ticket.customer` pointing at it are different things, and the second one means running both models at once for a release. Nothing is gained that the release notes cannot say more clearly.

The deprecated doctypes stay reachable over the API, so an integration that reads `HD Customer` directly keeps working and returns the frozen rows. It is the link on the ticket that moves.

## No rollback

There is none, and it is not only Helpdesk's to give. Reversing Helpdesk's patches on a bench whose Framework and ERPNext have also migrated would reverse nothing useful.

If an upgrade goes wrong, restore from backup.

Writing a reverse patch for this would mean reversing a merge of two doctypes across three masters, with references repointed in both directions, on a site that has been taking tickets since. It would be a second engineering project with worse testing than the first one.

The migrator's merge log has an undo, and that is a different thing: it reverses one merge shortly after it happened, not an upgrade. See [04-migrator.md](04-migrator.md).

So the release notes say take a backup, and the migrator's confirm step says it again.

## Which sites this is for

Helpdesk has never had a v15 or a v16. It is on a semver 1.x line, `v1.30.1` at the time of writing, and it has tracked the Framework through branches instead: `version-14`, `v16-support`. Its compatibility is declared in `pyproject.toml`:

```
frappe = ">=16.0.0-dev,<18.0.0"
```

So the upgrade is **the last 1.x release to 17**, not v16 to v17. There is no v16 to leave.

What "v16 users" means in practice is a site on Framework and ERPNext v16 running Helpdesk 1.x. That site upgrades all four apps at once, which is the same thing every other app in the suite asks of it.

**Supported and tested: the last 1.x release, on v16, to 17.** One hop, on a real site copy.

**Works but untested: older 1.x.** Patches are cumulative, so a site two years behind runs every intervening Helpdesk patch and then v17's. Nothing stops it. Nobody tests it either, and the release notes should say so rather than imply a guarantee.

**Not supported: skipping ERPNext v17.** That is the cross app check above, and it is the only thing the upgrade refuses on.

## How the ecosystem has handled this

Worth knowing before adding a gate, because the precedent is consistent and it is not what you might expect.

**Patches are never removed.** ERPNext's `patches.txt` still lists `erpnext.patches.v4_2.update_requested_and_ordered_qty` and `erpnext.patches.v8_1.removed_roles_from_gst_report_non_indian_account`, dated 2018. The Framework's goes back to `v10_0`. 482 lines in ERPNext, appended to for a decade, nothing taken out. So any site, however old, has a mechanically complete path forward.

**There is no version gate anywhere.** Nothing in the Framework or ERPNext refuses to migrate because the site is too far behind. No `before_migrate` check, no minimum version assertion, no error string. The machinery is deliberately permissive.

**Compatibility is declared, not enforced at migrate time.** `[tool.bench.frappe-dependencies]` in `pyproject.toml` and `required_apps` in `hooks.py` state a range, and bench checks it when the app is installed or updated. That is the existing mechanism and it already covers the "too old" case without any new code.

**Majors were tracked by branch.** `upstream/version-14`, `origin/v16-support`. The answer to "which Helpdesk works with my Frappe" has always been a branch, not a runtime check.

So the position that follows from the precedent: **declare the range, do not build a "you are too old" gate.**

The version check in [The version lock](#the-version-lock) is not that, and the distinction matters. It does not ask how old the site is. It asks whether the bench is internally coherent right now, because v17 is the first Helpdesk release whose patches reach into another app's schema. A site with Helpdesk v17 and ERPNext v16 is not behind, it is broken, and the patch would corrupt it rather than fail cleanly. That is a different argument and it is why the check earns an exception to the house style.

## Upgrading a standalone Helpdesk

Settled in [01-decisions.md](01-decisions.md): it cannot, and stay standalone. It installs ERPNext or it stays on the last 1.x. Written out here because the reasoning is worth keeping.

The version jump is not the problem. Nothing compares Helpdesk's own version number. Patches run from `patches.txt` in file order and are tracked by name in the `Patch Log`, so the `v12_0` style folder names are organisation, not a mechanism. A 1.x site runs every patch it has not run and lands on 17. Going from 1.30 to 17 is a normal forward jump.

The problem is ERPNext.

Helpdesk's `required_apps` is `["telephony"]` today. **Standalone is not a tolerated configuration, it is the supported one.** `HD Customer` exists precisely because there is no ERPNext `Customer` to link to. The product is sold and installed on its own.

So v17 asks every standalone site to install a full ERP to keep getting updates.

That reads worse than it is, because the ERPNext requirement is not what makes this upgrade hard. The Framework's own v17 breaking changes mean nobody takes it casually. Every site on this path is already doing a bench level migration across three apps, and Helpdesk adding a fourth is not the step that decides whether it is feasible.

### How big the ask actually is

Smaller than it first looks, which matters for the decision.

- ERPNext's `after_install` does **not** create a `Company`. The `Company` comes from the setup wizard, which nobody has to run.
- `Customer` has only two mandatory fields, `customer_name` and `customer_type`. There is no `company` field on it at all.
- `Customer Group` and `Territory` defaults come from the setup wizard's `install_fixtures`, not from `after_install`. `Customer.validate_customer_group` reads the group, so the upgrade has to create those defaults itself or confirm an insert survives without them. Worth testing early, because it decides whether an unconfigured ERPNext is a real option.

So installing ERPNext without configuring it is plausible. The site gains 529 doctypes it does not use and workspaces it did not ask for, but it never faces a setup wizard or a chart of accounts.

### What that leaves the site

Two choices, and both belong to the site owner rather than to this plan.

| | What it means |
|---|---|
| Install ERPNext and take v17 | the site stops being standalone. The section above is the argument that this is cheaper than it sounds |
| Stay on the last 1.x | security fixes only, no v17 features |

The third option, moving the shared masters somewhere smaller than ERPNext to preserve standalone, is rejected. It was the only path that kept standalone alive and it reopens the entire plan.

What this obliges the release notes to do is say it plainly and early, rather than letting a standalone site discover it mid-upgrade.

### Prior art worth knowing

ERPNext already reaches into Helpdesk on install. `erpnext/setup/install.py:414`:

```python
def create_helpdesk_fields():
    if "helpdesk" not in frappe.get_installed_apps():
        return
    from helpdesk.helpdesk.integrations.erpnext.customer import create_helpdesk_fields_in_customer
    create_helpdesk_fields_in_customer()
```

So the pattern of ERPNext carrying Helpdesk's fields on `Customer` is already established, which is useful for the `domain` and `country` contingency in [02-shared-masters/schema-changes.md](02-shared-masters/schema-changes.md).

Note the import path. The module in Helpdesk is `helpdesk.integrations.erpnext.customer`, with one `helpdesk`, not two. Either that line is broken or it targets a layout that has not landed. Check with whoever wrote it before relying on it.

## Verification

Run on a copy of a real site, not a fresh one, and run the whole bench upgrade rather than Helpdesk's patches alone. The interesting cases only exist on sites with history, and the interesting failures come from the order things run in.

```
A site that never had ERPNext              every HD Customer becomes a Customer, no duplicates
A site with the integration on             every ticket repoints through erpnext_customer, nothing created
A site that had both, integration off      exact names link, the rest are created and logged
A site with the same name on both sides    linked, not suffixed to "Netflix - 1"
A site with rows predating the toggle      they fall to tier 2 and 3, not silently skipped
A site with Helpdesk before ERPNext        before_migrate refuses, with the right message
A site with custom fields on HD Customer   they survive on the deprecated doctype, migrator offers them
A linked Customer with country already set the patch leaves it alone, migrator shows the conflict
A linked Customer with country empty       the patch fills it
A site with an SLA and no Issue            the clock still runs after step 5
```

The last one is the regression that would otherwise ship silently.

## Risks

The install order problem is the one that will actually happen. Every site that adopted Helpdesk before ERPNext has it, and that is a common way to arrive at both.

Patch runtime on a large site. Each tier 3 row runs ERPNext's insert controller, which does real work. Tier 1 and tier 2 skip the insert, so the cost scales with how much of the list is genuinely new rather than with the list. Worth measuring before the patch ships. See [02-shared-masters/customer.md](02-shared-masters/customer.md).

Name collisions during the patch, for the two masters with no tier 2. A `Holiday List` and an `HD Service Holiday List` with the same name, an agreement with the same name on both sides. The patch has to suffix rather than fail, and leave it for the migrator.

The clean break lands without warning for anyone who does not read release notes. This is accepted, but it means the release notes have to lead with it rather than list it.

## Open questions

None. The standalone question is settled in [01-decisions.md](01-decisions.md).

---

[← Migrator](04-migrator.md) · [App integration →](06-app-integration.md)
