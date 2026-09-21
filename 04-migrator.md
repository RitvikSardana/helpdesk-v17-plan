# Migrator

[Index](README.md) · previous: [03-support-module-cleanup.md](03-support-module-cleanup.md) · next: [05-upgrade.md](05-upgrade.md)

## What changes

A new app with a UI that cleans up after the upgrade.

The upgrade patches never score and never merge. On a site that ran Helpdesk and ERPNext without the integration between them, they leave `Customer` holding the real Acme Corporation that was always there and a new Acme Corp that came out of `HD Customer`. The migrator is how a human resolves that.

On a site that had the ERPNext integration enabled, there is nothing to **merge**. The patch used the links the sync had already written. Same where both lists held the company under exactly the same name, because the patch linked rather than creating. See [Three shapes of site](#three-shapes-of-site).

There may still be something to **map**. Linking moves no data, so a custom field on `HD Customer` has no home on the `Customer` it now points at. That is [step 2](#step-2-is-bigger-than-it-sounds), and it applies to a site with zero merges.

It ships as part of v17, so the flows can be tested together. See [07-order-of-work.md](07-order-of-work.md).

## The plan in short

Plain version of the whole thing. Everything below is the detail.

### What the upgrade patch does

It looks at each HD Customer and does one of three things.

1. The row is already linked to a Customer. Use that link. Nothing new. The link comes from the ERPNext integration, which pairs rows as they are created and has a Sync Customers button that pairs everything older.
2. A Customer with the exact same name exists. Point at it. Nothing new.
3. Neither. Make a Customer, and put the row on the migrator's list.

That is all. The patch does not compare names, does not merge anything, and never asks a question.

### What the migrator app does

Two jobs. They are separate and they do not move together.

**Job one, the duplicates.** Only the rows from case 3. The patch made an Acme Corp when ERPNext already had an Acme Corporation. The app shows both side by side, says it thinks they are the same, and you say yes or no. Nothing merges on its own.

**Job two, the fields.** The patch moved no custom field data anywhere. So for each field on HD Customer you choose once: send it to a field that already exists on Customer, make a new field on Customer for it, or drop it. Then it copies for every customer on the site.

Job two applies to all three cases, not just case 3. Even a fully synced site still has its field data sitting on HD Customer, because the sync only ever copied two fields, image and customer_type.

### When there is nothing to do

Integration was on, everything synced, no custom fields on HD Customer. Then both jobs are empty. The app should say so instead of showing six steps with nothing in them.

## Why a separate app

Three reasons, in order of weight.

It is not Helpdesk's problem alone. CRM has the same job, folding `CRM Organization` into `Customer`. A tool in Helpdesk cannot be used by CRM without Helpdesk being installed for no reason.

It is temporary in a way Helpdesk is not. Most sites run it once and never open it again. That work does not belong in the product's sidebar forever.

It keeps the merge log out of Helpdesk. The records being merged belong to ERPNext, the tool belongs to nobody in particular, and the log is the tool's.

## What it migrates

Masters, not documents. Three pairs on the Helpdesk side:

| From | To | Section |
|---|---|---|
| `HD Customer` | `Customer` | [02-shared-masters/customer.md](02-shared-masters/customer.md) |
| `HD Service Holiday List` | `Holiday List` | [02-shared-masters/holiday-list.md](02-shared-masters/holiday-list.md) |
| `HD Service Level Agreement` | `Service Level Agreement` | [02-shared-masters/sla.md](02-shared-masters/sla.md) |

Same screen for all three, because it is the same problem: two doctypes, overlapping rows, a human deciding which are the same thing.

CRM brings two more of its own, on its own schedule:

| From | To |
|---|---|
| `CRM Organization` | `Customer` |
| `CRM Service Level Agreement` | `Service Level Agreement` |

So the app ships five default pairs, not three, and an **Add a pair** escape for a doctype it has no default for.

No `HD Ticket` is touched. No `Issue` is touched. Documents keep their links, and their links already point at the right place after the upgrade patch.

## What the v17 patch already did

The migrator opens on a site the upgrade has already touched, so it is worth stating plainly what is left.

The patch had one job: make every `HD Ticket.customer` value resolve against `Customer` instead of `HD Customer`. Frappe validates every Link field that holds a value on every save, `get_invalid_links` in `frappe/model/base_document.py`, with no change detection. So a ticket naming a company with no `Customer` behind it would fail the next time an agent replied to it. That is the only reason the patch writes anything at all.

Per `HD Customer`, in order:

| Signal | Patch did | Leaves behind |
|---|---|---|
| `erpnext_customer` is set | linked to it | nothing to merge |
| `customer_name` matches a `Customer` exactly | linked to it | nothing to merge |
| neither | created a `Customer`, logged it | one row on the merge queue |

In no tier did it score a name, merge two records, move custom field data or ask a question. See [05-upgrade.md](05-upgrade.md).

So two independent things land here.

**The merge queue.** The third row only. Acme Corp sitting next to the Acme Corporation that was always there.

**The field data.** Every row, every tier. The patch copied standard fields onto records it created and nothing onto records it linked to, so everything living only on `HD Customer` is still living only on `HD Customer`. Nothing is lost, because the doctype is deprecated rather than deleted and the rows stay readable.

Those two do not move together. A site can have an empty merge queue and a full step 2, and the integration enabled site is exactly that shape: the sync wrote the links, but `FIELDS_TO_SYNC` in `helpdesk/integrations/erpnext/utils.py` is `image` and `customer_type` and nothing else, so every other value never crossed.

## Three shapes of site

The tool has to recognise which one it is looking at, because the work is different.

**Both apps, ERPNext integration enabled.** `ERPNext HD Settings.enabled` turns on a two way sync. `HDCustomer.after_insert` creates the `Customer`, `Customer.after_insert` creates the `HD Customer`, and `set_links` writes `erpnext_customer` and `hd_customer` on both sides. Every row created since the toggle is already paired and verified, and the upgrade patch used that mapping. **Nothing to merge here.** If `HD Customer` also carries no custom fields, the tool should say it has nothing to do rather than showing an empty stepper.

The toggle alone pairs nothing retroactively, but **Sync Customers** does. `sync_all_customers` in `helpdesk/integrations/erpnext/api.py:59` links every unlinked row with an exact name match on the other side and creates the counterpart for the rest, both directions, and `in_sync()` gates the button. So the question is not whether the toggle was on, it is whether anyone pressed it. A site that was never synced has unlinked rows that behave exactly like the case below.

**Both apps, integration never enabled.** Two customer lists that grew separately, no links between them. This is the real case and the reason the tool exists.

**Helpdesk alone.** Every `HD Customer` became a `Customer` during the upgrade. Nothing to match, nothing to merge.

The migrator does not need a backfill screen of its own. Helpdesk already ships one in the integration settings, it works on v16 today, and rebuilding it here would be a second implementation of the same rule.

## Matching

There is prior art in the same bench. ERPNext's bank transaction matcher does exactly this shape of job, and the migrator should follow it rather than inventing a scheme.

`erpnext/accounts/doctype/bank_transaction/auto_match_party.py`:

```
1. Exact identifier first        match on IBAN or account number
2. Fuzzy only if that fails      and only if the site enabled fuzzy matching
3. rapidfuzz, token_set_ratio    with default_process for normalisation
4. Cutoff at 80
5. Ambiguity means no match      if the top two score the same, return nothing
```

Every one of those carries over.

**Exact first.** `erpnext_customer` is the migrator's IBAN. Where it is set, that is the answer and nothing else runs. On an integration enabled site that covers most of the list, which is why fuzzy matching is the fallback here rather than the main path.

**`token_set_ratio` is the right scorer** for this data. It compares the set of tokens rather than the string, so Netflix against Netflix Inc Ltd scores high, which a plain ratio would not. That is the exact case in the brief.

**The ambiguity rule matters more here than in banking.** Two customers called Acme in ERPNext and one in Helpdesk means the tool must say it does not know, not pick the first. A wrong merge is far more expensive than an unresolved row.

Other signals to score alongside the name, cheap and worth having:

```
domain        exact match on the email domain is strong evidence
email_id      exact match is strong evidence
mobile_no     exact match is strong evidence
```

A name match plus a domain match is a different confidence from a name match alone, and the UI should say which one it is looking at.

`rapidfuzz` is already an ERPNext dependency, `rapidfuzz~=3.14.3`. ERPNext is always installed. So this costs nothing to add.

## Nothing merges by itself

One rule, and it is the safety property of the whole tool.

**An existing `erpnext_customer` mapping applies automatically. Everything found by a heuristic waits for a human.**

Fully automatic above a confidence score is faster, and it merges two companies that happened to look alike. Unmerging is painful and sometimes impossible. A site with two thousand customers has someone clicking through the ones the heuristic found, which is slow, and slow is the right trade here.

The UI's job is to make that clicking fast: good defaults, high confidence rows grouped together, bulk accept for a whole confidence band.

## Where the queue comes from

The migrator does not open on the whole `Customer` table. It opens on what the upgrade patch created.

The patch cannot write the log itself, because the log is this app's and this app may not be installed when the patch runs. See [01-decisions.md](01-decisions.md). So the patch marks its own side, on the already deprecated `HD Customer`, and the migrator turns those marks into log rows and a queue on first run. Rows the patch linked through `erpnext_customer` or an exact name match are not marked and never enter the queue. See [05-upgrade.md](05-upgrade.md).

So a row in step 3 reads:

```
Acme Corp          created by the upgrade
  likely match:    Acme Corporation      score 91
```

Three outcomes: merge into the suggested target, keep it as a separate company, or pick a different target. Keeping it is a decision rather than a skip, and it takes the row off the queue for good.

When the queue is empty the site has converged on the integration enabled shape, and the banner disappears.

The queue governs steps 3 and 4 only. Step 2 is a decision about the pair, not about a row, so it runs whether the queue holds two thousand rows or none.

## It is a merge tool, not only a deduplicator

The upgrade is the largest producer of work for it, not the only one.

| Job | Duplicates |
|---|---|
| Resolve the `Customer` records the patch created | yes, the queue above |
| Move custom field data off a linked `HD Customer` onto its `Customer` | no, the patch cannot do this at all |
| Settle a standard field where both sides hold a different value | no, the patch never touches a record it linked to |
| Decide which side's field values survive a merge | no |
| Holiday lists and agreements | yes, and more of them, because neither has a link field to read |
| `CRM Organization` into `Customer` | yes, same shape, CRM's schedule |
| Backfill a v16 site that enabled the integration late | not this tool. Sync Customers already does it |

## Choosing a pair

The first screen is not step one. It is a list of the pairs the app knows about, each with its row count and where it got to.

```
From                          Into                      App        Rows   Status
HD Customer                   Customer                  Helpdesk   1,284  504 left
HD Service Holiday List       Holiday List              Helpdesk       4  not started
HD Service Level Agreement    Service Level Agreement   Helpdesk       6  not started
CRM Organization              Customer                  CRM          842  not started
CRM Service Level Agreement   Service Level Agreement   CRM            0  nothing to do
```

A pair with no rows on either side says so and offers no button. A pair part way through resumes at the step it was left on, because a site with two thousand customers will not finish in one sitting.

Two pairs land in the same target. `HD Customer` and `CRM Organization` both become `Customer`. That is exactly why the merge log records the app each merged record came from. See [The merge log](#the-merge-log).

## The six steps

Mockup: [Migrator Stepper](https://claude.ai/artifact/V5UjbEyJVKHs2VyNUrHoyE). The pair chooser, six steps, and the sidebar banner.

They run once per pair, in any order.

```
1  Scan            what this site looks like, how many rows, what was found
2  Map fields      standard and custom, with defaults prefilled
3  Review matches   what the heuristics found, confirm or reject each
4  Unmatched       what will be created new, with the chance to match by hand
5  Confirm         the full summary before anything is written
6  Result          what happened, with the merge log
```

Nothing is written before step 5. Steps 1 to 4 are read only, so a user can walk the whole tool, see what it would do, and close it.

The same six steps serve every pair. Run the stepper once per pair, picked from the screen above.

## Step 2 is bigger than it sounds

It is not only custom fields. `HD Customer.country` is a standard Helpdesk field with no `Customer` equivalent, so it has the same problem a custom field has.

This step is why the tool still has work on a site with nothing to merge. The upgrade patch links a ticket to an existing `Customer` without moving any data, so every value that lives only on `HD Customer` is still sitting there afterwards. See [05-upgrade.md](05-upgrade.md).

Five cases, all of which have to be handled:

| Case | What the UI offers |
|---|---|
| Helpdesk field, ERPNext field with the same meaning, different name | map it |
| Both sides hold a value and they differ | conflict, show both, a human picks |
| Helpdesk field, nothing on the ERPNext side | create a Custom Field on `Customer` and copy the values |
| Helpdesk field nobody wants to keep | skip it, and say plainly the data will not come across |
| Same fieldname on both sides, different types | conflict, a human decides, no default |

Defaults prefilled by name and type match, so a typical site clicks through this step without thinking about it.

The one that needs care is skipping. It is a data loss and the wording has to say so rather than calling it a skip.

### The conflict screen

The tool does not guess which side is right. It finds the conflicts and asks.

It is already reading both sides to copy them, so it knows exactly which rows differ and on which field. That becomes a screen: one section per field in conflict, the two values side by side, and the rows they came from. The user picks a side for the field and applies it to every row at once, or opens the list and picks per row where the field matters enough.

Nothing is written until step 5, so a user can look at every conflict and walk away. A site with no conflicts never sees the screen at all.

Settled in [01-decisions.md](01-decisions.md).

## Merging

Frappe already has the mechanism: `rename_doc` with `merge=True`. It repoints every reference to the surviving name and deletes the other.

So the migrator does not write its own reference walker. It collects the decisions and calls the framework.

What it has to do itself is capture what happened before calling it, because `rename_doc` does not.

## The merge log

One generic doctype in the migrator app, not one per doctype pair. It covers customers, holiday lists, agreements, and whatever the tool grows later.

Each row records:

```
merged from        doctype and name
merged into        doctype and name
snapshot           the full record that was deleted
references         every document that was repointed
who, when
app                which app the merged record came from
```

**Capture the snapshot and the reference list from day one.** They exist only at merge time. Everything else can be added later; these cannot be recovered.

### Undo

Possible, and honest about its limits.

Undo recreates the deleted record from the snapshot and points the listed documents back. It works well shortly after a mistake and degrades the longer you wait:

- A ticket moved to a third customer after the merge must be skipped, not blindly repointed.
- The old name appearing inside a comment or a text field was never a reference and cannot be recovered.
- The original name has to still be free.

So the undo button is UI over data that is already being captured. Ship without it, add it when someone asks. That is the split: the capture is not deferrable, the button is.

## The banner

Helpdesk shows a banner in the bottom of the left sidebar, the same place and shape as the customer portal permission banner it already has. Last board in the [mockup](https://claude.ai/artifact/V5UjbEyJVKHs2VyNUrHoyE). It says there are unmerged customers and links into the migrator.

It is a nudge, not a gate. Dismissible, and it disappears when there is nothing left to merge.

CRM can show its own when it gets there.

## Existing data

The migrator creates nothing on install and touches nothing until someone runs it.

It has to be resumable. A site with two thousand customers will not finish in one sitting, so the decisions from steps 2 to 4 persist between visits and the stepper reopens where it was left.

Running it twice is safe. The second run sees the merges already done and has less to offer.

## The ERPNext integration goes away

Worth saying here rather than only in [03-support-module-cleanup.md](03-support-module-cleanup.md), because this tool is the last thing that needs it.

The integration exists for one reason: to keep two customer masters in step. v17 has one. So there is nothing left for it to do, and it is deleted rather than left running against a deprecated doctype.

What goes:

```
helpdesk/integrations/erpnext/customer.py          Customer and HD Customer sync
helpdesk/integrations/erpnext/user_permission.py   User Permission mirror
helpdesk/integrations/erpnext/doc_share.py         DocShare mirror
helpdesk/integrations/erpnext/mirror_sync.py       the shared mirror engine
helpdesk/integrations/erpnext/api.py               get_sync_info, Sync Customers
ERPNext HD Settings                                the toggle itself
hooks.py                                           the Customer, User Permission
                                                   and DocShare doc_events
```

The two mirrors need care. They exist so a User Permission or a DocShare granted against one customer master also applies to the other. With one master there is nothing to mirror, but the rows still have to end up pointing at `Customer` rather than at the deprecated `HD Customer`, and that is a migration rather than a deletion. See [02-shared-masters/permissions.md](02-shared-masters/permissions.md).

What stays:

```
HD Customer.erpnext_customer    tier 1 reads it, and so does this tool
Customer.hd_customer            the reverse link, same reason
```

Stopping the sync and dropping the link fields are two different jobs on two different schedules. The sync stops at the upgrade, because by then it has nothing to sync. The fields are the migrator's map back to where a row came from, so they outlive the tool's first run and only become dead weight once every pair on the site is resolved.

## Risks

A wrong merge is the main one, and it is why nothing is automatic. The ambiguity rule and the human confirmation are both there for this.

Fuzzy matching at scale is slow. The banking matcher loads every party name into memory and scores against all of them. Two thousand by two thousand is fine, fifty thousand by fifty thousand is not. The queue helps, since only the patch's own creations are scored rather than the whole table, but score in the background and page the results anyway.

The tool is temporary but the merge log is not. Uninstalling the migrator app should not take the log with it, and that needs thinking about before it ships.

Step 2 creating Custom Fields on `Customer` overlaps with the upgrade patch doing the same thing for `domain` and `country`. Both have to check before creating. See [02-shared-masters/schema-changes.md](02-shared-masters/schema-changes.md).

## Open questions

Tracked in [09-open-questions.md](09-open-questions.md).

- How the merge log outlives an uninstall. Who owns it is settled in [01-decisions.md](01-decisions.md).

---

[← Support module cleanup](03-support-module-cleanup.md) · [Upgrade →](05-upgrade.md)
