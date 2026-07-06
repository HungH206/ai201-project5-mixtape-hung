# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I used an AI coding assistant (Claude) throughout this project as a
pair-programming partner, and this section is an honest account of *what* it did,
*what I directed and decided*, and *where I had to verify or correct it*.

**Milestone 1 (codebase map).** I had the AI read the `routes → services → models`
layers and summarize the architecture, the data model, and which service each of
the five issues lived in. I used this to orient quickly, then confirmed the claims
by opening the files myself — e.g. checking that `playlist_entries` really carries
a `position` column and that rating and add-to-playlist both route through
`notification_service`.

**Milestone 2 (reproduction).** I had the AI write small Python harness scripts
that call the service functions directly inside the Flask app context, so I could
observe each bug against the seeded data instead of firing HTTP requests. This is
where the AI was most useful *and* where verifying it mattered most:

- For **Issue #3** ("same song shows up twice in search"), the plausible diagnosis
  was that the `outerjoin(song_tags)` returns one row per tag, so a 3-tag song
  would appear 3 times. When I actually **ran** it, the search returned **1** row,
  not 3 — SQLAlchemy 2.0's `Query.all()` de-duplicates full ORM entities by
  primary key, so the reported symptom does not reproduce in this environment. The
  raw SQL does return 3 rows (the join is genuinely wrong), but the user-visible
  bug doesn't manifest. This is the clearest case where the surface-level
  explanation was incomplete and only *running the code* settled it — so I dropped
  #3 and picked #4 instead.

**Milestone 3 (diagnosis, fixes, RCA).** For each chosen bug I had the AI trace
from the symptom to the suspect function and explain the code, then I read the
code myself to confirm the specific cause before changing anything:

- **Issue #1:** the AI flagged the `and today.weekday() != 6` clause in
  `update_listening_streak`. I verified the diagnosis by confirming Python's
  `datetime.weekday()` maps Monday=0…Sunday=6 (so `== 6` really is Sunday) and by
  reproducing the reset on a Sunday vs. a correct increment on a Monday.
- **Issue #5:** the AI pointed at the `songs[:-1]` slice in the return of
  `get_playlist_songs`. I confirmed by comparing the returned count to a direct
  `COUNT(*)` of `playlist_entries`, and checked the empty-playlist path so the fix
  wouldn't error on `[]`.
- **Issue #4:** the AI diffed `rate_song` against its working sibling
  `add_to_playlist` and identified the missing `create_notification` call. I
  confirmed the fix notifies on a friend's rating but *not* on a self-rating
  (the `shared_by != user_id` guard).

**Verification I owned.** Every fix was checked against the existing `pytest`
suite (**3 failed → 13 passed**) and against both sides of each boundary condition
(same-day / consecutive / skipped-day for #1; non-empty / empty playlist for #5;
friend-rate / self-rate for #4). I did not accept a fix as done until I had run it.

**Bottom line.** The AI accelerated navigation, reproduction scripting, and
drafting the write-ups, but I made the decisions (which bugs to pick, swapping #3
for #4), and the one place a confident-sounding explanation was wrong (#3) was
caught only by running the code myself — which is exactly why reproducing before
fixing was worth the time.

---

## Milestone 1 — Codebase Map

Mixtape is a Flask + SQLAlchemy social music app. The architecture is a clean
three-layer stack: **routes** (HTTP parsing + response formatting) → **services**
(all business logic) → **models** (SQLAlchemy ORM + the SQLite DB). Every route
delegates immediately to a service function; no business logic lives in the route
handlers.

---

### Top-level files

| File | Responsibility |
|------|----------------|
| [app.py](app.py) | Flask **application factory** (`create_app`). Owns the shared `db = SQLAlchemy()` object, reads config from env (`DATABASE_URL`, defaults to `sqlite:///mixtape.db`), registers the four blueprints under `/songs`, `/playlists`, `/users`, `/feed`, and calls `db.create_all()`. |
| [models.py](models.py) | All data models + three association tables. Imports `db` from `app`. |
| [seed_data.py](seed_data.py) | Standalone script (`python seed_data.py`) that drops/recreates the DB and populates 5 users, 25 songs (with 0 / 1 / 3+ tags), 3 playlists, listening events across ~2 weeks, streaks, and one example notification. Comments explicitly flag which rows exercise which issue (e.g. multi-tag songs → Issue #3, older events → Issue #2). |
| [requirements.txt](requirements.txt) | Dependencies (Flask, Flask-SQLAlchemy, pytest). |

---

### The data model ([models.py](models.py))

**7 models** + **3 association tables.**

Core models:
- **User** — `username`, `email`, `listening_streak` (int), `last_listened_at` (nullable datetime). Self-referential many-to-many `friends` relationship via the `friendships` table (`lazy="dynamic"`).
- **Song** — `title`, `artist`, `album`, `genre`, `shared_by` (FK → User), `shared_at`, `share_note`. Has `ratings`, `listening_events`, and `tags` (M2M).
- **Tag** — just `name`. Linked to songs via `song_tags`.
- **ListeningEvent** — a `(user_id, song_id, listened_at)` row. This is the raw event log that both streaks and the feed are computed from.
- **Rating** — `(user_id, song_id, score 1–5, rated_at)` with a **`UniqueConstraint(user_id, song_id)`** — a user can only have one rating row per song. The rating is its own model (not a column on Song).
- **Playlist** — `name`, `created_by`, `is_collaborative`. Songs are attached via the `playlist_entries` association table.
- **Notification** — `(user_id, notification_type, body, created_at, read)`. Generic: the `notification_type` string distinguishes kinds (e.g. `"song_added_to_playlist"`).

Association tables (important detail):
- **`friendships`** — symmetric M2M; seed data inserts **both directions** manually so friendship is bidirectional.
- **`song_tags`** — plain M2M join (song ↔ tag).
- **`playlist_entries`** — the interesting one. It's not a plain join table: it carries a **`position`** column (explicit ordering, not insertion order), plus `added_by` and `added_at`. So "the 3rd song in a playlist" is a real, stored value.

IDs everywhere are string UUIDs (`generate_uuid`). Every model has a `to_dict()` used for JSON responses.

---

### Routes layer ([routes/](routes/))

Thin blueprints. Each handler: parse input → `try` the service call → `jsonify`
the result → translate `ValueError` into a 400/404.

| Blueprint (prefix) | Endpoints | Calls into |
|---|---|---|
| [routes/songs.py](routes/songs.py) (`/songs`) | `GET /search?q=`, `GET /<id>`, `POST /<id>/rate`, `POST /<id>/listen` | `search_service`, `notification_service.rate_song`, `streak_service.record_listening_event` |
| [routes/playlists.py](routes/playlists.py) (`/playlists`) | `POST /`, `GET /<id>`, `GET /<id>/songs`, `POST /<id>/songs` | `playlist_service`, `notification_service.add_to_playlist` |
| [routes/users.py](routes/users.py) (`/users`) | `GET /<id>`, `GET /<id>/streak`, `GET /<id>/notifications`, `POST /notifications/<id>/read` | `streak_service.get_streak`, `notification_service` |
| [routes/feed.py](routes/feed.py) (`/feed`) | `GET /<id>/listening-now`, `GET /<id>/activity` | `feed_service` |

**Cross-cutting note:** the routing doesn't line up 1:1 with services. Rating a
song is a `/songs` route but the logic lives in `notification_service` (because
rating triggers a notification). Adding a song to a playlist is a `/playlists`
route but also lives in `notification_service`. So "which service owns this?" is
answered by *what side effect it produces*, not by the URL prefix.

---

### Services layer ([services/](services/)) — where the bugs live

- **[streak_service.py](services/streak_service.py)** — `record_listening_event` creates a `ListeningEvent` and calls `update_listening_streak(user, now)`, which compares `now.date()` to `last_listened_at.date()`: 0 days → no change, 1 day → increment, else → reset to 1. Also handles tz-naive `last_listened_at` by assuming UTC. `get_streak` just reads `user.listening_streak`. *(Issue #1 lives here.)*

- **[feed_service.py](services/feed_service.py)** — `get_friends_listening_now` filters `ListeningEvent`s from the user's friends within a `RECENT_THRESHOLD` (24h), orders newest-first, and dedupes to one (most recent) song per friend. `get_activity_feed` is the same query without the recency filter, capped by `limit`. *(Issue #2 lives here.)*

- **[search_service.py](services/search_service.py)** — `search_songs` does an `outerjoin` to `song_tags` and filters on `title`/`artist` ILIKE, returning `song.to_dict()` for each result row. `get_song` fetches one by id. *(Issue #3 lives here — note the join to a M2M table.)*

- **[notification_service.py](services/notification_service.py)** — misnamed slightly: it owns both notification CRUD *and* the two actions that generate notifications.
  - `create_notification` — low-level insert.
  - `add_to_playlist` — appends the song to the playlist and, **if the adder ≠ the original sharer**, creates a `song_added_to_playlist` notification for the sharer.
  - `rate_song` — upserts a `Rating` (respects the unique constraint by updating the existing row).
  - `get_notifications` / `mark_as_read`. *(Issue #4 lives here.)*

- **[playlist_service.py](services/playlist_service.py)** — `get_playlist_songs` joins `Song` to `playlist_entries`, filters by playlist, orders by `position` ascending. `create_playlist`, `get_playlist`, `get_user_playlists` are straightforward. *(Issue #5 lives here.)*

---

### Data flow trace — "a friend adds my song to a playlist, and I get notified"

1. Client → `POST /playlists/<playlist_id>/songs` with `{song_id, added_by}`.
2. [routes/playlists.py](routes/playlists.py) `add_song()` validates both fields are present, then calls `add_to_playlist(playlist_id, song_id, added_by)`.
3. [notification_service.py](services/notification_service.py) `add_to_playlist`:
   - loads the `Song`, the adding `User`, and the `Playlist` (each raising `ValueError` if missing → route returns 400);
   - if the song isn't already in `playlist.songs`, appends it and commits (this writes a `playlist_entries` row);
   - compares `song.shared_by` to `added_by_user_id`. If they differ, calls `create_notification(user_id=song.shared_by, type="song_added_to_playlist", body="...")`.
4. `create_notification` inserts a `Notification` row for the **sharer** and commits.
5. Later, the sharer calls `GET /users/<id>/notifications` → `get_notifications` → returns the row.

Contrast — **rating** a song (`POST /songs/<id>/rate` → `rate_song`): it upserts a
`Rating` and commits, but the call chain **does not** invoke `create_notification`.
That asymmetry (add-to-playlist notifies, rate does not) is exactly what Issue #4
describes.

---

### Patterns I noticed

1. **Strict layering.** Routes never touch `db` for business logic (only `users.py` does a trivial `db.session.get` for a profile lookup). All queries and mutations live in services. To debug an endpoint, jump straight to its service.
2. **`ValueError` as the error protocol.** Services raise `ValueError` for "not found" / "bad input"; routes catch it and map to 400/404. There's no custom exception hierarchy.
3. **Commit-per-service-call.** Each service function that mutates state commits its own transaction. There's no unit-of-work spanning multiple service calls.
4. **`to_dict()` is the serialization boundary.** Nothing else formats JSON; models decide their own wire shape.
5. **`playlist_entries.position` is authoritative ordering** — the app cares about explicit song order, so any playlist read must sort by `position`, and any off-by-one in slicing/ordering is visible to users (relevant to Issue #5).
6. **Timezone handling is defensive but inconsistent.** `datetime.now(timezone.utc)` is used for new rows, and streak logic re-attaches UTC to naive datetimes — a sign that datetime edge cases (day boundaries, 24h windows) are where the time-based bugs cluster (#1, #2).
7. **The seed script is a map to the bugs.** Its comments name which fixtures exist to reproduce each issue — a useful oracle when writing tests.

---

## The Five Open Issues (read all before choosing three)

| # | Title | Service |
|---|-------|---------|
| 1 | Listening streak keeps resetting | `streak_service.py` |
| 2 | "Friends Listening Now" shows people from yesterday | `feed_service.py` |
| 3 | Same song shows up twice in search | `search_service.py` |
| 4 | Notified when a friend adds my song to a playlist, but not when they rate it | `notification_service.py` |
| 5 | The last song in a playlist never shows up | `playlist_service.py` |

*(Fixes and their write-ups will go in the milestones below.)*

---

## Milestone 2 — Reproduce Before Fixing

I chose **Issue #1, #4, and #5**. (I originally picked #3, but it does not
reproduce in this environment — see the note at the end — so I swapped it for #4
per the milestone's "try a different one" guidance.)

Reproduction method: I seeded a fresh DB (`python seed_data.py`) and called each
service function directly inside the Flask app context, comparing observed output
against the DB ground truth. No fixes were applied.

### Issue #1 — Listening streak keeps resetting

- **How I reproduced it:** Took `darius` from the seed data (streak = 3,
  `last_listened_at` = yesterday) and called
  `streak_service.update_listening_streak(darius, now)` twice: once with `now`
  set to a **Sunday** (2026-07-05, `weekday()==6`) and once with `now` on a
  **Monday** (2026-07-06). Both should raise the streak from 3 → 4 (he listened
  yesterday, listens again today).
- **Observed:** Sunday → streak resets to **1**; Monday → streak correctly becomes
  **4**. The reset only happens on Sundays.
- **Root cause (located, not yet fixed):** [streak_service.py:73](services/streak_service.py#L73)
  — the increment branch is guarded by `and today.weekday() != 6`, so a valid
  consecutive-day listen on a Sunday falls through to the `else` and resets to 1.

### Issue #5 — The last song in a playlist never shows up

- **How I reproduced it:** Called `playlist_service.get_playlist_songs()` for the
  "Late Night Vibes" playlist and compared the result length to a direct
  `COUNT(*)` of that playlist's `playlist_entries` rows.
- **Observed:** DB has **7** entries; the function returns **6** songs. The
  highest-`position` (last) song is always missing.
- **Root cause (located, not yet fixed):** [playlist_service.py:66](services/playlist_service.py#L66)
  — the return slices the ordered list with `songs[:-1]`, dropping the final element.

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it

- **How I reproduced it:** Took a song shared by `simone` and had `darius`
  interact with it. Counted `simone`'s notifications before/after each action:
  (A) `add_to_playlist(...)`, then (B) `rate_song(darius, song, 5)`.
- **Observed:** Adding to a playlist creates a notification for the sharer
  (count 0 → 1); rating the song creates **none** (count stays 1). Expected a
  notification for both interactions.
- **Root cause (located, not yet fixed):** [notification_service.py:73-110](services/notification_service.py#L73-L110)
  — `rate_song` upserts a `Rating` and commits but never calls
  `create_notification`, unlike `add_to_playlist` at
  [notification_service.py:64-70](services/notification_service.py#L64-L70).

### Issue #3 — could not reproduce (why I switched)

The reported symptom is "the same song shows up twice in search." The underlying
SQL **is** wrong: `search_songs` does `outerjoin(song_tags)` and returns one
result row per join row, so a song with 3 tags produces 3 rows at the SQL level
(I confirmed a raw `query(Song.id).outerjoin(...)` returns **3** rows for a 3-tag
song). **However**, the service returns `db.session.query(Song)...all()`, and
SQLAlchemy 2.0's legacy `Query.all()` de-duplicates full ORM entities by primary
key — so the user-visible search result is **1** row, not 3. The reported
duplicate symptom therefore does not manifest in this environment, so I chose
#4 instead. (The join is still latently incorrect and would surface duplicates if
the query selected columns instead of the full entity.)

---

## Milestone 3 — Root Cause Analysis & Fixes

**Baseline before any fix:** `pytest tests/` → **3 failed, 10 passed**. The three
failures were the target tests for these bugs
(`test_streak_increments_on_sunday`, `test_playlist_returns_all_songs`,
`test_playlist_returns_songs_in_order`). **After all three fixes:** `pytest tests/`
→ **13 passed**.

### Issue #1 — My listening streak keeps resetting

- **How I reproduced it:** Seeded the DB, took `darius` (streak = 3,
  `last_listened_at` = yesterday), and called
  `update_listening_streak(darius, now)` with `now` on a Sunday (2026-07-05) vs a
  Monday (2026-07-06). Sunday reset the streak to **1**; Monday correctly gave
  **4**. The bundled test `test_streak_increments_on_sunday` (Sat → Sun) also
  failed with `1 != 2`.
- **How I found the root cause:** Symptom is in the streak, so I opened
  [streak_service.py](services/streak_service.py). `record_listening_event`
  delegates the streak math to `update_listening_streak`, so I read that function
  line by line. The `days_since_last == 0 / == 1 / else` ladder was correct, but
  the "listened yesterday" branch had an extra clause: `and today.weekday() != 6`.
  That clause has nothing to do with consecutive-day logic — that was the moment I
  was sure this was the specific cause, not just a suspicious area.
- **The root cause:** [streak_service.py:73](services/streak_service.py#L73). Python's
  `datetime.weekday()` returns **6 for Sunday**. The increment branch was written
  as `elif days_since_last == 1 and today.weekday() != 6:`. So whenever "today" is
  a Sunday, a genuine consecutive-day listen (exactly 1 day since the last listen)
  fails the `and`, falls through to the `else`, and resets the streak to 1. Every
  streak that crossed into a Sunday was silently wiped — matching the "keeps
  resetting" report.
- **My fix & side-effect check:** Removed the spurious `and today.weekday() != 6`
  so the branch is just `elif days_since_last == 1:`. Verified both sides of the
  day boundary: same-day listen still no-ops (`days_since_last == 0`), a skipped
  day still resets (`else`), and consecutive days increment on **every** weekday
  including Sunday. Full suite: 13/13 pass (previously-failing Sunday test now
  passes; the other streak tests still pass).
- **AI usage:** After I located the `weekday() != 6` clause myself, I confirmed my
  understanding that `datetime.weekday()` maps Monday=0…Sunday=6 (so `== 6` is
  Sunday), then verified the fix by reading it and running the tests.

### Issue #5 — The last song in a playlist never shows up

- **How I reproduced it:** Called `get_playlist_songs()` on the seeded
  "Late Night Vibes" playlist and compared to a direct `COUNT(*)` of its
  `playlist_entries` rows: **7 in the DB, 6 returned**. The highest-`position`
  song was always missing. Bundled tests `test_playlist_returns_all_songs`
  (expects 5) and `test_playlist_returns_songs_in_order` (expects Track 1–5) both
  failed.
- **How I found the root cause:** Opened [playlist_service.py](services/playlist_service.py)
  and read `get_playlist_songs`. The query itself was correct — join to
  `playlist_entries`, filter by playlist, `order_by(asc(position))`. The tell was
  the very last line: the ordered `songs` list was sliced with `songs[:-1]` in the
  return statement.
- **The root cause:** [playlist_service.py:66](services/playlist_service.py#L66). The
  return was `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice
  drops the final element of the list, and because the list is ordered ascending
  by `position`, the dropped element is always the last (highest-position) song.
  The query fetched all N songs correctly; the slice threw one away on the way out.
- **My fix & side-effect check:** Changed the slice to iterate the full list:
  `return [song.to_dict() for song in songs]`. Boundary check on both ends: a
  non-empty playlist now returns all 7 songs in `position` order (Track 1…5 in the
  test), and an empty playlist still returns `[]` without error
  (`test_empty_playlist_returns_empty_list` passes — `songs` is `[]`, so the
  comprehension yields `[]`). Full suite 13/13.
- **AI usage:** None needed for diagnosis — the `[:-1]` was self-evident once read.
  Used AI only to sanity-check that `list[:-1]` on an empty list is `[]` (so the
  empty-playlist path was unaffected), which I then confirmed by running the test.

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it

- **How I reproduced it:** Took a song shared by `simone`, had `darius` (a
  different user) act on it, and counted `simone`'s notifications after each
  action. `add_to_playlist(...)` → notifications 0 → **1**; `rate_song(darius,
  song, 5)` → stayed at **1** (delta 0). Adding notifies the sharer; rating does
  not.
- **How I found the root cause:** Opened [notification_service.py](services/notification_service.py)
  and compared the two sibling functions directly, since one works and one
  doesn't. `add_to_playlist` ends with an `if song.shared_by != added_by_user_id:`
  guard that calls `create_notification(...)`. `rate_song` upserts the `Rating`,
  commits, and returns — with **no** `create_notification` call anywhere. The
  structural difference between the two blocks was the confirmation.
- **The root cause:** [notification_service.py:73-110](services/notification_service.py#L73-L110).
  `rate_song` never creates a notification. It's not a broken condition — the
  side-effect step that the parallel `add_to_playlist` performs is simply absent
  from the rating path, so the sharer is never told their song was rated.
- **My fix & side-effect check:** After the rating commits, added the same
  sharer-notification pattern used by `add_to_playlist`, guarded by
  `if song.shared_by != user_id:` with type `"song_rated"`. Checks: (a) a friend
  rating your song now creates exactly one notification (delta 1); (b) rating your
  **own** song creates none (verified: 0 → 0), because of the `shared_by != user_id`
  guard; (c) the notification is created only after the rating commit, so a bad
  score (`ValueError` for out-of-range) still short-circuits before any
  notification; (d) the `Rating` unique-constraint upsert path is unchanged, so
  re-rating still updates the existing row. Full suite 13/13 (no test regressed;
  no existing test covered this path).
- **AI usage:** Used AI to diff the two blocks (`add_to_playlist` vs `rate_song`)
  and confirm the working notification pattern I was mirroring; I verified the fix
  by re-running my reproduction script and the self-rating edge case.
