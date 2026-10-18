# Notes: update-user endpoint

## Plan
I approved a plan to add `PUT /users/:id` in `routes/users.js`, with a new
`updateUser(id, { name, email })` helper in `db/store.js`. The helper reuses
`getUserById` and returns `undefined` for an unknown id, so the route decides
when to send a 404, the same way `GET /:id` does. Validation runs before the
lookup and is stricter than `POST`: name and email must be non-blank strings,
so `123` or `"   "` gets a 400, not only a missing field. A non-numeric id
becomes `NaN`, matches no user and gets a 404, so nothing crashes. I approved
the plan without edits. It already covered the route, the store helper, both
error cases and lint, which CI runs.

## Model
Opus 5.5. The change is small, but the review step and the edge cases
(validation order, bad ids, unexpected body fields) take careful reasoning.
On a small diff the cost difference is minor.

## Commits
1. `Add updateUser helper to the in-memory store`: the data-access change on its own.
2. `Add PUT /users/:id endpoint with validation and not-found handling`: the
   route. The update-user tests turn green here.
3. `Store trimmed name and email on PUT /users/:id`: the fix from review.
4. This NOTES.md.

I split it this way so each commit makes sense alone: the store layer, then the
HTTP layer that uses it, then the write-up. I ran `npm test` and `npm run lint`
after each step.

## Review
I ran `/code-review` on the branch before opening the PR. It found no crashes.
It checked that a missing, `null` or array body can't crash the route; each
gets a 400. It flagged 7 lower-severity issues, and I checked each one myself:

- **Fixed:** validation checked the trimmed name and email, but the
  untrimmed values were saved, so `"  Ada  "` kept its padding. Commit 3
  saves the trimmed values.
- **Real, but out of scope:** `POST /users` validates less strictly than
  PUT (it accepts blank or non-string values), and the two routes repeat
  the validation logic. A shared `validateUserInput` helper would fix both,
  but this change shouldn't alter POST. Left as a follow-up.
- **Not required:** no email format or uniqueness check. The task doesn't
  ask for one. Follow-up.
- **Intentional:** validation runs before the not-found check, so
  `PUT /users/9999` with a bad body returns 400, not 404. `/users/1.0`
  reaches user 1 because PUT parses ids with `Number()`, the same as the
  existing `GET /:id`. `updateUser` has no guard against partial input, but
  its only caller always passes both fields.

Before the review I also tried edge cases by hand: a non-numeric id gets a
404, a non-JSON body gets a 400, and `id` or other extra fields in the body
are ignored.
