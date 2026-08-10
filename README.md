# Sip

**A café-discovery iOS app for Los Angeles, built on reviews from people you actually follow
rather than aggregate star ratings.**

Solo project. React Native / Expo, Supabase Postgres, ~3,000 real LA cafés built from open geodata.
Currently in launch prep for the App Store.

This repository is a write-up. **The application source is private** — it holds a live database's
schema and the row-level-security policies that are the app's actual security boundary, and it
isn't launched yet. What's here is the architecture, the engineering problems worth reading about,
and the numbers, all of which are verifiable against the artifacts described below.

The parts that *can* be public are:
**[open-place-toolkit](https://github.com/chrislach1546-ops/open-place-toolkit)** — nine
dependency-free modules extracted from this project's data pipeline, with 137 tests and no
dependencies.

---

## The idea

Every café app answers "what's near me, and what's its average rating?" Averages are the problem.
A 4.3 from two thousand strangers tells you nothing about whether *you* will like a place, and the
people whose taste you actually trust are invisible in it.

Sip inverts that. You follow people. You see where **they** went, what they ordered, and what they
thought. A café's rating is assembled from the people in your graph, not from the internet at
large. The unit of content is a *sip* — one drink, at one café, with a photo and a rating.

---

## Screenshots

> Being captured — this section is the one remaining piece.

---

## What it's made of

| Layer | Choice | Why |
|---|---|---|
| App | React Native + Expo SDK 57, TypeScript | One codebase, real native camera and gestures, OTA updates |
| Navigation | Expo Router | File-based routes |
| Backend | Supabase (Postgres + GoTrue + Storage + Realtime) | Managed Postgres where the security model lives *in the database* |
| Auth | Supabase Auth + Sign in with Apple | Never roll your own — hashed passwords, JWTs, OAuth, all standard |
| Map | Native map rendering with a custom style + clustering | 3,000 pins need clustering, not a list |
| Errors | Sentry, with PII scrubbed before send | `sendDefaultPii: false`, and a `beforeSend` that strips emails, IPs, and JWT-shaped strings |
| Data | OpenStreetMap + Overture Maps | Open licences. No Google or Apple Places data, deliberately — see below |

### By the numbers

| | |
|---|---|
| App source | 289 files, ~44,000 lines of TypeScript |
| Commits | 1,053 |
| Database migrations | 89 |
| Cafés in the catalog | 3,087 |
| Unit/component tests | 122 suites, 1,342 tests |
| Data-pipeline tests | 388 |
| Database security tests | 54 pgTAP files, 560 assertions |
| Public tables | 24 |
| Tables with row-level security enabled | **24 — all of them** |
| RLS policies | 61 |

---

## The three problems worth reading about

### 1. [Deciding whether two records are the same café](docs/place-matching.md)

Merging two open-geodata sources means answering "is this the same place I already have?" a few
million times. Get it wrong one way and the map fills with duplicate pins; wrong the other way and
you silently delete real cafés.

Contains: how `'steak_house'` came to match `'tea'` and put steakhouses on a café map; how a
distance-only rule quietly discarded 224 real cafés; and how a coffee shop that closed years ago
kept haunting its own address.

### 2. [Testing the security boundary, not the login screen](docs/security-model.md)

The app uses a public API key. That's not a mistake — it's how the architecture works, and it means
**row-level security is the only thing standing between one user's data and another's**. So the
policies get tested like application code: 54 pgTAP files, 560 assertions, running against a real
Postgres instance, asserting that user A *cannot* read user B's private rows.

Contains: why the anon key being public is fine, what deny-by-default actually looks like in
practice, and the bugs this caught.

### 3. [Building a catalog you can trust](docs/catalog.md)

A café catalog is not a one-time import. Sources disagree, businesses close, names get corrected,
and a tenant turns over without anyone updating the map.

Contains: the rule that no import may ever re-key or delete an existing row, how removals stay
reversible, and why every deletion ships as a migration listing each affected id.

---

## Things I'd want a reader to notice

**The security model is enforced in the database.** Not in the UI, not in a middleware layer. Every
table denies by default and grants back narrowly, and the policies have their own test suite. If
the app were compromised tomorrow, the policies would still hold.

**No Google or Apple Places data, on purpose.** Their licences forbid building a competing place
database, and the whole premise here is real opinions from real people rather than re-serving
someone else's aggregate. The catalog is OpenStreetMap and Overture, both openly licensed, with
attribution.

**Data changes are reviewable.** Every café removal is a migration that lists every affected id and
the evidence for it. Nothing is deleted by a category sweep that a human never read.

**Scraping is done politely or not at all.** robots.txt is honoured per host, requests are capped
at one per second per host, and the User-Agent says who is calling. That code is public in
[open-place-toolkit](https://github.com/chrislach1546-ops/open-place-toolkit).

---

## Status

Feature-complete backend; in App Store submission prep. Remaining work is Apple Developer Program
enrolment, the store listing, and a production build — not engineering.

## Attribution

Café data derived from [OpenStreetMap](https://www.openstreetmap.org/copyright) (ODbL) and
[Overture Maps](https://overturemaps.org/). OSM-derived Overture records remain ODbL.
