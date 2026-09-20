# SLA

[Index](../README.md) · [Shared masters](README.md)

## What changes

One SLA engine, owned by ERPNext, used by Helpdesk and CRM.

`HD Service Level Agreement` folds into `Service Level Agreement` and is deprecated. `CRM Service Level Agreement` does the same, on CRM's schedule.

Helpdesk deletes its SLA screens and renders ERPNext's UI package instead.

19 references move on the Helpdesk side.

## Why

Three implementations of the same clock. ERPNext runs one on `Issue`, Helpdesk runs a better one on tickets, CRM runs a smaller one on deals and leads. Every fix has to be made three times.

ERPNext owns it because ERPNext is the base app and both consumers can link to it.

## Neither one wins outright

ERPNext's version is more generic than Helpdesk's. Helpdesk's is better at deciding which agreement applies.

**ERPNext has, Helpdesk does not:**

| Field | What it does |
|---|---|
| `document_type` | makes the agreement work on any doctype. Helpdesk is hardcoded to tickets |
| `pause_sla_on` | statuses that pause the clock |
| `sla_fulfilled_on` | statuses that stop it |
| `entity` / `entity_type` | per customer, customer group or territory agreements |

**Helpdesk has, ERPNext does not:**

| Field | What it does |
|---|---|
| `rank` | tie break when several agreements match |
| `condition_json` | the UI built condition. ERPNext only has the raw Python expression. Replaced, see below |
| `description` | |
| `ticket_reopen_status` | ticket specific, needs generalising |
| `default_ticket_status` | ticket specific, needs generalising |

So the merge takes ERPNext's scaffolding and Helpdesk's matching logic.

Name clash: Helpdesk calls it `default_sla`, ERPNext calls it `default_service_level_agreement`. ERPNext's name wins because the doctype is ERPNext's.

`entity` and `entity_type` are a straight gain for Helpdesk. Per customer agreements are not possible today.

## Conditions follow the Notification pattern

Helpdesk carries two condition fields today, `condition` for Python and `condition_json` for the UI built version, with nothing saying which one is in use.

Frappe's `Notification` already solves this:

```
condition_type    Select    Python / Filters
condition         Code      the Python expression
filters           Code      JSON
filters_editor    HTML      the builder UI
```

The agreement adopts that shape. One selector, two storage fields, and a filter editor that already exists in core rather than a Helpdesk specific one.

`HD Service Level Agreement.condition_json` migrates into `filters` with `condition_type` set to Filters. Agreements using the Python expression migrate into `condition` with `condition_type` set to Python.

## Child tables

`HD Service Day` and `Service Day` are field for field identical. Delete the Helpdesk one, repoint.

`HD Service Level Priority` and `Service Level Priority` are identical except the link target. See the selector section below.

## What a document type declares

Two things, both fieldnames on the document:

```
HD Ticket   status field:    status
            selector field:  priority

CRM Deal    status field:    status
            selector field:  communication_status
```

Everything else comes from meta. `HD Ticket.status` is a Link whose options are `HD Ticket Status`, so the status master resolves with one lookup. Nothing is stored twice and nothing goes stale when a doctype moves apps.

CRM is the reason these are two separate declarations. Its clock category comes from `status`, while its targets are keyed on `communication_status`. One field cannot do both jobs.

Where the registry lives is a build time decision.

## Status category drives the clock

The engine sorts every status into three buckets: open, paused, fulfilled. Transitions between buckets set `on_hold_since` and accumulate `total_hold_time`, and the accumulated hold time is added back onto the due dates.

That model is already in both apps today. Only the storage differs.

- ERPNext stores it per agreement, in `pause_sla_on` and `sla_fulfilled_on`.
- Helpdesk reads it off `HD Ticket Status.category`, which is Open, Paused or Resolved.

**One of them survives, not both. Helpdesk's way wins.** A site that adds a custom status picks up the right clock behaviour immediately, with nobody editing every agreement. One place to configure, impossible to get wrong by forgetting.

**So the status master with a category becomes a hard requirement.** A doctype that wants SLA needs a status master with a category field. `pause_sla_on` and `sla_fulfilled_on` are deprecated alongside `Issue`, kept readable so old agreements still show what they were configured with.

The only doctype that needed those child tables was `Issue`, which has a plain Select status and no master. `Issue` is being deprecated and its clock never runs again, so nothing is left that needs them.

CRM already has most of this. `CRM Deal Status` and `CRM Lead Status` have a `type` field with Open, Ongoing, On Hold, Won and Lost. Same idea, different field name and value set. Whether the contract standardises on one name or carries a value map per consumer is a build time decision.

## Target selector per document type

The child table column holding response and resolution targets is not really a priority. It is whatever field on the document selects which target applies.

```
HD Ticket   priority             -> HD Ticket Priority
CRM Deal    communication_status -> CRM Communication Status
```

These are genuinely different concepts, not two lists of the same thing. `CRM Communication Status` is Open, Replied and so on. Forcing it into a priority master would merge two different ideas into one table, which is worse than the duplication it removes.

So the agreement carries a `priority_doctype`, and the child table's column is a Dynamic Link against it. In Frappe a child row's Dynamic Link reads a field on the child itself, so the value is carried down from the parent on save.

**This is why `HD Ticket Priority` does not have to move to ERPNext.** The original plan moved it because `Service Level Priority.priority` was a plain Link and ERPNext cannot link to a Helpdesk doctype. Once it is a Dynamic Link, ERPNext links to nothing in particular. Helpdesk keeps its master, CRM keeps its own, `Issue Priority` stays in ERPNext and is deprecated with `Issue`.

Rule to hold: a new consumer uses an existing master unless its concept is genuinely different. Nothing enforces this technically.

## Hooks

The engine stays neutral. App specific behaviour lives in the app.

Core handles agreement matching, working hours, the response and resolution clock, pause and resume, and breach.

Extension points, to be worked out at build time:

- which agreement applies
- how targets are computed
- what counts as a first response
- when the clock pauses or resumes
- what happens on breach
- resetting or rolling the response clock

CRM's rolling responses is the first consumer of the last one. Every time the customer replies a new response clock starts. That is CRM's layer, contributed by CRM, not core. Helpdesk could use it too, which it cannot today.

First response is the part that does not generalise. In Helpdesk it is an outgoing email or reply. In CRM it is a status change. It needs a hook, not a setting.

## The writeback field set

Any doctype running SLA carries these, because the engine writes them. Taken from `HD Ticket` as it is today.

```
sla                               Link
service_level_agreement_creation  Datetime
agreement_status                  Select
response_by                       Datetime
resolution_by                     Datetime
first_response_time               Duration
first_response_failed_by          Duration
resolution_date                   Datetime
resolution_time                   Duration
user_resolution_time              Duration
resolution_failed_by              Duration
on_hold_since                     Datetime
total_hold_time                   Duration
status_category                   Data
```

`status_category` is already a denormalised copy of the status master's category. That makes it part of the contract rather than a Helpdesk detail.

## UI

ERPNext ships the SLA and Holiday List screens as a local link package, the same way Helpdesk links the framework UI package. The package is customisable so Helpdesk and CRM can both use it.

Helpdesk deletes its own SLA screens and renders the package.

Sequencing matters: delete only once the package works. See [../07-order-of-work.md](../07-order-of-work.md).

## Existing data

1. Create a `Service Level Agreement` for every `HD Service Level Agreement`, with `document_type` set to `HD Ticket`.
2. Copy working hours and target rows across.
3. Set `priority_doctype` to `HD Ticket Priority`.
4. Move conditions into `condition_type`, `condition` and `filters`.
5. Repoint `holiday_list` at the ERPNext list. See [holiday-list.md](holiday-list.md).
6. Repoint `HD Ticket.sla`.
7. Leave `HD Service Level Agreement` rows in place, deprecated.

The engine's site wide `track_service_level_agreement` switch has to go before any of this runs. It defaults to off and it gates the engine's entry point, so tickets would move onto an engine that finds no agreement and says nothing. See [../03-support-module-cleanup.md](../03-support-module-cleanup.md).

No matching. A site running both apps ends up with duplicate agreements, which the migrator can merge later using the same tool as customers. See [../04-migrator.md](../04-migrator.md).

`Issue`, its agreements and its frozen clock values are untouched. Existing issues keep whatever they last computed.

## Risks

This is the largest single piece of engineering in v17. Three implementations becoming one, with two of them still in use.

Name collisions between Helpdesk and ERPNext agreements on a site running both.

ERPNext ends up hosting the engine and running nothing on it, because `Issue` is deprecated and `Warranty Claim` does not use SLA. Accepted, but it means ERPNext ships a Support workspace showing an agreement list that applies to nothing in ERPNext.

The UI package is an external dependency. If it slips and Helpdesk has already deleted its screens, there is no SLA UI.

Nothing prevents a future app adding a sixth target master. Convention only.

## Open questions

Tracked in [09-open-questions.md](../09-open-questions.md).

- Whether the status category contract standardises on one field name and value set, or carries a per consumer map.
- Where the per document type registry lives.

---

[← Team](team.md) · [Permissions →](permissions.md)
