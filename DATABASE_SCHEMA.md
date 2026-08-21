# Database Schema Reference

Full reference for all tables, key functions, and access rules. See `CONTRIBUTING.md` for how to make changes safely.

---

## Tables

### `clubs`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key |
| `name` | text | |
| `location` | text | |
| `created_date` | timestamptz | |

**RLS:** Everyone (authenticated) can `SELECT`. Only `super_admin` can `INSERT`.

---

### `users`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key, matches `auth.users.id` |
| `name` | text | Display name |
| `username` | text | Login handle (used to build the fake email `username@armbrydning.local`) |
| `club_id` | uuid | FK → `clubs.id`, nullable |
| `roles` | text[] | Any of `'member'`, `'judge'`, `'club_admin'`, `'super_admin'` |
| `status` | text | `'pending_approval'` \| `'active'` \| `'deactivated'` |
| `rating_left`, `rating_left_rd`, `rating_left_volatility` | numeric | Glicko-2 state, left arm |
| `rating_right`, `rating_right_rd`, `rating_right_volatility` | numeric | Glicko-2 state, right arm |
| `weight`, `height` | numeric, nullable | **Sensitive** — general `SELECT` revoked, see below |
| `birth_date` | date, nullable | **Sensitive** — general `SELECT` revoked, see below |
| `gender` | text, nullable | `'male'` \| `'female'` |
| `consent_date` | timestamptz | When the user accepted the privacy policy |
| `parental_consent_given` | boolean, nullable | `null` = not a minor at signup; `true`/`false` = minor's self-affirmed parental consent status |
| `created_date` | timestamptz | |

**Sensitive column protection:** `birth_date`, `weight`, `height` have had general `SELECT` privilege **revoked** from `authenticated`/`anon`. They are only readable via:
- `get_my_profile()` — own data
- `get_member_profile_for_admin(user_id)` — admin only, same-club or super_admin
- `get_classification(user_id)` / `get_classifications_bulk(ids)` — returns only the *derived* age/weight class labels, never the raw values

**RLS on the table itself:**
- `SELECT`: active users' non-sensitive columns visible to everyone authenticated; admins can see pending members in their club (via `is_admin_for_club()` to avoid recursion)
- `INSERT`: only your own row (`id = auth.uid()`)
- `UPDATE`: own profile fields only (name/weight/height — **not** roles, status, or ratings, enforced via `WITH CHECK`); admins can update `status`/`roles` within their club
- Rating columns cannot be updated by any policy — only by the `SECURITY DEFINER` functions below

---

### `matches` (club matches — the primary match type in current use)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | |
| `player_a_id`, `player_b_id` | uuid | FK → `users.id` |
| `winner_id` | uuid | FK → `users.id`, must be A or B |
| `date` | timestamptz | |
| `arm` | text | `'left'` \| `'right'` |
| `reported_by` | uuid | Must be player A or B |
| `confirmed_by` | uuid, nullable | Set on confirmation |
| `status` | text | `'pending_confirmation'` \| `'confirmed'` \| `'cancelled'` \| `'voided'` |
| `rating_a_before`, `rating_b_before`, `rating_a_after`, `rating_b_after` | numeric, nullable | Filled in only on confirmation |

**RLS:**
- `SELECT`: confirmed matches visible to everyone; a pending match visible only to the two players involved
- `INSERT`: only one of the two players, and only as `reported_by = auth.uid()`
- No direct `UPDATE` policy exists — status changes happen exclusively through `confirm_club_match()` / `void_club_match()`

---

### `tournaments`, `tournament_matches`, `supermatches`, `supermatch_games`
Support the two alternate match formats (bracket tournaments, best-of-N supermatches). Currently **read-only** via RLS (`SELECT` policies only) — the recording UI is hidden, and no `INSERT` policy exists yet. If reintroduced, add matching `INSERT`/`UPDATE` policies and ideally port the confirmation logic to a locked `SECURITY DEFINER` function the same way club matches work.

---

## Key Functions (all `SECURITY DEFINER`, `SET search_path = public`)

| Function | Args | Returns | Purpose |
|---|---|---|---|
| `glicko2_update` | rating, rd, volatility, opp_rating, opp_rd, score | `(new_rating, new_rd, new_volatility)` | Pure Glicko-2 math for one game. Includes safety clamps: rating swing capped at ±400 per game, RD capped 30–350, volatility capped 0.01–0.15 |
| `confirm_club_match` | match_id | void | Verifies caller is the other player or a judge, locks both player rows (`FOR UPDATE`, consistent order by `id`), computes and applies the rating update, marks the match confirmed |
| `void_club_match` | match_id | void | Admin-only. Rolls the rating **values** back to their pre-match snapshot (does not restore old RD/volatility — see note below) |
| `get_my_profile` | — | `users` row | Full own profile, including sensitive fields |
| `get_member_profile_for_admin` | target_user_id | `users` row | Full profile of another user, only if caller is admin for their club |
| `get_classification` | target_user_id | `(age_category, weight_class)` | Derived labels only |
| `get_classifications_bulk` | user_ids[] | table of `(user_id, age_category, weight_class)` | Batched version, used by leaderboard/matchmaking to avoid N+1 queries |
| `get_age_category` | birth_date | text | `'U15'` / `'U18'` / `'U21'` / `'Senior'` / `'Masters'` / `'Grandmaster'` / `'SeniorGrandmaster'` / `'UltraGrandmaster'` |
| `get_weight_class` | age_category, gender, weight | text | IFA-style weight class threshold lookup |
| `is_admin_for_club` | target_club_id | boolean | Helper used inside RLS policies — must be `SECURITY DEFINER` itself to avoid infinite RLS recursion when a policy needs to query `users` to check the caller's own role |

**Note on `void_club_match`:** it restores `rating_left`/`rating_right` to the pre-match value stored on the match row, but RD and volatility are **not** stored per-match and so are not rolled back — they remain at whatever they've drifted to since. This is a known, accepted limitation for an admin "undo a mistake" tool, not a bug.

---

## Edge Functions (Deno, in `supabase/functions/`)

| Function | Purpose | Auth model |
|---|---|---|
| `admin-reset-password` | Resets a member's password | Verifies caller's JWT via a client scoped to the caller's own token, checks their role via a `service_role` client, only then calls `auth.admin.updateUserById` |
| `admin-reject-member` | Rejects a pending applicant — deletes both the `users` row and the Auth account | Same two-client pattern; refuses to act on anyone whose `status` isn't `pending_approval`, as a safety check against accidentally deleting an active member |

Both require `service_role` privileges and therefore **cannot** run as regular RLS-protected client calls — this is the only place in the codebase the `service_role` key is used.

---

## Realtime

Enabled on: `users`, `matches`, `tournaments`, `supermatches`, `supermatch_games`.

**Gotcha:** channel names must be unique per subscription attempt, or React Strict Mode / Fast Refresh can trigger "cannot add postgres_changes callbacks... after subscribe()" errors. Every `useEffect` that opens a channel appends a random suffix to the channel name for this reason — see any `useXxx.ts` hook with a `supabase.channel(...)` call for the pattern.
