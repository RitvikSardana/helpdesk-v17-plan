# Field mapping

[Index](../README.md)

Field by field, for the three masters that move. Generated from the doctype JSON, not from memory.

`SAME` means a field with that exact name exists on the ERPNext side. It does not mean the values are compatible, only that there is somewhere obvious for them to go.

## HD Customer into Customer

```
autoname   field:customer_name        ->   naming_series:
```

| Helpdesk field | Type | ERPNext side |
|---|---|---|
| `customer_name` | Data | same |
| `customer_type` | Select | same |
| `image` | Attach Image | same, hidden on `Customer` today and unhidden in v17 |
| `email_id` | Data | same |
| `mobile_no` | Data | same |
| `domain` | Data | **nothing**. Core field or Custom Field, see [../02-shared-masters/schema-changes.md](../02-shared-masters/schema-changes.md) |
| `country` | Link | **nothing**. Same contingency |
| `contacts` | Table (`HD Customer Member`) | `Dynamic Link`, or stays a child table. The Critical open question |
| `primary_contact` | Link (`Contact`) | **nothing**. Falls out of the contacts answer |
| `erpnext_customer` | Data | the link field itself. Survives the upgrade, see [../06-app-integration.md](../06-app-integration.md) |

The naming line matters more than it looks. `HD Customer` is named by `customer_name` and so is `Customer` on a default site, because `Selling Settings.cust_master_name` defaults to "Customer Name". A site that set it to `Naming Series` instead gets `CUST-2026-00001` and every ticket's customer value becomes an opaque code. See [../05-upgrade.md](../05-upgrade.md).

## HD Service Holiday List into Holiday List

```
autoname   field:holiday_list_name    ->   field:holiday_list_name
```

Nine of eleven fields match by name, including the child table shape.

| Helpdesk field | Type | ERPNext side |
|---|---|---|
| `holiday_list_name` | Data, reqd | same |
| `from_date` | Date, reqd | same |
| `to_date` | Date, reqd | same |
| `total_holidays` | Int | same |
| `weekly_off` | Select | same |
| `get_weekly_off_dates` | Button | same |
| `holidays` | Table (`HD Holiday`) | same, `Holiday` |
| `clear_table` | Button | same |
| `color` | Color | same |
| `description` | Data | **nothing**. ERPNext gains it |
| `recurring_holidays` | JSON | **nothing**. ERPNext gains it |

This is the cheapest of the three. Same autoname, same shape, two fields to add.

## HD Service Level Agreement into Service Level Agreement

```
autoname   field:service_level        ->   format:SLA-{document_type}-{service_level}
```

Both sides have sixteen fields and they are not the same sixteen.

| Helpdesk field | Type | ERPNext side |
|---|---|---|
| `service_level` | Data, reqd | same |
| `enabled` | Check | same |
| `start_date` | Date | same |
| `end_date` | Date | same |
| `holiday_list` | Link, reqd | same, retargeted to `Holiday List` |
| `support_and_resolution` | Table, reqd | same, `Service Day` |
| `priorities` | Table, reqd | same, `Service Level Priority` |
| `default_priority` | Link (`HD Ticket Priority`) | same name, becomes a **Dynamic Link** on `priority_doctype` |
| `apply_sla_for_resolution` | Check | same |
| `condition` | Code | same |
| `condition_json` | Code | **nothing**. ERPNext gains `filters` and `filters_editor` |
| `default_sla` | Check | **nothing** |
| `rank` | Int | **nothing**. ERPNext gains it |
| `description` | Data | **nothing**. ERPNext gains it |
| `default_ticket_status` | Link (`HD Ticket Status`) | **nothing**. ERPNext gains `default_status` |
| `ticket_reopen_status` | Link (`HD Ticket Status`) | **nothing**. ERPNext gains `reopen_status` |

The autoname difference is the interesting one. ERPNext names an agreement by its document type, Helpdesk names it by the service level alone. Two agreements called Standard, one for `HD Ticket` and one for `CRM Deal`, coexist on the ERPNext scheme and collide on Helpdesk's. That is an argument for the ERPNext scheme, not against it.

`default_sla` has no ERPNext counterpart and needs one, since something has to decide which agreement applies when no condition matches.

## What the migrator does with the gaps

Every **nothing** above that ERPNext is not gaining as a core field is step 2 of the migrator: map it, create a Custom Field for it, or drop it and say so. See [../04-migrator.md](../04-migrator.md).

---

[← Index](../README.md)
