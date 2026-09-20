# Doctype inventory

[Index](../README.md)

Every Helpdesk doctype and what v17 does to it. 36 on disk at the time of writing.

## Deprecated

Read only, no create button, badge on the form, rows stay readable, API still answers. Nothing deleted, no rows migrated out. See [../01-decisions.md](../01-decisions.md).

| Doctype | Child of | Replaced by |
|---|---|---|
| `HD Customer` | | `Customer` |
| `HD Customer Member` | `HD Customer` | depends on the contacts question |
| `HD Service Holiday List` | | `Holiday List` |
| `HD Holiday` | `HD Service Holiday List` | `Holiday` |
| `HD Service Level Agreement` | | `Service Level Agreement` |
| `HD Service Day` | `HD Service Level Agreement` | `Service Day` |
| `HD Service Level Priority` | `HD Service Level Agreement` | `Service Level Priority` |

The four child tables are field for field identical to ERPNext's and the instinct is to delete them. They stay, because a deprecated parent still has to render its rows.

## Removed

| Doctype | Why |
|---|---|
| `ERPNext HD Settings` | the integration it gates has nothing left to sync. See [../06-app-integration.md](../06-app-integration.md) |

## Unchanged

Everything else. Grouped by what it does rather than listed flat, because the list is long and the point is that most of Helpdesk is untouched.

```
Ticket          HD Ticket, HD Ticket Status, HD Ticket Priority, HD Ticket Type,
                HD Ticket Feedback Option, HD Ticket Template,
                HD Ticket Template Field
People          HD Agent, HD Agent Status, HD Team, HD Team Member
Knowledge       HD Article, HD Article Category, HD Article Feedback
Replies         HD Saved Reply, HD Saved Reply Team
Search          HD Stopword, HD Synonyms, HD Synonym
UI              HD View, HD Field Layout, HD Form Script
Settings        HD Settings
Other           HD Email Feedback, HD Comment Reaction
```

`HD Ticket Priority` and `HD Ticket Type` were in the original commonification plan and dropped out. Priority no longer has to move because `Service Level Agreement.default_priority` becomes a Dynamic Link, so ERPNext's engine points at Helpdesk's priorities without owning them. See [../02-shared-masters/sla.md](../02-shared-masters/sla.md).

## Already replaced, still on disk

`HD Ticket Comment`, `HD Ticket Activity` and `HD Notification` are not part of v17. Their migration to core `Comment`, the activity timeline and core notifications is finished, and the doctypes remain only because nothing deletes deprecated doctypes yet.

They are listed here so nobody counts them as v17 work. See [../08-future-scope.md](../08-future-scope.md).

## Field counts, for scale

The three masters that move, against what they move into:

```
HD Customer                   10 fields    Customer                  48
HD Service Holiday List       11 fields    Holiday List              13
HD Service Level Agreement    16 fields    Service Level Agreement   16
```

`Customer` being nearly five times wider than `HD Customer` is the whole of the field mapping problem and the reason the financial fields question came up at all. See [field-mapping.md](field-mapping.md).

---

[← Index](../README.md)
