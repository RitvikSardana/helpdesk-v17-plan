# Support module cleanup

[Index](README.md) · previous: [02-shared-masters](02-shared-masters/README.md) · next: [04-migrator.md](04-migrator.md)

## What changes

`Issue`, `Issue Type` and `Issue Priority` are deprecated. Helpdesk's ticket takes over.

The Support module does not go away. `Warranty Claim` stays, `Service Level Agreement` stays and grows, and the Maintenance and Quality modules are not touched at all.

## Why

A site running both apps has two ticketing systems. `Issue` and `HD Ticket` do the same job, and `HD Ticket` does it better: a real status master, a working SLA clock, an agent UI, a customer portal, canned responses, teams, assignment rules.

Keeping both means every support feature gets built twice and customers get asked which one to use.

## What is deprecated

Read only, no create button, badge on the form, rows stay readable, the API still answers. No rows are migrated out. See [01-decisions.md](01-decisions.md).

| Doctype | Module | Replaced by |
|---|---|---|
| `Issue` | Support | `HD Ticket` |
| `Issue Type` | Support | `HD Ticket Type` |
| `Issue Priority` | Support | `HD Ticket Priority` |
| `Pause SLA On Status` | Support | status category, see [02-shared-masters/sla.md](02-shared-masters/sla.md) |
| `SLA Fulfilled On Status` | Support | status category |

Deprecated alongside them, because their only subject is `Issue`. Every one already has a Helpdesk equivalent running on `HD Ticket`:

| Thing | Type | Helpdesk equivalent |
|---|---|---|
| `First Response Time for Issues` | report | `First Response Time for Tickets` |
| `Issue Analytics` | report | `Ticket Analytics` |
| `Issue Summary` | report | `Ticket Summary` |
| `Support Hour Distribution` | report | `Support Hour Distribution` |
| `issues` | web form | the Helpdesk customer portal |

Four for four, same names with tickets in place of issues, all Script Reports on `HD Ticket`. Helpdesk also has `Ticket-Search Analysis`, an agent dashboard and an analytics tab on the ticket list, none of which ERPNext has anything like.

This is the parity argument in concrete form. Nobody loses a report.

The ERPNext reports still run. They report on frozen data, which is the correct behaviour for a deprecated doctype.

## How little is actually linked

This is the part that makes the deprecation cheap. Across all of ERPNext, the link fields pointing at these three doctypes are:

```
Issue            Task.issue
                 Issue.issue_split_from
Issue Type       Issue.issue_type
Issue Priority   Issue.priority
                 Service Level Agreement.default_priority
                 Service Level Priority.priority
```

One external link in total: `Task.issue`, in Projects. Everything else is `Issue` pointing at itself, or the two SLA fields that become Dynamic Links anyway. See [02-shared-masters/schema-changes.md](02-shared-masters/schema-changes.md).

`Issue Type` and `Issue Priority` are referenced nowhere outside the Support module. That is why `HD Ticket Type` never had to move into ERPNext to replace `Issue Type`. There was nothing on the ERPNext side to keep working.

`Task.issue` keeps working. It points at deprecated rows, which stay readable.

## What stays

| Doctype | Module | Why |
|---|---|---|
| `Warranty Claim` | Support | tracks a claim against an item and a serial number. Nothing in Helpdesk does this |
| `Maintenance Visit` | Maintenance | untouched |
| `Maintenance Schedule` | Maintenance | untouched |
| `Installation Note` | Selling | untouched |
| `Service Level Agreement` | Support | becomes the one SLA engine. See [02-shared-masters/sla.md](02-shared-masters/sla.md) |
| `Support Settings` | Support | the `/support` help portal reads it. Its SLA fields leave. See below |
| `Support Search Source` | Support | untouched |

The first four appear in Helpdesk as read only tabs on the customer view, deep linking to the desk document. See [02-shared-masters/customer.md](02-shared-masters/customer.md).

## Quality and Maintenance are out of scope

Neither module is touched in v17.

**Maintenance** has no overlap with Helpdesk. Scheduled preventive visits against installed items are not tickets, and nothing in Helpdesk competes with them.

**Quality** was checked because `Quality Feedback` looked like it might overlap with ticket satisfaction ratings. It does not. It is a template driven survey with multiple weighted parameters, taken about a `User` or a `Customer`, not about a ticket. Different subject, different shape. Ruled out.

## What Support looks like afterwards

The module keeps a workspace and it is not empty.

Live: `Warranty Claim`, `Service Level Agreement`, `Support Settings`, `Support Search Source`, and the Maintenance links the workspace already carries.

Deprecated, still listed: `Issue`, `Issue Type`, `Issue Priority`.

The workspace needs editing so the Issue links are not the first thing on it.

One honest oddity: ERPNext ends up hosting the SLA engine and running nothing on it, because `Issue` is deprecated and `Warranty Claim` does not use SLA. Accepted. See [02-shared-masters/sla.md](02-shared-masters/sla.md).

## Support Settings keeps its doctype and loses its SLA fields

Worth checking properly, because the first answer was wrong.

`Support Settings` is not needed by the SLA engine, and keeping the dependency is a live upgrade hazard.

`track_service_level_agreement` is a site wide on or off switch for the whole engine, and **it defaults to 0**. Five places read it. Two are Issue specific:

```
validate_doc                              guarded by document_type == "Issue"
change_service_level_agreement_and_priority   guarded by frappe.db.exists("Issue", ...)
```

Three are generic and would gate Helpdesk the moment tickets move onto this engine:

```
get_active_service_level_agreement_for    returns early, no agreement is found
get_service_level_agreement_filters       returns early
the enabled check in validate_doc
```

`get_active_service_level_agreement_for` is the engine's entry point for any document type. So on a site that never enabled this flag, which is every site that never used `Issue`, tickets would upgrade onto an engine that silently finds no agreement. No error, no breach, no clock.

**So the switch goes with `Issue`.** Remove the three generic reads. Whether an agreement applies is already per agreement, through `enabled`, which is the right granularity. A site wide flag defaulting to off is legacy from when `Issue` was the only consumer, and it does not survive the engine becoming shared.

`allow_resetting_service_level_agreement` gates `reset_service_level_agreement`. It is not Issue specific in the code, so it either moves onto the agreement or goes. Build time call.

The doctype itself stays, for a reason that has nothing to do with `Issue`. The `/support` help portal reads it: `greeting_title`, `greeting_subtitle`, the forum fields and `search_apis`, through `erpnext/www/support/index.py`, `templates/pages/help.py` and `templates/pages/search_help.py`. That portal runs on `Help Article` and `Help Category`, which live in the Framework, not in Support. None of it is affected by this section.

`close_issue_after_days` becomes dead and is deprecated in place. Do not restructure the Single.

Every field, and what reads it after v17:

| Field | After v17 | Read by |
|---|---|---|
| `greeting_title` | live | `www/support/index.py` |
| `greeting_subtitle` | live | `www/support/index.py` |
| `get_started_sections` | live | `templates/pages/help.py` |
| `forum_url` | live | `templates/pages/help.py` |
| `get_latest_query` | live | `templates/pages/help.py` |
| `response_key_list` | live | `templates/pages/help.py` |
| `post_title_key` | live | `help.py`, `search_help.py` |
| `post_description_key` | live | `help.py`, `search_help.py` |
| `post_route_key` | live | `help.py`, `search_help.py` |
| `post_route_string` | live | `templates/pages/help.py` |
| `search_apis` | live | `templates/pages/search_help.py` |
| `close_issue_after_days` | dead | `Issue.auto_close_tickets` only |
| `track_service_level_agreement` | leaves | the SLA engine, see above |
| `allow_resetting_service_level_agreement` | leaves | `reset_service_level_agreement` |
| `show_latest_forum_posts` | already dead | nothing, today |

Eleven of sixteen fields are the help portal. One is Issue's. Two leave with the engine. One has no reader at all and already did not before v17.

### What the help portal is

Three pages, written before Helpdesk existed, doing what Helpdesk's knowledge base and portal do now.

**`/support`** is the landing page. Greeting, then the six most viewed `Help Article` records, found by joining against `Web Page View`, then every published article grouped by `Help Category`.

**`/help`** is an older variant of the same idea. Configurable link sections from `get_started_sections`, a JSON blob. The three latest posts fetched live over HTTP from whatever forum `forum_url` points at. The signed in user's three most recent issues. A search box.

**`/search_help`** is federated search, and it is the one interesting piece. Each `Support Search Source` row is either an external JSON API or a local doctype:

```
API      base_url + query_route, the search term parameter name,
         a key path to dig results out of the response,
         and keys for each result's title, description and route
Link     a doctype, searched through global search,
         with fields naming the title, preview and route
```

Results render as one section per source. So a customer searching once gets hits from the company's own articles, its Discourse forum, and any documentation API it has configured.

The use case is deflection. Land on the help centre, search once across everything, and only raise an issue if nothing answers. Standard help centre design, and the same job Helpdesk's knowledge base does.

One thing to catch: `help.py` calls `frappe.get_list("Issue", ...)` for the signed in user's last three issues. After the deprecation that list shows frozen rows. Leave it, the page is orphaned anyway.

### And nothing links to it

"Live" is generous. The three pages are served, but nothing routes to them.

| Route | File | Linked from |
|---|---|---|
| `/support` | `www/support/index.py` | nothing |
| `/help` | `templates/pages/help.py` | nothing |
| `/search_help` | `templates/pages/search_help.py` | the search box on `/help` |

Not in `standard_portal_menu_items`, not in the Support workspace, not linked from the Framework. Reachable only by typing the URL.

So `Support Settings` survives on the strength of a portal that has no way in. That is the state today, before v17, and v17 does not change it. Recorded because someone will ask why Support still has a settings page.

It also sharpens the future scope note below. ERPNext has a knowledge base at `/support` that nobody is routed to, and Helpdesk has one that customers actually use.

Noted, not scoped: that help portal overlaps with Helpdesk's own knowledge base. Two portals over the same Framework doctypes, one of them unreachable. Not part of v17.

The trigger for removing it is Helpdesk shipping a public portal. Once Helpdesk has an unauthenticated help site with search, these three routes, `Support Settings` and `Support Search Source` are the same product built twice and the ERPNext copy is deprecated. Until then it stays, because it is the only one that exists. See [08-future-scope.md](08-future-scope.md).

## The email flow

`Issue` has `email_append_to: 1`, so it appears in the `append_to` picker on `Email Account`. An account pointed at `Issue` turns inbound mail into issues.

`HD Ticket` has the same flag, so both appear in that picker today.

After the deprecation, `Issue` should stop appearing, and an existing account pointed at `Issue` has to go somewhere. Leaving it creates rows nobody reads.

Leaning: repoint it in a patch so mail creates an `HD Ticket`. Open, because the repoint rewrites a live site's mail configuration without being asked. Tracked in [09-open-questions.md](09-open-questions.md).

## The portal page

`/issues` is one of the nine portal routes gated on the ERPNext `Customer` role, in `erpnext/hooks.py:282`. It is backed by the `issues` web form.

After the deprecation it is a page listing frozen rows with no way to add one. Remove the menu item. The web form stays, deprecated, so an old bookmark does not 404.

This is also one of the arguments in [09-open-questions.md](09-open-questions.md) against merging Helpdesk's portal role into ERPNext's `Customer` role. Sharing the role would hand every helpdesk portal user a link to a dead page.

## The public commitment

There is a forum thread on this: [Support module to be removed in v17](https://discuss.frappe.io/t/support-module-to-be-removed-in-v17/162745).

The objection, from a manufacturer, is that tracking issues and warranty claims by customer, item code and serial number is essential, and that there is no equivalence between the two systems.

Two replies from the Frappe side set a bar: that the Support module would not be deprecated, and that until complete parity with the module is achieved, it would not be deprecated.

**The plan meets that bar, and it does so without a parity gate.**

The objection is about item and serial number tracking. `Issue` never had `item_code` or `serial_no`. Those fields are on `Warranty Claim`, and `Warranty Claim` is not being deprecated. It stays exactly as it is, and it becomes more visible than before, as a tab on the Helpdesk customer view.

So nothing that the objection describes is lost. What is deprecated is a ticketing doctype that has a strictly better replacement in the same bench.

What the plan does not do is move item and serial onto `HD Ticket`. That is deliberate and it reopens if agents need to raise a ticket against a serial directly. See [09-open-questions.md](09-open-questions.md).

The wording matters when this ships. "The Support module is removed" is false and will restart the thread. "`Issue` is replaced by the Helpdesk ticket, warranty and maintenance are untouched" is what actually happens.

## Existing data

Nothing moves.

1. Mark the three doctypes deprecated.
2. Set `Issue.email_append_to` to 0 so it leaves the picker.
3. Remove the `/issues` portal menu item.
4. Edit the Support workspace.

No `Issue` row is converted into an `HD Ticket`. No agreement is rewritten. Existing issues keep whatever their clock last computed, frozen.

The migrator does not touch issues either. It is for masters, not documents. See [04-migrator.md](04-migrator.md).

## Risks

The announcement is the risk, not the code. The thread above shows the module is read as one thing, and the difference between deprecating `Issue` and removing Support is not obvious to someone who does not read the doctype list.

`Task.issue` pointing at deprecated rows is correct but will look like a bug to someone who finds it.

A site whose inbound support mail lands on `Issue` silently stops being processed if the repoint is not handled. This is the sharp edge of the open question above.

## Open questions

Tracked in [09-open-questions.md](09-open-questions.md).

- What happens to an `Email Account` pointed at `Issue`. Leaning repoint.
- Item and serial number on `HD Ticket`. Deferred with a trigger.

---

[← Schema changes](02-shared-masters/schema-changes.md) · [Migrator →](04-migrator.md)
