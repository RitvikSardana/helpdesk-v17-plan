# Order of work

[Index](README.md) · previous: [06-app-integration.md](06-app-integration.md) · next: [08-future-scope.md](08-future-scope.md)

## What this is

Sequencing, not new design. Every decision here was made in an earlier section. This one says what has to happen before what, and what it costs when something arrives late.

## The one hard deadline

ERPNext v17 ships about a month out. Helpdesk's suite work lands about four months out. Both numbers are ballparks from [00-overview.md](00-overview.md), but the gap between them is the shape of this section.

**Anything Helpdesk needs inside ERPNext's schema has a one month window. Everything else has four.**

That single asymmetry decides the order. Work that has to be in ERPNext v17 goes first even when it is small, and work that lives in Helpdesk waits even when it is large.

## The tracks

Six of them. They are not equal in size and they do not start together.

| # | Track | Lives in | Cannot start until |
|---|---|---|---|
| A | ERPNext schema changes | ERPNext | the contacts question is answered |
| B | The SLA engine | ERPNext | the status contract is settled |
| C | Support module deprecation | ERPNext | nothing |
| D | Helpdesk repoints and deprecates | Helpdesk | A and B are merged |
| E | The migrator app | its own app | the deprecation list is final |
| F | HRMS leave | Helpdesk | nothing |

A and B are the only ones on the one month clock. C is also in ERPNext but it removes rather than adds, so it can follow.

## The critical path

```
the contacts question
        ↓
ERPNext schema changes              ← one month window
        ↓
Helpdesk repoints its link fields
        ↓
the upgrade patches
        ↓
testing on a copy of a real site
```

Everything else hangs off this line. The SLA engine is the largest single piece of work in the plan and it is still not the critical path, because it does not block the repoint.

## What has to be answered before anything starts

Two questions from [09-open-questions.md](09-open-questions.md), and they do not cost the same.

**Contacts on Customer: child table or Dynamic Link.** This one is on the critical path. It decides a field on `Customer`, which decides what goes into ERPNext v17, which has a one month window. Answer it first or track A cannot start.

**The automatic Customer role assignment.** Blocks track D's permission work, not the schema. Can be answered a month late without moving anything.

## What can start on day one

Track C and track F. Neither waits on a decision and neither blocks anyone.

Track F is the obvious first thing to build. It is small, it is off by default, it touches one behaviour, and shipping it proves the four app bench works before anything irreversible depends on that.

Track C is the announcement risk rather than the code risk. The forum thread is already public, so the sooner the deprecation is visible the less it reads as a surprise. See [03-support-module-cleanup.md](03-support-module-cleanup.md).

## Where the migrator sits

It ships as part of v17 so the flows can be tested together, but it is not on the critical path and it does not gate the upgrade. [01-decisions.md](01-decisions.md) pins that: migration never blocks the upgrade.

One dependency used to run the wrong way and no longer does. The patch needs a record of every `Customer` it created, and the merge log that record belongs in is the migrator app's, which may not be installed when the patch runs.

Settled in [01-decisions.md](01-decisions.md): the patch marks its own side, on the deprecated `HD Customer`, and the migrator reads the marks on first run. No cross app write and no ordering constraint between Helpdesk and the migrator.

Worth knowing why a fourth option was never available: after the patch, a created and a linked `Customer` are indistinguishable. Some record has to be written at patch time, whoever ends up owning it. See [04-migrator.md](04-migrator.md).

## One ordering constraint that crosses sections

The User Permission and DocShare mirrors keep a customer's portal access working on both masters today. Delete the mirrors before those rows are repointed at `Customer` and access depends on whichever master the rows happen to sit on.

So inside track D: **repoint the permission rows, then delete the mirrors.** Never the other order. See [06-app-integration.md](06-app-integration.md) and [02-shared-masters/permissions.md](02-shared-masters/permissions.md).

## What slips, and what it costs

The useful question is not whether something is late. It is whether being late is recoverable.

| If this misses | Cost | Recoverable |
|---|---|---|
| `Customer.domain` and `country` miss ERPNext v17 | Helpdesk creates Custom Fields instead, which is already the contingency | yes, the pattern exists |
| The SLA fields miss ERPNext v17 | worse. The engine is ERPNext's and it cannot be shimmed from Helpdesk | no, track B waits for v17.1 |
| The Dynamic Link conversion misses | `default_priority` still points at `Issue Priority`, so Helpdesk cannot use the engine at all | no |
| The migrator is not ready | the upgrade still ships. Sites carry duplicates for longer | yes, by design |
| HRMS leave is not ready | drop it from the release | yes, it is a feature not a migration |

The pattern: **anything landing in ERPNext is hard to recover from, anything landing in Helpdesk or the migrator is soft.** That is the argument for front loading track A and track B even though they feel like schema plumbing rather than product work.

## Testing is not a phase at the end

[05-upgrade.md](05-upgrade.md) lists eight site shapes to verify. They need a copy of a real site with history and the whole bench upgraded, not Helpdesk's patches alone, because the interesting failures come from the order things run in.

That environment takes time to assemble and it is needed before the patches are finished, not after. Two checks in particular should happen early because they can invalidate design rather than find bugs:

- **Does a `Customer` insert survive on an unconfigured ERPNext?** `validate_customer_group` reads the group, and the defaults come from the setup wizard. If it does not, the standalone option changes shape. See [05-upgrade.md](05-upgrade.md).
- **Does the SLA clock still run after the `track_service_level_agreement` reads are removed?** That is the regression that would otherwise ship silently.

Neither needs the patches written. Both can be answered now.

## Risks

The one month window is the whole risk. Everything else in this plan has slack and track A does not.

Deciding the contacts question slowly. It is one question, it looks like a detail, and it is the gate on the only deadline in the plan.

Treating the migrator as part of the upgrade. It is not, and letting it slip into the critical path would turn a soft dependency into a hard one for no gain.

## Open questions

None. The merge log ownership question is settled in [01-decisions.md](01-decisions.md).

---

[← App integration](06-app-integration.md) · [Future scope →](08-future-scope.md)
