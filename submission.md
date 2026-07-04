# Project 5: Mixtape Bug Hunt — Submission

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
