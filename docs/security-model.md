# Testing the security boundary, not the login screen

Sip stores real people's accounts, their locations, their private notes, and their messages. The
part that protects all of it is not the login screen. It's 65 row-level-security policies in
Postgres — and those policies have their own test suite, 86 files and 2,241 assertions, run like
application code on every push.

---

## The API key in the app is public, and that's the design

Every copy of the app ships with a Supabase **anon key**. Anyone can extract it from an `.ipa` in
about a minute. That is not a leak — it's how the architecture works.

The anon key is an *identifier*, not a password. It says "a Sip client is calling." It carries no
authority by itself. What decides whether a given row can be read is the **user's** JWT plus the
RLS policy on that table. Which means:

> The security boundary is not "can you reach the database." It's "given that you *have* reached
> the database, what can you see?"

That framing changes what's worth testing. A login screen test proves a form works. An RLS test
proves that an authenticated stranger asking for your private sip **gets nothing back**.

The service-role key — the one that bypasses RLS entirely — never appears in the app bundle, never
appears in the repository, and is read from the environment in the handful of server-side scripts
that need it. That's verified: scanning every blob in the git object database, including 151
commits unreachable from any branch, turned up no real credential in the project's entire history.
Since September a secret scanner (gitleaks) re-checks the full history on every push.

---

## Deny by default, on every table, no exceptions

| | |
|---|---|
| Public tables | 35 |
| Tables with RLS enabled | **35** |
| Tables without RLS | **0** |
| Policies | 65 |

<sub>Measured 25 September 2026.</sub>

There is no "we'll add policies to that one later" table. A table without policies denies
everything by default, which is the correct failure mode, and no table ships without explicit
policies granting back exactly what's intended.

The policies are narrow. On `sips`, for example:

```
insert own sips        INSERT
update own sips        UPDATE
delete own sips        DELETE
read visible sips      SELECT
```

Writes are scoped to the owner. Reads go through a visibility rule that has to account for public
sips, friends-only sips, private sips, blocked users, and hidden-by-moderation posts that their
*author* can still see.

---

## What the tests actually assert

pgTAP runs inside Postgres, so the tests execute as a real authenticated user against the real
policies — not against a mock, and not through the app's own data layer, which could be bypassed.

The assertions that matter are the **negative** ones:

```sql
select is(
  (select count(*) from public.sips
     where id = 'aaaaaaaa-0000-0000-0000-000000000000')::int,
  0,
  'accepted friend CANNOT see a private sip');

select is(
  (select count(*) from public.sip_ratings
     where sip_id in (select id from public.sips where user_id = '1111...'))::int,
  0,
  'stranger cannot read ratings on invisible sips');
```

That second one is the class of bug this suite exists to catch. The *sip* was correctly hidden —
but its **ratings** live in a different table, and a naive policy there would leak the rating of a
post you were never allowed to see. Visibility has to be enforced on every table that references
the hidden thing, not just the one holding it.

Other things the suite pins:

- A sip inserted with **no** visibility specified defaults to `private`, not public. Fail-safe, not
  fail-open.
- A blocked user can't read, message, or appear to the person who blocked them — across every
  surface, not just the obvious one.
- Users keep access to **their own** posts when a moderator hides one. Moderation shouldn't lock
  you out of your own writing.
- Rate limits live in the database, so they can't be skipped by talking to the API directly instead
  of through the app.

---

## What this caught

**Privileges are not policies.** A `GRANT` and an RLS policy are different things, and having the
right policy with the wrong column grant produces a table that looks secure and isn't. There's a
dedicated test file for privileges alone because I got this wrong once.

**Aggregates leak.** A café's rating is computed from sips, including private and friends-only
ones. Getting that right meant the *aggregate* could be published while the individual rows behind
it stayed hidden — and proving it required tests asserting you can see the number but not the rows.

**Denial paths need their own file.** It's easy to test that the right person *can* do a thing. The
suite has a file dedicated to asserting the wrong person *cannot*, because that's the assertion
that actually protects a user, and it's the one that silently stops being true when a policy is
edited.

---

## Two residuals, disclosed rather than hidden

Both are written into the relevant migration headers rather than left for someone to discover:

- **A three-mark award can be raced.** Two concurrent inserts can each read `marked = 2` and both
  be granted, producing four. The fix is an advisory transaction lock; it isn't applied yet because
  the window is small and the consequence is cosmetic.
- **`on_site` is client-asserted.** The phone checks that you're within 75 m of the café before a
  sip can be shared, and a database trigger refuses a shared sip that arrives without that flag. But
  the phone decides the flag, so a modified client could still claim it from anywhere. Closing that
  would mean sending raw coordinates to the server, which the privacy policy promises doesn't happen,
  and the privacy guarantee was judged worth more than the hole. The trigger closes the likelier
  gap: a future code path that forgets the check.

Writing these down is the point. A known, documented limitation is a decision. An undocumented one
is a surprise waiting for whoever reads the code next.
