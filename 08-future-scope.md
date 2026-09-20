# Future scope

[Index](README.md) · previous: [07-order-of-work.md](07-order-of-work.md) · next: [09-open-questions.md](09-open-questions.md)

## What this is

Deferred on purpose. Each one says what it is, why it is not in v17, and what would bring it back.

This is not the open questions list. Nothing here is undecided. The decision was to not do it now. See [09-open-questions.md](09-open-questions.md) for what is actually unresolved.

## Doctype renames

Dropping the `HD` prefix. `HD Ticket` becomes `Ticket`, `HD Agent` becomes `Agent`.

**Why not now.** v17 already breaks `HD Ticket.customer` and `HD Ticket.sla` for every integration. Renaming the doctype on top of that turns one breaking change people can read about into a rewrite. And the rename is pure cosmetics: nothing in the plan needs it.

**Trigger.** A release with no other API break in it. The two changes should never ship together.

## A shared shell and app switcher

One interface across Helpdesk, CRM and ERPNext, rather than four sites that link to each other.

**Why not now.** [01-decisions.md](01-decisions.md) pins it: islands, not a shared shell. v17 makes the records shared, not the interface. Deep linking out to another app's form is enough to prove the data is shared, and it is the version that ships in four months rather than twelve.

**Trigger.** The records actually being shared, which is what v17 delivers. This is the natural next release, not a someday item.

## Shared form builder and view settings

Layouts, saved views and field visibility defined once and honoured by every app.

**Why not now.** Each app has its own view machinery and its own settings doctypes. Unifying them is a framework project, not a Helpdesk one, and nothing in v17 depends on it.

**Trigger.** Somebody owning it at the framework level.

## Warranty Claim moves into Helpdesk

Today it stays in ERPNext and appears in Helpdesk as a read only tab on the customer view.

**Why not now.** `Warranty Claim` tracks a claim against an item and a serial number, which nothing in Helpdesk does. Moving it means Helpdesk grows item and serial number concepts, and that is a product direction rather than a migration. See [03-support-module-cleanup.md](03-support-module-cleanup.md).

**Trigger.** Item and serial number on a ticket, which is itself deferred with a trigger in [09-open-questions.md](09-open-questions.md). One follows the other.

## Helpdesk and CRM flows

A deal that raises a ticket, a ticket that shows its customer's open deals, a shared timeline.

**Why not now.** [00-overview.md](00-overview.md) draws the line at the shared customer key. The two apps have zero references to each other today, no doctype link, no import, no hook, so every flow is new work rather than a migration. And the flows only make sense once both apps are actually on `Customer`, which CRM does on its own schedule.

**Trigger.** CRM finishing its own move to `Customer` and `Service Level Agreement`. Until then a flow would be built against a record one side does not use yet. See [06-app-integration.md](06-app-integration.md).

## The migrator's undo button

Reversing a single merge from the log.

**Why not now.** The capture is not deferrable and the button is. The snapshot and the reference list exist only at merge time, so [04-migrator.md](04-migrator.md) records them from day one. The UI over that data can wait for someone to ask.

**Trigger.** The first person who merges the wrong two customers.

## The help portal

`/support`, `/help` and `/search_help` survive v17 untouched, along with `Support Settings` and `Support Search Source`.

**Why not now.** They are orphaned rather than broken. Nothing anywhere links to those three routes, and `Support Settings` has one field, `show_latest_forum_posts`, with no reader at all. Removing them is a separate cleanup with its own announcement, and bundling it into a release that is already deprecating `Issue` would blur what is being deprecated. See [03-support-module-cleanup.md](03-support-module-cleanup.md).

**Trigger.** Helpdesk shipping a public portal. That is the same product as `/support` and `/help`, so the day Helpdesk has an unauthenticated help site with search, ERPNext's three routes are deprecated along with `Support Settings` and `Support Search Source`.

Not before that, because until Helpdesk has one, deprecating ERPNext's would remove the only one that exists, unreachable or not.

The gap is smaller than it sounds. Helpdesk's knowledge base is already public: `HD Article Category` carries `allow_guest_to_view` and the knowledge base API allows guests. What is missing is the portal shell around it and the search, not the content.

## Knowledge base consolidation

There are two knowledge bases in the bench and they do not know about each other. Helpdesk has `HD Article` and `HD Article Category`, with guest viewing and a public API. Frappe has `Help Article` and `Help Category`, which is what the `/help` portal renders.

**Why not now.** Nothing in v17 touches either one, and neither blocks the customer, holiday list or SLA work. Consolidating them is its own migration with its own content decisions.

**Trigger.** The help portal decision above. They are the same question asked twice.

## Canned responses and Email Template

Helpdesk has zero references to `Email Template`. It uses `HD Canned Response` and `HD Ticket Template` instead.

**Why not now.** It is a duplicate, but it is a cheap one: no data is shared with another app through it, so leaving it costs nothing that the customer and SLA duplication costs. v17's argument is about records two apps both own, and nothing else owns a canned response.

**Trigger.** Another app wanting to send from the same template set.

## The last three replaced doctypes

`HD Ticket Comment`, `HD Ticket Activity` and `HD Notification` still exist on disk. Their migration to core is finished. See [appendix/doctype-inventory.md](appendix/doctype-inventory.md).

**Why not now.** [01-decisions.md](01-decisions.md) pins deprecate, never delete, and v17 is not the release that starts making exceptions.

**Trigger.** A release that deletes deprecated doctypes, which does not exist yet and would need its own policy.

## What is not here

Two things read like future scope and are not.

**Quality module consolidation.** Rejected rather than deferred. See [03-support-module-cleanup.md](03-support-module-cleanup.md).

**Converting `Issue` rows into `HD Ticket` rows.** Never, in any release. Deprecated rows stay where they are and stay readable. See [01-decisions.md](01-decisions.md).

---

[← Order of work](07-order-of-work.md) · [Open questions →](09-open-questions.md)
