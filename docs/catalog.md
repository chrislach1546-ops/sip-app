# Building a catalog you can trust

A café catalog is not a one-time import. Sources disagree, businesses close, names get corrected
upstream, and a tenant turns over without anyone updating the map. The catalog holds **2,600 open
Los Angeles cafés** (September 2026), drawn from Overture Maps and OpenStreetMap plus a few
hand-seeded, and it has been rebuilt, corrected, expanded, verified and pruned across more than 170
migrations.

The whole design follows from one constraint.

---

## The constraint: a café id is a foreign key to someone's memory

Every sip, review, note, list entry, message card and presence row points at a café by id. So an
import that re-keys a row doesn't just churn a database record — it detaches the photo somebody
took of their coffee from the place they drank it.

That gives the rule the entire pipeline is built around:

> **An import may correct a name or nudge a pin. It may never re-key an existing row, and it may
> never delete one.**

Insertions are additive. Corrections happen in place. Deletions are a separate, deliberate,
human-reviewed act.

### Proving it, rather than asserting it

Before and after every import, the full id list is dumped and diffed:

```console
$ comm -23 cafes-before-ids.txt cafes-after-ids.txt
$
```

Empty output is the pass condition — no id present before is missing after. That ran after every
commit of the largest import, each time against the *original* pre-migration list rather than the
previous step's, so drift couldn't accumulate unnoticed. It printed nothing every time.

Backed up by six foreign-key orphan checks across `sips`, `cafe_reviews`, `cafe_notes`,
`list_items`, `messages` and `profiles.current_cafe_id` — all returning zero.

The single largest import took the catalog from 1,293 to 2,978 cafés: **+1,685 new rows and 910
in-place refreshes, with not one existing id changed and not one row deleted.**

---

## Imports must be idempotent, and that took two tries

Running the importer twice should propose nothing the second time. The first attempt at that
proposed **130 spurious inserts**. Two real defects, both worth naming:

**Coverage-frontier creep (121 rows).** The "stay within 1,500m of the existing catalog" test
measured against the *live* catalog — which the import had just grown by 1,685 rows. Every run
would push the boundary outward, expanding the map indefinitely. It now anchors on the stable
pre-import catalog.

**Reconciliation losers resurfacing (9 rows).** Applying a winning candidate's coordinates moved
the matched café away from its losers, which then matched nothing and would have been inserted as
duplicates of the very café they belonged to. Contested rows now keep their coordinates.

Now every run reconciles all 6,467 considered candidates into named buckets that sum to exactly
6,467, and **the importer throws if that sum ever falls short** — so a candidate can never vanish
silently. A second run is a clean no-op: zero inserts, zero updates.

---

## The bug that structurally couldn't be caught by tests

A dry run reported 1,142 updates. Those 1,142 updates covered only **910 distinct ids**.

189 cafés were receiving two to five conflicting updates each, with whichever came last in the file
winning arbitrarily. One café would have ended up named *"Yifang Taiwan Fruit Tea Old Town
Pasadena "* instead of *"Prolece Tea"* — while every sip ever taken there stayed bound to that id.

No unit test would have caught it. Each individual update was correct; the defect only existed in
the *set*. It was found by a reviewer reading the dry-run output and noticing the two numbers
should have matched. The fix was one-to-one reconciliation, and the count check above now makes the
same class of problem loud.

That's the argument for dry runs that emit a reviewable artifact rather than importers that just
run.

---

## Deletions: reviewable, listed, reversible

Cafés do get removed — boba shops when the catalog briefly narrowed to coffee (a ruling later
reversed — the case rule 3 below is for), places that aren't cafés at all, businesses
that closed. Every removal obeys three rules:

**1. It ships as a migration listing every affected id**, with the tags or evidence justifying each
one. Nothing is deleted by a category sweep no human read. When 91 rows were removed in one pass,
each had an individual verdict written down first — restaurants with a named cuisine, retail
outlets, venues, a car-meet event, a TV studio, a cryotherapy clinic, a cookie company, and nine
places researched individually and found closed.

**2. It updates a denylist**, so a later import can't quietly re-add it. The denylist binds on
both the insert and the update path, and — because sources use different id namespaces — it matches
on *place*, not only on id, so a removed café can't return under a new source's unfamiliar id.

**3. It shrinks the denylist on restore.** Removals aren't one-way. When two cafés were restored
after review, the denylist came *down* by two. A denylist that only ever grows is a ratchet, and a
ratchet eventually locks out something real.

Two rows sit permanently unresolved — the deciding evidence is a photograph nobody has taken — and
a test pins that they are still allowed to exist, so a future cleanup can't sweep them up by
accident.

---

## What's still not solved

**Closure was under-detected, and is now mostly checked.** Nine dead cafés turned up in a
hand-sample of twenty-one, while the automated closure re-check returned zero. So the whole catalog
got a liveness pass (August 2026): **2,133 cafés verified open, 18 retired, 11 renamed.** The rows
that pass couldn't settle went to a hand check in September, which retired 13 more and confirmed 20.

It isn't finished. About 485 open cafés still carry no verification stamp, and how many of those
were never really checked hasn't been measured. A catalog that confidently lists a café that closed
last year is exactly the failure this project is supposed to avoid, so that number is written down
rather than rounded away.

**City names are partly placeholder.** Resolved by true polygon containment where boundaries exist,
nearest-node where they don't, and still carrying a county-level fallback for some rows.

---

## Where the data comes from, and what's kept out

The catalog's *rows* come from OpenStreetMap and Overture Maps, both openly licensed, with
attribution carried in every generated migration header. When a café's own website is read for
opening hours, it's done politely: robots.txt honoured per host, one request per second, an honest
User-Agent, and never an aggregator. That code is public in
[open-place-toolkit](https://github.com/chrislach1546-ops/open-place-toolkit).

Open data is where a row starts, not proof that it's right. Names carry typos and old trading
names, and pins drift onto the business next door. So cafés are checked against their current
public listings in hand-run passes: is it still open, is the name right, is the pin on the right
storefront. Each pass writes a proposal a human reads, and whatever changes ships as a migration
like every other catalog change.

**Ratings are the one thing never imported.** Every rating in Sip has a real, logged visit behind
it: someone was there, ordered something, and said what they thought of it. Seeding the catalog
with another platform's star averages would put ratings in the app that nobody in the community
actually gave, which defeats the premise before the first user arrives. Third-party ratings are
rejected outright, not deferred.
