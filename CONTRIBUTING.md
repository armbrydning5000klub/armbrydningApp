# Developer Guide

A practical guide for making changes to this project — written so someone unfamiliar with the codebase can get productive quickly.

---

## First Steps

1. Clone the repository, run `npm install`
2. Ask whoever holds the credentials for the `.env` file contents (see README.md → Getting Started Locally)
3. Run `npx expo start --web` to confirm everything works before changing anything

---

## Project Structure Cheat Sheet

| I want to... | Look in... |
|---|---|
| Change what a screen looks like | `src/views/` |
| Change what data a screen loads/saves | `src/viewmodels/` (hooks) |
| Change business rules (e.g. leaderboard thresholds) | `src/services/` |
| Change how data is read/written to Supabase | `src/repositories/` |
| Add a new screen/route | `app/` (Expo Router — filename = URL path) |
| Change database structure or add a new secure function | Supabase SQL Editor directly, then update the matching repository |
| Change something only admins should be able to do | Consider whether it needs an Edge Function (`supabase/functions/`) — anything requiring the `service_role` key must go there, never in client code |

---

## Common Tasks, Step by Step

### Adding a new field to a user's profile

1. Add the column in Supabase SQL Editor: `ALTER TABLE users ADD COLUMN ...`
2. Update `src/models/User.ts` — add the field to the `User` interface
3. Update `src/repositories/UserRepository.ts`:
   - Add to `UserRow` interface (snake_case)
   - Add to `mapUserRow()` / `mapPublicUserRow()` (translates snake_case → camelCase)
   - Add to `toRow()` (translates camelCase → snake_case for writes)
4. If the field is sensitive (like birth date/weight), consider whether it needs the same column-privilege protection described in README.md → Database
5. Update whichever view/form should let users edit or see the new field

### Adding a new screen

1. Create the view component in `src/views/`
2. Create a matching route file in `app/` (e.g. `app/my-screen.tsx`) that imports and renders it
3. If it needs its own data, create a viewmodel hook in `src/viewmodels/`
4. If it needs a new route in the navigation stack (e.g. a modal), register it in `app/_layout.tsx`

### Adding a new database function that changes sensitive data

Follow the pattern already used by `confirm_club_match` and `void_club_match`:

1. Write the function as `SECURITY DEFINER` in SQL, with `SET search_path = public`
2. Inside the function, **explicitly check** `auth.uid()` against whatever permission is required — never assume the caller is authorized just because they could call the function
3. If updating multiple rows that could race (e.g. two players' ratings), use `SELECT ... FOR UPDATE` in a **consistent order** (e.g. always by `id` ascending) to avoid deadlocks and lost updates
4. Grant execute only to `authenticated`, and revoke from `PUBLIC` explicitly:
   ```sql
   REVOKE EXECUTE ON FUNCTION my_function(...) FROM PUBLIC;
   GRANT EXECUTE ON FUNCTION my_function(...) TO authenticated;
   ```

### Adding an Edge Function (for actions requiring `service_role`)

Use `admin-reset-password` or `admin-reject-member` in `supabase/functions/` as a template. The pattern every Edge Function should follow:

1. Create two Supabase clients: one using the **caller's** auth token (to verify who is asking), one using `service_role` (to actually perform the privileged action)
2. **Always** verify the caller's identity and role using the first client, before touching anything with the second
3. Return clear error responses (`{ error: "..." }`) rather than throwing unhandled exceptions
4. Include CORS headers on every response, including error responses — otherwise the browser silently blocks the response and you'll see a generic network error with no useful message
5. Deploy with `supabase functions deploy <name>`

### Making a UI change

Standard flow. One thing worth knowing: this project follows MVVM strictly — if you find yourself wanting to call `supabase` directly from a `.tsx` file in `src/views/`, that's a sign the logic belongs in a repository/service/viewmodel instead.

---

## Testing

```bash
npm test           # run all tests
npx tsc --noEmit    # type-check without building
```

`GlickoMath.test.ts` is a good template for testing pure functions — it tests *properties* the algorithm must hold (e.g. "an upset win gives more points than an expected win"), not just that the code executes.

If you change `glicko2_update` in SQL, there's no automated test for it — verify manually by calling it directly:
```sql
SELECT * FROM glicko2_update(1500, 350, 0.06, 1500, 350, 1);
```
Compare against the JS version's output for the same inputs (they should produce nearly identical numbers, since both implement the same Glicko-2 algorithm).

---

## Deployment

Deployment is **manual by design** — pushing to `main` only runs tests, it does not deploy. To deploy:

1. GitHub → Actions → "CI and Deploy" → "Run workflow"
2. Wait for both the `test` and `deploy` jobs to show green

If you need to change the deployment process itself, edit `.github/workflows/deploy.yml`. See README.md → Known Pitfalls before touching the Vercel export step — there's a non-obvious fix in `scripts/fix-web-export.js` that must run before every Vercel upload.

---

## Domain Concepts Glossary

| Term | Meaning |
|---|---|
| **Arm** | `'left'` or `'right'` — every player has two entirely separate ratings, one per arm |
| **RD (Rating Deviation)** | How uncertain the system is about a rating. Starts at 350, decreases as more games are played. Lower = more confident |
| **Established / Provisional** | A player only shows on the main leaderboard ("established") if their RD is below 200 **and** they've faced at least 4 distinct opponents **and** belong to a connected cluster of at least 5 players. Otherwise they show in a separate "not enough data yet" section |
| **Connected cluster** | A graph of players linked by having played each other (directly or via a chain of opponents). Prevents rating comparisons between players who've never had any common opponents |
| **Club match** | A casual, one-off match between two club members, confirmed via a "handshake" flow (one reports, the other confirms) |
| **Tournament / Supermatch** | Two other match formats, built but currently hidden from the UI — the code exists in `src/services/TournamentService.ts` / `SupermatchService.ts` if reintroduced |
| **Judge** | A role that can confirm a club match on behalf of the two players (e.g. a referee at an event). Currently only assignable via SQL — no UI for it yet |

---

## Things to Be Careful With

- **Never** write rating values directly from client code — always through `confirm_club_match` / `void_club_match`. Client-side RLS policies deliberately block direct writes to rating columns.
- **Never** put the `service_role` key in any client-side file, `.env` value prefixed `EXPO_PUBLIC_`, or committed code. It belongs only in Supabase/GitHub secrets, used only inside Edge Functions.
- When editing SQL functions with iterative/numerical logic (like the Glicko-2 port), double-check variable names don't accidentally get reused for two different purposes — see README.md → Known Pitfalls, item 5, for a real example of how subtle this bug can be.
