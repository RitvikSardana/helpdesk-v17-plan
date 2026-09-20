# Reference counts

[Index](../README.md)

How much code actually touches the doctypes v17 deprecates. Counted on the branch at the time of writing, not estimated.

Counts are occurrences of the doctype name. They overstate slightly, because a string appears in tests and in comments as well as in live code.

## Helpdesk

| Doctype | Python files | Python refs | Frontend files | Frontend refs |
|---|---|---|---|---|
| `HD Customer` | 33 | 303 | 11 | 20 |
| `HD Service Level Agreement` | 14 | 29 | 4 | 5 |
| `HD Service Holiday List` | 6 | 14 | 5 | 8 |
| `HD Service Level Priority` | 5 | 11 | 1 | 1 |
| `HD Customer Member` | 6 | 7 | 1 | 1 |
| `HD Holiday` | 2 | 5 | 1 | 1 |
| `HD Service Day` | 3 | 3 | 1 | 1 |

Excluding tests, `HD Customer` is 25 Python files and `HD Service Level Agreement` is 10.

`HD Customer` is an order of magnitude ahead of everything else, which is why [../02-shared-masters/customer.md](../02-shared-masters/customer.md) is the long section and the other two are short.

## The schema blast radius is three fields

The reference counts above are code. The schema is much smaller, and this is the number that decides what the upgrade patch has to do.

Only three link fields point into the deprecated family from outside it:

```
HD Ticket.customer              Link -> HD Customer
HD Ticket.sla                   Link -> HD Service Level Agreement
User Invitation.customer        Link -> HD Customer      (Custom Field)
```

Everything else is internal: a deprecated parent pointing at its own child table.

```
HD Customer.contacts                        Table -> HD Customer Member
HD Service Holiday List.holidays            Table -> HD Holiday
HD Service Level Agreement.holiday_list     Link  -> HD Service Holiday List
HD Service Level Agreement.support_and_resolution  Table -> HD Service Day
HD Service Level Agreement.priorities       Table -> HD Service Level Priority
```

### The third one is not in the patch order yet

`User Invitation.customer` is a Custom Field created by `helpdesk/patches/add_customer_field_in_user_invitation.py`, with `options: "HD Customer"`. [../05-upgrade.md](../05-upgrade.md) step 9 repoints `HD Ticket.customer` and `HD Ticket.sla` and says nothing about it.

It has to be repointed on the same terms, for the same reason: a Link field whose value does not exist fails validation on the next save. `HD Customer.unlink_invitations` already writes to this field, so it is live rather than vestigial.

## ERPNext

The Support module side, for contrast. Every reference to the three deprecated doctypes in all of ERPNext:

```
Issue            Task.issue
                 Issue.issue_split_from
Issue Type       Issue.issue_type
Issue Priority   Issue.priority
                 Service Level Agreement.default_priority
                 Service Level Priority.priority
```

One external link in the whole application, `Task.issue`. The rest is `Issue` pointing at itself or the SLA engine pointing at priorities, and the SLA ones are being converted to Dynamic Links anyway.

Deprecating `Issue` is cheap. Deprecating `HD Customer` is not. See [../03-support-module-cleanup.md](../03-support-module-cleanup.md).

## Custom fields Helpdesk adds to other apps

Relevant because the migrator's step 2 creates more of them, and because one of them is the third link field above.

```
Tag                 app, color
Assignment Rule     assign_condition_json, unassign_condition_json
User Invitation     contact, customer
Customer            hd_customer            (Data, created by the ERPNext integration)
Comment             see helpdesk/setup/comments.py
```

`Customer.hd_customer` is a `Data` field rather than a `Link`, so it holds a name without validating it. That is why it survives `HD Customer` being deprecated without any special handling.

---

[← Index](../README.md)
