# Deciding whether two records are the same café

Sip's catalog is assembled from two open sources — OpenStreetMap and Overture Maps — that overlap
heavily, disagree constantly, and each know about places the other doesn't. Merging them means
answering one question a few million times: **is this candidate a café I already have, or a new
one?**

Both wrong answers are expensive. Say "same" too eagerly and real cafés silently vanish. Say
"different" too eagerly and the map fills with duplicate pins on the same doorway. This is the
hardest problem in the project, and all three stories below are ways I got it wrong first.

---

## 1. How `'steak_house'` came to match `'tea'`

OSM encodes multi-valued tags as a single semicolon-joined string: `cuisine=coffee_shop;sandwich`.
The original category check did the obvious thing:

```js
cuisine.includes('tea')   // is this a tea house?
```

`'steak_house'.includes('tea')` is **true**. There's a `tea` inside `s-TEA-k`.

That one fact produced three separate shipped defects, in three different parts of the pipeline:

- The OSM importer queried `cuisine~"...|tea"`, so **every steakhouse in the bounding box was
  imported as a café.**
- The kind classifier tested `cuisine.includes('tea')`, so those rows were labelled *Tea House*.
  Ruth's Chris, Outback, Morton's and Mastro's all shipped as tea houses.
- A research query using `ILIKE '%tea%'` over Overture categories returned **325 steakhouses and 93
  sports teams** — `amateur_sports_team` contains `tea` as well.

The fix isn't a longer list of exceptions. It's a rule, and one place that enforces it:

```js
/** OSM encodes multi-values as `a;b;c`. Compare TOKENS, never the raw string. */
export function tokensOf(value) {
  return String(value).split(';').map((t) => t.trim().toLowerCase()).filter(Boolean);
}
```

**Anchored equality or whole-token membership. Never a substring test, never an unanchored regex.**
Every category decision in the project now routes through a single module, so there is exactly one
place to get it right, and it has a test suite whose job is to stay paranoid about this.

The lesson I actually took: the bug wasn't `includes()`. It was that the same judgement was
implemented independently in three places, so fixing one didn't fix the others.

---

## 2. The distance rule that quietly deleted 224 real cafés

Matching starts with proximity. Two thresholds, both measured against a real run rather than
guessed:

```js
export const SAME_SPOT_M  = 40;   // within 40m: it's the same place, ignore the names
export const NAME_MATCH_M = 120;  // 40–120m: same place only if the names also match
```

The 40-metre branch **ignores names entirely, deliberately.** That's what lets a business whose
name was corrected upstream — `Acuarela` → `Aquarela`, a single letter — be recognised as the same
row rather than inserted a second time. Fuzzy name matching would also merge genuinely different
cafés, so proximity does that job instead.

Then I measured what the rule was actually doing. Of 302 candidates suppressed as duplicates,
**222 paired a candidate with a business under a completely different name:**

| Candidate | Suppressed against | Distance |
|---|---|---|
| Peet's Coffee & Tea | Starbucks | 39 m |
| The Coffee Bean & Tea Leaf | blkdot coffee | 27 m |
| Ding Tea | First Flight | 7 m |

Those are real, distinct cafés in dense city blocks that a proximity-only rule glued onto whichever
neighbour happened to be closest. **Roughly 200 real Los Angeles cafés were being silently dropped
by a project whose entire purpose is to not drop cafés.**

The fix made the 40-metre branch name-*aware* rather than name-blind, and **224 real cafés came
back** — without renumbering or deleting a single existing row.

What makes this worth telling isn't the fix. It's that the rule looked correct, passed its tests,
and was quietly wrong at the scale it actually ran at. Nothing surfaced it except deliberately
measuring the decisions it had made.

---

## 3. The café that closed years ago and kept haunting its address

Then the opposite failure arrived, from a bug report: a pin on the live map for **Two Guns
Espresso**, sitting on top of a shop called *Bread, Espresso &* — 4.8 metres away.

Two Guns had closed. The new tenant had moved in. Both records existed upstream.

The name-blind 40-metre rule had collapsed them correctly. The name-*aware* fix — the one that
recovered 224 cafés — un-collapsed them, because it read "different name, same spot" as *two
neighbouring businesses.* At 4.8 metres apart, though, that reading is equally consistent with
**tenant turnover**, and that's what this was.

Two facts settled it. Overture scores the phantom at **0.398 confidence** against **0.971** for the
brand's real, still-open shop in another city. And a genuine neighbour at 4.8m is essentially
impossible; that's the same doorway.

Only the phantom pin was removed. The real Two Guns location and three other records for it all
survive, pinned by a test so a future cleanup can't take them out.

**The root cause is a class, not two rows.** The importer read neither `categories.alternate` nor
`confidence`, so it was blind to both signals that would have caught this. A read-only audit script
now measures how many other rows sit in that blast radius, and a later sweep removed 91 more
non-cafés found that way — restaurants with a named cuisine, retail, a car-meet event, a TV studio,
a cryotherapy clinic and a cookie company.

---

## What generalises

**Proximity and names are both necessary and neither is sufficient.** Same spot + same name is
easy. Same spot + different name is genuinely ambiguous — neighbour, rename, or tenant turnover —
and no distance threshold separates those, because I checked: the rename case sits at 4.0m and a
confirmed different-business case at 3.8m.

**Measure what your rule decided, not just whether it ran.** The 222 collapses were invisible until
I made the pipeline write down every suppression with both names and the distance.

**Removals must be reversible and reviewable.** Every café deletion ships as a migration listing
each id and the evidence, plus a denylist so a re-import can't quietly re-add it. Two rows sit
permanently unresolved — the deciding evidence is a photograph — and a test pins that they're still
allowed to exist.

The matching code itself is public in
[open-place-toolkit](https://github.com/chrislach1546-ops/open-place-toolkit), including the
spatial index and the threshold reasoning.
