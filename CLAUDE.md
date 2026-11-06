# HARDCORE — Runbook & Claude Code Handoff

**Project:** HARDCORE — Minecraft hardcore run tracker (one death ends the run for everyone)
**Players:** Kylan, Raiden, Ethan
**Status:** v3 deployed 2026-11-05 to GitHub Pages + Supabase (cloud mode). Verified; test data wiped.
**Repo:** github.com/kylanjack/hardcore
**Live URL:** kylanjack.github.io/hardcore/

---

## 1. Purpose

Single source of truth for the HARDCORE tracker: what's built, how it works, what's left, and the rules any future session (human or Claude Code) must follow when touching it. This document lives in the repo root. If renamed to `CLAUDE.md`, Claude Code reads it automatically on every session — recommended.

## 2. Architecture & Design Decisions

- **Single-file app.** Everything — HTML, CSS, JS, pixel art — lives in `index.html`. No build step, no dependencies beyond two Google Fonts (Press Start 2P, VT323). Do not split into multiple files.
- **Dual-mode storage adapter.** `CONFIG` block at the top of the `<script>`. Both values empty → localStorage mode (key `hardcore_runs_v1`). Both filled → Supabase cloud mode via plain `fetch` against PostgREST (`/rest/v1/hc_runs`) — no supabase-js client, keep it that way. The `store` object (`load` / `insert` / `update`) is the only place persistence logic lives.
- **Supabase:** SAME project as the Soul Link tracker, new isolated table `hc_runs`. Same project URL + publishable anon key as Soul Link. RLS is per-table; the two apps cannot interact. Writes are protected by convention only (public anon key + permissive policies) — accepted risk, same as Soul Link.
- **One-time migration:** on load in cloud mode, if cloud is empty and localStorage has runs, the app offers to push them up (strips legacy `seed` field, defaults `mc_day` / `play_ms` / `session_started_at`).
- **Design system:** MC pixel theme. Press Start 2P (display, class `px`), VT323 (everything else), zero border-radius globally, beveled MC-GUI borders (light top-left / dark bottom-right, inverted on `:active`), palette: bg `#141109`, panels `#2b2115`, green `#72c234`, red `#e23b3b`, gold `#fcdc5f`.
- **All art is inline SVG.** Hearts, mob faces, full-body mobs (enderman/creeper/zombie/skeleton), floating blocks, grass tile (data-URI), moon, stars. Scene layout lives in `buildDecor()` — plain position/size/opacity arrays.

### 2.1 Standing Rules (non-negotiable)

- **NEVER add external icon/font CDNs without fetching the exact URL to verify first.** Preferred answer is always inline SVG. (Rule written in blood on prior projects.)
- `localStorage` only — never `window.storage` (that API only exists in Claude artifact previews, breaks when deployed).
- iOS-safe interactions only: discrete buttons, no click-to-cycle, no nested buttons (avoid needing `stopPropagation` by structure).
- The 1-second ticker updates text nodes in place (`#liveTime`) — it must NEVER call `render()`, or it wipes input focus mid-typing.
- Session state changes (`toggleSession`, death) save immediately via `saveNow()`; rapid-tap changes (milestones, MC day) use `queueSave()` (600ms debounce). Keep that split.
- All user-entered strings pass through `esc()` before hitting `innerHTML`. Any new rendered field must too.
- Filename is `index.html` exactly (GitHub Pages, case-sensitive, Windows hides extensions — watch for `index.html.html`).

## 3. Data Model (`hc_runs` / run object)

| Field | Type | Notes |
|---|---|---|
| id | uuid | client-generated `crypto.randomUUID()` |
| attempt_number | int | sequential, never reused |
| mc_day | int | MANUAL counter (browser can't read the game); sync via `/time query day` |
| play_ms | bigint | accumulated session playtime, ms |
| session_started_at | timestamptz/null | non-null = session live right now |
| started_at / ended_at | timestamptz | wall-clock run bounds |
| status | text | `active` or `dead`; exactly one active run at a time |
| death_player / death_cause / death_note | text | filled on death |
| players | jsonb | `["Kylan","Raiden","Ethan"]` |
| milestones | jsonb | `[{label, done}]`; next attempt inherits labels, resets done |

Full `create table` SQL + RLS policies + `alter table` patch block: top comment of `index.html`. That comment is the schema's source of truth — keep it updated with any schema change.

## 4. As Built (history — do not redo)

- **v1:** core tracker — attempt lifecycle, milestone checklist, death flow (who/cause/last words), graveyard, stats strip, export/import JSON, dual-mode storage, original dark-brutalist theme.
- **v2:** full MC retheme (pixel fonts, bevels, hardcore heart row, XP bars); seed field removed from UI and schema; manual MC DAY counter added; "deepest MC day" replaced wall-clock stat.
- **v3 (current):** session-based playtime (START/END SESSION, live H:MM:SS ticker, tap-to-edit total, 8h forgotten-session warning, death folds live session into total); scenic background (grass ground strip, 15 floating blocks of 8 types, 4 full-body mobs on the grass, moon + stars); footer border removed.

## 5. Deployment Procedure (ACTIVE — next work)

Order matters: Supabase first so the app is cloud-native from commit one, nothing to migrate.

1. Supabase dashboard → Soul Link project → SQL Editor → paste the full `create table hc_runs` block from the `index.html` top comment (incl. `alter table ... enable row level security` and all three `create policy` lines) → Run → expect "Success. No rows returned."
2. Table Editor → confirm `hc_runs` exists with `mc_day`, `play_ms`, `session_started_at` columns.
3. Copy project URL + anon key (identical to the values in the Soul Link HTML's config).
4. New public GitHub repo under `kylanjack` → upload `index.html` to `main` root.
5. Settings → Pages → Deploy from branch → `main` / root.
6. Edit `index.html` → paste URL + key into `CONFIG` → commit.
7. Actions tab → green check. ~10 min cache on changes; "missing" changes are usually cache, not failed deploys.

## 6. Verification (run after deploy, in order)

1. Badge next to "one death ends it all" reads **CLOUD** in green. LOCAL = CONFIG paste failed (stray quote/space).
2. No red error box. `401` = wrong key; policy/`42501` error = a `create policy` line didn't run.
3. Attempt 01 auto-created AND visible as a row in Supabase Table Editor (proves writes, not just reads).
4. Toggle a milestone → hard refresh → still toggled.
5. Open live URL on phone → same state (sync proof).
6. Full death-flow test: start session → end session → set MC day → RUN ENDED → pick player, cause "test" → graveyard renders it, Attempt 02 spawns with milestones reset.
7. Wipe test data: Table Editor → delete all `hc_runs` rows → refresh page → fresh Attempt 01 auto-creates. (This is the clean-slate procedure, reusable anytime.)
8. Mobile visual check: mob spacing at ~390px width (`left` percentages, last four lines of `buildDecor()`), block density in narrow margins (`blocks` array).

## 7. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Badge says LOCAL, expected CLOUD | CONFIG values empty/malformed — check quotes, trailing spaces |
| Red box `Supabase 401` | Wrong anon key |
| Red box mentions policy / `42501` | RLS policy missing — re-run the three `create policy` lines |
| Phone shows empty tracker | App is in LOCAL mode (localStorage is per-device) — expected until CONFIG is filled |
| Change not visible after deploy | GitHub Pages ~10 min cache; check Actions tab for green check first |
| Playtime total inflated | Forgotten session — END SESSION, then tap the total to set correct minutes |
| Checked milestone lost after instant refresh | Debounce window (600ms) — wait a beat before refreshing after rapid taps |

## 8. Open Items

| # | Item | Owner | Status |
|---|---|---|---|
| 1 | Create `hc_runs` table in Soul Link Supabase project (SQL in file comment) | Claude | DONE 2026-11-05 (migration `create_hc_runs`, 3 policies verified) |
| 2 | Create repo, deploy to GitHub Pages, paste CONFIG | Kylan + Claude | DONE 2026-11-05 |
| 3 | Run full verification checklist (§6), then wipe test data | Claude | DONE 2026-11-05 except step 5 (real phone) |
| 4 | Mobile check: mob spacing + block density on real phone | Kylan | 390px emulation OK; real phone pending |
| 5 | PWA install support (manifest + apple-touch-icon) so it installs to iOS home screen like The Backlog | Claude Code | DEFERRED — V2 |
| 6 | Supabase Realtime subscription so open devices update live without refresh | Claude Code | DEFERRED — V2 |
| 7 | Per-player playtime | Kylan decision | DECIDED: no for V1 — shared group timer |
| 8 | "Best progress" stat uses current run's milestone count as denominator; switch to percentage-across-attempts if milestone lists diverge | Claude Code | DEFERRED — only if it bothers anyone |
| 9 | More block types (emerald, lapis) — one line in `blockSVG()` color map + one `blocks` array entry | Claude Code | NICE-TO-HAVE |

Known accepted risks (not open items): public anon key means writes are convention-protected only; concurrent writes are last-write-wins; session timer is one shared group timer, not per-player (see item 7).

## 9. Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| v1 | 2026-11-05 | Claude (chat) | Initial build: tracker core, dual-mode storage |
| v2 | 2026-11-05 | Claude (chat) | MC retheme, seed removed, MC day counter |
| v3 | 2026-11-05 | Claude (chat) | Session playtime, scenic background, footer line removed |
| v3 | 2026-11-05 | Claude (chat) | Deployed: hc_runs table, CONFIG filled, Pages live, §6 verified, test rows wiped |
