# Life Group Planner — Project Handoff

Handoff for continuing work in Claude Code. Written 2026-10-18, updated 2026-11-03 (added one-off events, opened Firestore rules to all collections). Lives in the repo root so every session has full context.

## What this is

A schedule page for a 4-family life group, built as a **single self-contained `index.html`** (vanilla HTML/CSS/JS, no build step, no dependencies beyond the Firebase CDN). Anyone in the group can view and edit it; all shared state lives in Firebase Firestore.

- **Repo:** `noah-austin/Life-Group-Planner` (branch `main`, GitHub Pages from root)
- **Live site:** https://noah-austin.github.io/Life-Group-Planner/
- **Firebase project:** `life-group-planner` (Firestore only; web config is pasted near the top of the `<script>` block in `index.html`)
- **Deploy flow:** edit `index.html` → push to `main` → Pages redeploys automatically.

## Core design principle

The schedule is **computed, not stored**. A rotation engine in the page generates all 52 weeks from rules + seed values; Firestore only stores *deviations* (one-week edits, rotation shifts, view stats). An empty database = the pure default rotation. This means "Reset to default" is always possible per-week, and regenerating or extending the schedule never loses user edits.

## New pattern from Nov 11, 2026 (`NEW_START`)

Decided with the owner 2026-11-05 (second revision the same day). Weeks **before** `NEW_START = "2026-11-11"` still use the original rules below, so history never changes. From Nov 11 on, `computeBase()` reads a **fixed 15-week table, `NEW_TABLE`**, that repeats:

| # | Night | Host | Leads |
|---|---|---|---|
| 1 | All Together | Austin's | Johnson's |
| 2 | Split | Johnson's | Havlin's (Mitchell & Haleigh) |
| 3 | All Together | Havlin's | Austin's |
| 4 | Fun | — | — |
| 5 | Split | Johnson's | Smith's (Pablo & Emmalie) |
| 6 | All Together | Smith's | Johnson's |
| 7 | Split | Johnson's | Austin's (Noah A & Jillian) |
| 8 | All Together | Austin's | Smith's |
| 9 | Fun | — | — |
| 10 | Split | Johnson's | Havlin's |
| 11 | All Together | Havlin's | Johnson's |
| 12 | Split | Johnson's | Austin's |
| 13 | All Together | Smith's | Havlin's |
| 14 | Fun | — | — |
| 15 | Split | Johnson's | Smith's |

Why it's this way (the owner's requirements):
- **Split and together alternate** with fun every 5th week, so nobody goes long without one kind of night. An earlier version flipped the order each cycle and caused near-month gaps, which the owner rejected.
- **The Johnson's host every split night** because they drive far. The Austin's, Havlin's and Smith's take turns hosting together nights.
- **Fun weeks use no hosting or leading turn.** This is the owner's explicit preference, so don't change it.
- **Leading:** one family per week. On split nights the husband leads the men and the wife leads the women (`coupleOf(f)`). Every family leads 3 times per 15 weeks. The Johnson's lead 3 together nights and never a split, since they host every split. Everyone else leads 2 splits and 1 together. Nobody leads a night they host.
- **Spacing:** the leader order was picked by brute force over all 3,600 valid orders (2026-11-06) to maximize the gap between a family's turns: every family has at least two other meetings between its turns. A gap of 3+ for everyone is impossible under these rules. The owner flagged the earlier order because the Smith's led back-to-back meetings. Re-check spacing after any change to the table.
- **Why a table instead of rotation queues:** no simple queue satisfies these rules. Every start order locks some family out of leading one kind of night (verified by brute force).
- New-era default times (`NEW_TIMES` via `defaultTime()`): **split nights 6:00 PM**, **together and fun nights 6:30 PM** (owner's request 2026-11-06).

Other details:
- **Rotation anchors in the new era** (`tableSlotFor`):
  - If the current slot already has the anchored type and host, the table stays put and the anchor's leader applies to that week only.
  - Otherwise the table jumps to the best-matching slot (type, then host, then leader), nearest going forward. Setting a week to the previous week's values slides the whole table back one week.
  - An anchor's own host and leader always show for its week.
- **Edit modal:** shows Split / All Together / Fun for new-era weeks, and the old four types for earlier weeks.
- **Cleared edits:** the one-week edits saved under the old pattern for Nov 11, Nov 25 and Jan 20 were cleared when the first version went live.

## Original rotation rules (before Nov 11, 2026 — don't break these)

- Meetings every **Wednesday**, starting **Aug 12, 2026** (`START = [2026, 7, 12]`, `NUM_WEEKS = 52`).
- 4-week **type cycle**: Men's Night (6:30 PM) → Women's Night (6:30 PM) → All Together (6:00 PM) → Fun Meetup (out somewhere, time TBD).
- (Original) **Hosting** rotates Austin's → Havlin's → Smith's → Johnson's, but **fun weeks don't consume a hosting turn**. Seed: Havlin's host first (`SEED.host = 1`).
- **Leaders** rotate per meeting type:
  - Men: Noah A (Austin), Mitchell (Havlin), Pablo (Smith), Noah J (Johnson) — `SEED.men = 0`
  - Women: Jillian (Austin), Emmalie (Smith), Haleigh (Havlin), Carly (Johnson) — `SEED.women = 3` (Carly first)
  - All Together: led by a family — `SEED.all = 0` (Austin's first)
- **Invariant: nobody hosts and leads the same week.** The engine walks week-by-week keeping leader *queues*; if the next leader's family is hosting, they're skipped to their next turn (queue-defer). The seeds were chosen so the default schedule has zero skips. The edit modal also blocks saving a same-family host/leader combo directly.
- Family names display with apostrophes: "the Austin's, Havlin's, Smith's, Johnson's" (as `&rsquo;` / `’` in code).
- `ADDRESSES` array (parallel to `FAMILIES`: Austin, Havlin, Smith, Johnson order) powers the "📍 Directions" pill on host lines — Apple Maps link on iOS user agents, Google Maps otherwise.

## Firestore data model

| Collection | Doc ID | Purpose |
|---|---|---|
| `schedule` | `YYYY-MM-DD` | One-week overrides. Full set (`type, host, lead, time, note, cancelled`) when saved as "only this week"; partial (note/cancelled/custom values only) when saved alongside a rotation shift. Fields merge on top of the computed base in `merged()`. |
| `rotation` | `YYYY-MM-DD` | Rotation **anchors** ("shift from here on"): `{type, hostIdx (0–3), leadName (men/women) or leadFam (all)}`. The engine re-anchors its cycles when it reaches an anchored date. Deleting the doc un-shifts. |
| `events` | auto ID | One-off events (any day): `{title, date (YYYY-MM-DD), time, place, note}` — only `title` and `date` are required. Rendered inline in the schedule by date; same-day items sort meeting first, then events by time. Independent of the rotation engine. |
| `stats` | `total`, `week-<upcoming-Wed ISO>` | View counter: `views` (increment) and `devices` (arrayUnion of a localStorage id `lg-id`) per meeting week. Shown in the footer. |

Current rules (published 2026-10-19) — open read/write on **any** top-level collection, so new features that add a collection need no rules change:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{collection}/{doc} { allow read, write: if true; }
  }
}
```

The previous per-collection version is in the Firebase console's rules history (Aug 8, 2026). Rules are edited at console.firebase.google.com → Life Group Planner → Firestore → Rules → Publish; Claude can't change them (no admin credentials), but it **can** read and write the data itself via the Firestore REST API (`https://firestore.googleapis.com/v1/projects/life-group-planner/databases/(default)/documents/<collection>`), since the rules are open. Read freely to check real data; don't write test data without the owner's OK beyond a create-then-delete check. **When editing a doc via REST, always send every field** — a PATCH with `updateMask` through this sandbox's proxy replaced the whole doc and wiped its other fields (happened once on 2026-11-06; restored immediately). Read the doc first, then PATCH the full field set.

## UI features (all in index.html)

- Two tabs: **Schedule** and **Prayer Chain** (segmented control under the header; no emoji on the tab — owner's preference).
- Schedule list grouped by month; "Next up" badge on the coming Wednesday; auto-scrolls there on load; past weeks hidden with a "Show past weeks" toggle (fully-past month headers hide too).
- **Edit mode** (✎ button — intentionally **no password**; it was removed on purpose): tap any week → modal with meeting-type select, host dropdown (4 families / "Out somewhere!" / Other… free text), leader dropdown (per-type lists / None / Other…), time, note, cancelled checkbox, and a scope choice: **"Only this week"** (writes `schedule/{date}`) vs **"Shift the rotation from here on"** (writes `rotation/{date}` anchor + any extras). "Reset to default" deletes both docs.
- Markers: orange dot = one-week edit; ⟳ = rotation shift starts here; "Cancelled" strikes through.
- **One-off events**: in edit mode a purple "+ Add event" button appears; tapping an event opens its editor with Delete (confirm prompt). Events show a purple left stripe, weekday in the date chip, and a 📍 Directions pill when `place` is set (a family name like "Austin's" maps to that family's address via `placeAddress()`). Past events hide with past weeks; "Next up" stays on meetings only.
- **Men's Prayer Chain** tab: each man calls/prays for one other; offsets 1→2→3 over a 3-week cycle (week 2 is mutual pairs), so each man covers all three others, then repeats. **Flips Wednesdays at 5:00 PM local** (`firstFlip = START date at 17:00`) so new assignments can be announced at that night's meeting. Pure date math — nothing stored.
- Footer view counter: "N views · N this week (N people)" — "week" is meeting-to-meeting, uniqueness is per-device.
- Firebase-down / not-configured states degrade gracefully to the computed default schedule with a banner.

## Gotchas & honest caveats

- **Everything is world-writable** (by design, for a 4-family group). The repo is public (required for free GitHub Pages), so the home addresses in `ADDRESSES` are publicly visible — the owner accepted this, but flag it if hardening ever comes up. The upgrade path discussed: Firebase Auth (Google sign-in for the 8 adults) + tightened rules.
- Schedule horizon: `NUM_WEEKS = 52` from Aug 12, 2026 → runs out **Aug 2027**. Extending is just bumping `NUM_WEEKS` (it's computed); Firestore edits keyed by date carry over.
- `merged()` override semantics are subtle: a one-week *type* change resets lead/time defaults unless explicitly overridden. Read it before touching.
- The rotation engine is sequential simulation — anchors apply *at* their date as the walk passes them. Multiple anchors over time all apply in date order.
- ES5-style JS throughout (var, no arrows) — keep it; the page targets any phone browser.
- Changing MEN/WOMEN/FAMILIES/SEED alters the *entire* computed schedule including the past; permanent people changes should ideally be done via a rotation anchor or with care about history.

## Deploy notes

- **Bump `APP_VERSION` in `index.html` on every deploy.** Phones (especially the home-screen web app, which has no refresh button) keep a cached copy; GitHub Pages serves `max-age=600`. On load, when the app comes back to the foreground, and every 30 min, the page fetches itself with `cache: no-store`. If the live `APP_VERSION` differs, it reloads via `?v=<version>`, a fresh URL, so the CDN and browser caches are bypassed. It tries once per version per session so it can't loop, and it skips the check while editing. The footer's ↻ Refresh link forces the same reload. Forgetting the bump means phones only update when their cache expires.
- Earlier history is web-UI uploads ("Delete index.html" / "Add files via upload"). Since 2026-10-19 changes are committed and pushed to `main` from Claude Code; Pages redeploys in a minute or two.

## Testing approach used so far

Headless Playwright (Chromium) against `file://` with:
- `addInitScript` Date-class override to simulate specific dates/times (used to verify the 5 PM prayer flip boundary, past-week hiding, 3-week cycle repeat);
- a stubbed `db`/`firebase` object to exercise save paths and the host=lead guard without network;
- invariant check: walk `computeBase()` and assert zero host/lead family collisions across all 52 weeks after any engine change. **Re-run this check after touching rotation logic.**

Note: the current cloud sandbox **can** reach the Firestore REST API (but not gstatic or the github.io site), so a good check is: pull real `schedule`/`rotation`/`events` docs via REST and seed the stubbed `db` with them.

## Feature backlog (discussed, consciously deferred)

~~One-off events~~ (shipped 2026-10-19), RSVP headcount, snack signup, prayer requests board, fun-week idea voting, add-to-calendar (.ics), PWA/home-screen app feel, 2-year schedule extension, women's prayer chain (5-minute add: mirror `renderPrayerChain` with the WOMEN array). Owner's stated preference: keep it simple, resist scope creep, ship small.
