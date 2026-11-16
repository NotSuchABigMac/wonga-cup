# Golf Weekend Platform: Detailed Plan

Status: draft for review. Written from a review of this repo at commit
`9573e17`. This file lives here only until the new platform repo exists,
then it moves there as the founding document.

Working name used below: **the platform**. The product name, domain and
repo name are open (see section 13).

## 1. Goal and non-goals

**Goal.** Any organiser can sign up, describe their golf weekend (players,
courses, rounds, formats, teams, branding), and get a hosted event site
with a live scorecard, TV board and commentary feed, without touching
code. Going live with real scoring is paid.

**Decision already made.** Evolve the existing code in a new repo. This
repo stays as the live Wonga Cup site until the platform can host the
Wonga Cup as an ordinary tenant.

**Non-goals for v1.**
- Native mobile apps. The no-install web app is a selling point.
- Club or society competition management (handicap submission, tee sheets,
  memberships). That market belongs to Golf Genius and similar.
- More than two teams, or individual-only events. Planned for v2, and the
  data model must not block them.
- Player accounts. Scorers keep joining with a name and an event PIN.
- Reproducing every Wonga Cup flourish (the song, flamingo shower,
  hand-written Wraps) for every tenant.

## 2. Product shape

**Niche for launch.** Team-versus-team golf trips, Ryder Cup style: two
teams, one to four rounds, a mix of match play, scramble and Stableford,
8 to 32 players. This is exactly what the existing code does well, and
it is a clear message for marketing.

**What a customer gets.**
1. An event site at `<domain>/e/<slug>/` with home, players, format,
   practical info and results pages generated from their config.
2. The live scorecard (offline tolerant, multi-device).
3. TV/clubhouse board, live commentary feed, worm charts, QR join codes,
   printable paper backups, practice mode.
4. An organiser dashboard: setup wizard, admin tools (locks, handicap
   overrides, field history, restore, rollback), end-of-event archive.

**Free versus paid.**
- Free: create an event, run the full wizard, use practice mode with a
  capped amount of sandbox data, preview every page.
- Paid, once per event: go live with real scoring, share links, TV board,
  permanent read-only archive after the event.

## 3. Review findings that drive the plan

| Area | Today | Consequence |
|---|---|---|
| Sync | Append-only `tournament_updates` log, replay via `applyUpdateToState()`, offline queue, rollback marker, field history. Segregated by `tournament_id` (practice mode proves it). | Keep the design. Generalise the row coordinates. |
| Scoring | `scoring.js` is pure and tested, but has roughly 300 references to `day1`/`day2`/`day3`, fixed groups `a4/a3/b4/b3`, fixed nearest-the-pin holes, Golf Australia handicap constants. | Largest piece of work: a format engine. |
| Data | `players.js` (14 players), `courses.js` (3 courses), dates and Melbourne timezone as constants. | Becomes per-event config stored in the database. |
| Security | One global scorer PIN and one admin PIN in `app_secrets`, one global rate limit, admins matched by a hardcoded name list (`ADMIN_NAMES`). | Unsafe across customers. Needs per-event secrets and real organiser accounts. |
| Front end | `scorecard-live.html` is 420KB of inline HTML, CSS and JS. Six hand-written content pages. | Split into modules. Content pages become templates. |
| Ops | GitHub Pages, one domain, migrations pasted by hand into the SQL editor, no server code, no payments. | New hosting, automated migrations, Edge Functions for Stripe. |
| Tests | Fast Node suite in CI plus about 55 Playwright repro scripts. A real event's full update log is saved in `snapshots/tournament-updates.json`. | The snapshot becomes a golden regression test for the whole refactor. |

## 4. Target architecture

**Principles carried over.** Static front end, Supabase as the only
backend, event-sourced log, pure scoring logic testable in Node, security
enforced in the database and never in the client.

**Changes.**

| Concern | Choice | Why |
|---|---|---|
| Front end | Plain ES modules, no framework. A `package.json` for dev tooling only (Supabase CLI, Playwright, a linter). Add a bundler only if load time demands it. | Keeps the current code portable. Avoids a rewrite. |
| Hosting | Cloudflare Pages. | Free tier, path rewrites (`/e/* -> /event/index.html`), custom domains later. GitHub Pages cannot do rewrites. |
| Routing | Path based: `/e/<slug>/...`. Subdomains and custom domains are a later paid add-on. | Simplest thing that works with one TLS certificate and one service worker scope per event. |
| Database | New Supabase project, two environments (staging, production). Migrations in `supabase/migrations/` applied by the Supabase CLI from CI. | Ends hand-run SQL. |
| Organiser auth | Supabase Auth, email magic link. | No passwords to manage. Replaces the admin PIN and `ADMIN_NAMES`. |
| Scorer auth | Name plus per-event PIN, as today. | Zero friction on the first tee. |
| Server code | Supabase Edge Functions: Stripe checkout session, Stripe webhook, optional course search proxy. | Only place secrets live. |
| Payments | Stripe Checkout, one-off payment per event. | No card handling on our side. |
| Storage | Supabase Storage for logos and hero images. | Per-event branding. |
| Monitoring | Existing error beacon, keyed per event, with an organiser-independent view for us. | Already built. |

## 5. Data model

All tables in `public` have RLS enabled. Secrets and checks live in a
non-exposed `internal` schema, same pattern as migrations 003 and 005.

### 5.1 Tables

**`events`**

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| slug | text unique | URL segment, validated, reserved words blocked |
| owner_id | uuid | references `auth.users` |
| name | text | |
| status | text | `draft`, `live`, `archived` |
| timezone | text | IANA name, replaces the hardcoded Melbourne logic |
| starts_on / ends_on | date | drives countdown, live and frozen phases |
| config | jsonb | see 5.2 |
| config_version | int | incremented on every save |
| paid_at | timestamptz | null until Stripe confirms |
| created_at | timestamptz | |

Anyone can read a non-draft event's public config by slug. Only the owner
and co-admins can read drafts or write.

**`event_admins`**: `event_id`, `user_id`, `role`. Lets an organiser add
co-organisers. Replaces `ADMIN_NAMES`.

**`event_secrets`**: `event_id`, `scorer_pin_hash`, `sandbox_pin_hash`.
Hashed with `pgcrypto`. No grants to `anon` or `authenticated`. Read only
by `internal.check_event_pin(event_id, pin)`.

**`event_updates`** (successor to `tournament_updates`)

| Column | Type | Notes |
|---|---|---|
| id | uuid PK | |
| event_id | uuid | |
| sandbox | boolean | true for practice mode rows, replaces the `wonga-demo` id trick |
| round_idx | int null | which round, null for event-wide updates |
| update_type | text | namespaced by format, e.g. `matchplay.hole` |
| entity_key | text null | match index, group code or player id, replaces `match_idx` and `player_id` |
| field_key | text null | |
| value | text | |
| updated_by | text | |
| write_token | text | PIN, checked by RLS on insert, never selectable |
| updated_at | timestamptz | server default |

Insert policy: PIN matches, and either `sandbox = true` (within the
sandbox row cap) or the event is `live`. No update or delete for `anon`.
Rollback stays an RPC, now authorised by the caller being an event admin
instead of a second PIN.

**`pin_attempts`**: per-event failure counter for the rate-limited
`validate_event_pin(slug, pin)` RPC. Fixes the current global lockout,
where one attacker could lock out every tournament.

**`courses`**: shared scorecard library. `id`, `name`, `location`,
`country`, `tees` (jsonb: per tee, rating, slope, 18 holes of par, stroke
index, distance), `created_by`, `verified`. Organisers enter a card once
and every later event at that course can reuse it. Events copy the chosen
tee into their own config so later edits to the library never change a
finished event.

**`payments`**: `event_id`, `stripe_session_id`, `amount`, `currency`,
`status`, `created_at`. Written only by the webhook function.

**`client_errors`**: as today, plus `event_id`.

### 5.2 Event config document

Stored as one versioned JSON document. Roster edits that matter mid-event
(handicap overrides, team moves, locks) stay in the update log as they do
today, so they keep history and restore.

```json
{
  "schema": 1,
  "branding": {
    "title": "The Wonga Cup",
    "tagline": "Three courses, three days",
    "theme": "classic-green",
    "logo": "storage://...",
    "teams": [
      { "key": "A", "name": "Flamingos", "colour": "#e85d9b", "emoji": "🦩" },
      { "key": "B", "name": "Gorillas", "colour": "#3a3a3a", "emoji": "🦍" }
    ]
  },
  "handicap": { "system": "GA", "allowance": 0.93 },
  "players": [
    { "id": 0, "name": "Gary King", "short": "G. King", "index": 8.0, "gender": "M" }
  ],
  "rounds": [
    {
      "idx": 1,
      "date": "2026-08-07",
      "course": { "name": "Murray", "tee": "Blue", "rating": 72.3, "slope": 128, "holes": [] },
      "format": "matchplay.singles",
      "options": { "pointsPer": "nine", "matches": 6 },
      "sideGames": [{ "type": "ntp", "holes": [8, 17], "points": 0 }],
      "modifiers": []
    },
    {
      "idx": 2,
      "date": "2026-08-08",
      "format": "scramble.ambrose",
      "options": { "groupsPerTeam": 2 },
      "modifiers": [{ "type": "anthem", "sang": -1, "didNot": 2 }]
    },
    { "idx": 3, "date": "2026-08-09", "format": "stableford.individual", "options": {} }
  ],
  "tiebreak": "putt-off",
  "content": { "practical": "markdown...", "houseRules": "markdown..." }
}
```

A JSON Schema for this document lives in the repo and is validated in the
wizard, in a database check function, and in tests.

## 6. Format engine

This replaces the `day1`/`day2`/`day3` structure. It is the largest and
riskiest piece, so it is done as a strangler refactor with the real
Wonga Cup log as the safety net.

### 6.1 Format module interface

Each format is one pure module (no DOM) plus one UI module.

```
format = {
  id: 'matchplay.singles',
  initialState(round, config)        -> round state
  applyUpdate(state, row, ctx)       -> mutates round state (replay)
  descriptors                        -> update types, labels, addressing, restorable flags
  teamPoints(state, ctx)             -> { A, B, detail }
  projected(state, ctx)              -> projected points while in progress
  isComplete(state, ctx)             -> boolean
  describeEvent(row, before, after)  -> commentary headline or null
  eventStillHolds(row, state)        -> boolean
}
ui = { renderEntry(el, state, ctx), renderBoard(el, state, ctx), renderPrint(...) }
```

`ctx` carries players with overrides applied, the round's course, team
membership and handicap settings. Event-level updates (team names, team
membership, handicap overrides, locks, tiebreak, washout, rollback) stay
in a small core reducer outside any format.

### 6.2 Formats

| Order | Format | Source |
|---|---|---|
| Launch | Singles match play, points per nine or per match, optional two-opponent challenge | existing Day 1 |
| Launch | Ambrose scramble, variable group sizes and group count | existing Day 2 |
| Launch | Individual net Stableford, team total by points per hole or raw total | existing Day 3 |
| Next | Fourball better ball match play | new, most requested trip format |
| Next | Foursomes (alternate shot) match play | new |
| Later | Net stroke play, skins, individual-only events, more than two teams | new |

Side games (nearest the pin, long drive) and modifiers (the anthem rule)
are config-driven add-ons to any round. Handicap conversion becomes a
strategy: `GA` (existing formula), `WHS` (course handicap with format
allowances), or `raw`.

### 6.3 Refactor sequence

1. **Golden test first.** Replay `snapshots/tournament-updates.json`
   through today's `scoring.js` and record every derived number (match
   results, group scores, Stableford table, team totals, worm points,
   commentary headlines). This fixture must stay byte-identical through
   every step below.
2. Parameterise constants: dates, timezone, nearest-the-pin holes, group
   codes, player count, read from a config object shaped like 5.2. Ship a
   `wonga-cup-2026.json` config that reproduces current behaviour.
3. Extract the three formats behind the interface, one at a time, with
   the old `day1`/`day2`/`day3` functions becoming thin wrappers.
4. Introduce the generic `event_updates` row shape with a translation
   layer from the old shape, so the saved log still replays.
5. Generalise: variable match count, group sizes, team sizes, round count.
   Add property-style tests (random valid configs must replay without
   throwing and totals must equal the sum of round points).
6. Delete the wrappers once nothing calls them.

## 7. Front end restructure

```
/                      marketing site (static)
/dashboard/            organiser app: sign in, event list, wizard, billing
/e/<slug>/             event home (templated)
/e/<slug>/players      roster
/e/<slug>/format       generated from rounds config plus house rules text
/e/<slug>/info         organiser's practical info (markdown)
/e/<slug>/score        live scorecard
/e/<slug>/tv           TV board
/e/<slug>/print        paper backups
```

Source layout in the new repo:

```
src/core/        sync (insert, load, queue, resync, rollback), state, config loader
src/formats/     one folder per format: logic.js, ui.js, tests
src/scoring/     handicaps, strokes allocation, shared maths
src/ui/          shell, tabs, modals, toasts, commentary feed, worm charts, theme
src/admin/       locks, overrides, field history, restore, rollback
src/pages/       event site templates, dashboard, wizard
supabase/        migrations, functions (stripe-checkout, stripe-webhook), seed
test/            unit, golden, Playwright
```

`scorecard-live.html` is split by moving code, not rewriting it: first
the inline script into module files with no behaviour change (Playwright
repro scripts must still pass), then the day tabs become a loop over
`config.rounds` rendering each format's UI module.

**Theming.** `theme.js` already switches themes. Extend it to set CSS
variables from `config.branding` (team colours, accent, logo). Ship four
to six presets. No free-form CSS for tenants.

**Tenant-specific extras.** The song, flamingo shower and hand-written
Wraps become either (a) dropped from the platform, or (b) a generic
"daily recap" generated from the commentary events with a template, with
an AI-written version as a later paid add-on. The Wonga Cup can keep its
custom pieces through a per-event `extras` hook that only your event uses.

## 8. Organiser experience

**Wizard steps.**
1. Basics: name, dates, timezone, slug.
2. Players: paste a list or add rows (name, handicap index). Import from
   a previous event.
3. Teams: name, colour, emoji. Assign now, or run the draft live later.
4. Rounds: for each day pick a course (search the library or enter the
   card), tee, format and options, side games.
5. Look: theme preset, logo, hero image, tagline.
6. Content: practical info and house rules in markdown.
7. Review: full preview in practice mode, then pay and go live.

**After go-live.** The dashboard shows the scorer PIN, QR codes, share
links, TV link, and the admin tools. Config edits that would invalidate
entered scores (removing a round, changing a course) are blocked or
warned once scoring has started, following the same "allow but warn"
rule the handicap override already uses.

**After the event.** Status moves to `archived`: read-only, results and
recap stay online, "run it again next year" clones the config.

## 9. Monetisation

**Model.** One-off payment per event. Groups use this once or twice a
year, so a subscription would churn and annoy.

**Starting hypothesis, to be tested in the beta, not a researched price.**

| Tier | Price | Includes |
|---|---|---|
| Free | $0 | Build and preview, practice mode, capped sandbox data |
| Event | about A$59 | Go live, up to 24 players, archive for 12 months |
| Event Plus | about A$99 | Up to 60 players, permanent archive, custom recap |
| Add-ons | later | Custom domain, AI daily recap, photo wall |

**Mechanics.**
1. Dashboard calls the `stripe-checkout` Edge Function with the event id.
2. Function verifies the caller owns the event and creates a Checkout
   Session with the event id in metadata.
3. `stripe-webhook` verifies the signature, writes `payments`, sets
   `events.paid_at` and `status = 'live'`.
4. RLS on `event_updates` only accepts non-sandbox inserts for live
   events, so payment is enforced in the database and cannot be bypassed
   from the client.

Refund policy, tax handling (Stripe Tax) and promo codes for beta users
are part of this milestone.

## 10. Milestones

Sizes are relative (S, M, L, XL). Each milestone ends with the exit
criteria met and CI green.

### M0. Validate and bootstrap (S)
- You: talk to 3 to 5 other organisers, note their formats and what they
  would pay. Check competitor pricing.
- You: create the empty GitHub repo, a new Supabase project (staging),
  a Cloudflare account. Pick a working name.
- Me: seed the new repo from this one with history preserved, set up CI
  (unit tests, Playwright smoke run), Supabase CLI migrations, Cloudflare
  Pages deploy from `main`, `CLAUDE.md`, `README.md`, this plan.
- Exit: the new repo deploys a working copy of the current site to a
  staging URL against the staging database.

### M1. Golden test and config extraction (M)
- Golden replay fixture from the saved log (6.3 step 1).
- Wonga config file, constants parameterised (6.3 step 2).
- Inline script of `scorecard-live.html` moved into modules.
- Exit: no hardcoded player, course, date or timezone in logic files.
  Golden test and all Playwright repro scripts pass.

### M2. Multi-tenant core (L)
- Migrations for `events`, `event_admins`, `event_secrets`,
  `event_updates`, `pin_attempts`, `client_errors`, with RLS tests that
  run against a local Supabase in CI (tenant A cannot read drafts of,
  write to, or roll back tenant B).
- Config loaded from the database by slug. Path routing `/e/<slug>/`.
- Storage keys, service worker cache and session keys namespaced by slug.
- Organiser sign-in. Admin tools authorised by account, admin PIN retired.
- Sandbox flag replaces the demo tournament id.
- Exit: two events with different rosters run side by side on staging
  with no cross-talk, proven by an automated two-tenant Playwright test.

### M3. Format engine (XL)
- Steps 3 to 6 of section 6.3.
- TV board, commentary feed, worm charts, print cards and ribbon driven
  by rounds config.
- Exit: an event with a different shape (for example 2 rounds, 8 players,
  scramble then singles) runs end to end on staging. Golden test still
  identical for the Wonga config.

### M4. Self-service (L)
- Dashboard, wizard, config validation, course library, theme presets,
  logo upload, templated event pages, clone-from-previous.
- Exit: a person who has never seen the code creates and previews an
  event in under 20 minutes without help.

### M5. Payments and go-live gate (M)
- Stripe Checkout and webhook functions, `payments` table, database
  enforced live gate, sandbox caps, receipts, refund path.
- Exit: test-mode purchase flips an event live. Unpaid events provably
  cannot write real scores (RLS test).

### M6. Launch readiness (M)
- Privacy policy, terms, cookie and data notes, account and event
  deletion, data export.
- Backups: Supabase point-in-time recovery or scheduled dumps, replacing
  the per-tournament snapshot workflow. Restore runbook.
- Marketing site, onboarding emails, support inbox, status of errors per
  event for us.
- Load check: 20 simultaneous events of 24 scorers polling every 30
  seconds. Consider Supabase Realtime to replace polling if cost or
  latency demands it.
- Exit: beta events from M0 contacts run for real, then pricing is
  switched on.

### M7. After launch
- Fourball and foursomes formats, more than two teams, individual events,
  custom domains, AI recap, migrating the Wonga Cup itself onto the
  platform and retiring this repo's scorecard.

**Order and dependency.** M0, M1, M2, M3 are strictly sequential. M4 and
M5 can overlap once M2 is done. The first point at which a stranger could
use the product is the end of M4. The first point at which money can be
taken is the end of M5.

## 11. Testing strategy

- **Golden replay**: the real 2026 log, asserted after every refactor.
- **Unit**: per format module, handicap strategies, config schema.
- **Database**: RLS and RPC tests against local Supabase in CI. Every
  policy gets an "other tenant cannot" test.
- **Browser**: existing repro scripts kept and parameterised by slug, a
  two-tenant isolation test, wizard and payment happy paths. A smoke
  subset runs in CI, the full set before release.
- **Docs**: keep the docs-consistency test so README and architecture
  notes do not rot.

## 12. Risks

| Risk | Mitigation |
|---|---|
| Small, seasonal market with free alternatives | M0 validation before M2. Sell the personality (site, TV, commentary), not just scoring. |
| Format engine takes far longer than expected | Golden test, strangler refactor, launch with three formats only. |
| A scoring bug during someone's paid weekend | Practice mode on the real config, paper backups, field history and restore, error beacon, a feature freeze rule during peak weekends. |
| Tenant data leak | Database enforced isolation with tests, no secrets in client, per-event PINs, security review before launch. |
| Course data entry is tedious | Shared course library grows with use. Licensed course database only if demand justifies the cost. |
| Offline devices with stale config or code | Config version check alongside the existing version check and refresh prompt. |
| Support load from non-technical organisers | Wizard validation, preview, a short guide, sensible defaults. |
| Player personal data (names, handicaps) | Privacy policy, deletion and export, option to hide an event from search engines. |

## 13. What I need from you

**To start M0 (blocking).**
1. Create an empty GitHub repo for the platform and give this Claude
   GitHub App access to it. I cannot create repositories from here.
2. A working name (it can change).

**Soon after.**
3. A new Supabase project for staging (keep the Wonga Cup project as is).
4. A Cloudflare account for Pages.
5. A Stripe account (needed at M5).
6. A domain (needed before the beta).

**Decisions, with my default if you have no preference.**

| Decision | Default |
|---|---|
| Launch niche | Two-team trips only |
| URL scheme | `/e/<slug>/` paths |
| Scorer auth | Name plus event PIN, no player accounts |
| Organiser auth | Email magic link |
| Pricing model | One-off per event, numbers tested in beta |
| Handicap systems at launch | Golf Australia and raw, WHS in M7 unless a beta group needs it |
| Wonga-specific extras | Kept for your event only via an extras hook |
| Zero-dependency rule | Relaxed to "no runtime dependencies, dev tooling allowed" |
| Repo history | Preserved in the new repo |
