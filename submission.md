# Project 5 — Mixtape Bug Hunt — Submission

## AI Usage
*(Written last — see Milestone 4. This section will describe specifically how AI tools were
used for codebase navigation and debugging, what they helped explain or trace, and where I
verified or corrected their output.)*

---

## Codebase Map
*(Written during orientation, before any bug work.)*

Mixtape is a **Flask JSON API** backed by SQLAlchemy over SQLite. It has no HTML frontend —
every endpoint returns JSON. The codebase is organized in three clean layers, and the same
pattern repeats everywhere: **a route parses the request and formats the response; a service
holds all the business logic; models define the data.**

### Main files and their roles

- **`app.py`** — The application factory (`create_app`). Creates the Flask app, configures
  the SQLite database (`mixtape.db`), initializes the shared `db = SQLAlchemy()` object, and
  registers four blueprints under URL prefixes: `/songs`, `/playlists`, `/users`, `/feed`.
  It calls `db.create_all()` inside an app context on startup. (Run via
  `FLASK_APP=app:create_app flask run`, *not* `python app.py`, to avoid a double-import of
  the models.)

- **`models.py`** — Defines all data. Entities: **User, Tag, Song, ListeningEvent, Rating,
  Playlist, Notification**. Three association tables model many-to-many relationships:
  - `friendships` — symmetric user↔user friendships (inserted in both directions on seed).
  - `song_tags` — song↔tag.
  - `playlist_entries` — playlist↔song, but with **extra columns**: `position` (explicit
    ordering — songs have a defined position, not just insertion order), `added_by`, and
    `added_at`.
  Most models have a `to_dict()` used by the routes to serialize responses.

- **`routes/`** — One blueprint per area. Routes do request parsing + JSON formatting only,
  then immediately delegate to a service:
  - `songs.py` — `/songs/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`.
  - `playlists.py` — create playlist, get playlist, get playlist songs, add song.
  - `users.py` — get user, get streak, get/read notifications.
  - `feed.py` — `/feed/<user_id>/listening-now` and `/feed/<user_id>/activity`.

- **`services/`** — Where all logic (and all the bugs) live:
  - `streak_service.py` — records listening events and updates a user's consecutive-day
    listening streak.
  - `feed_service.py` — "Friends Listening Now" (recent, time-windowed) and the general
    activity feed (most-recent-N, unfiltered).
  - `search_service.py` — song search by title/artist, returning tags with each song.
  - `notification_service.py` — creates/retrieves notifications; also owns `rate_song` and
    `add_to_playlist` (both of which can generate notifications).
  - `playlist_service.py` — playlist creation and ordered song retrieval.

- **`seed_data.py`** — Drops and recreates the DB, then inserts 5 users (with friendships),
  25 songs (with 0, 1, and 3+ tags), 3 playlists, listening events (some within the last
  30 min, some days old), pre-set streaks, and a sample playlist-add notification.

- **`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py` (pytest).

### Data flow trace — rating a song

`POST /songs/<song_id>/rate` (body: `user_id`, `score`)
→ `routes/songs.py::rate()` parses `user_id`/`score`, casts score to int, and calls
→ `services/notification_service.rate_song(user_id, song_id, score)`, which validates the
score is 1–5, loads the `Song` and rating `User`, checks for an existing `Rating`
(unique per `(user_id, song_id)`) and either updates its score or inserts a new `Rating`,
commits, and returns the `Rating`
→ back in the route, the `Rating.to_dict()` is returned as JSON with HTTP 201.

Notably, rating lives in `notification_service.py` alongside `add_to_playlist` — the module
is where "friend interacts with a song" actions are centralized, since those actions are the
ones that *should* notify the song's original sharer.

### Data flow trace — friends listening now

`GET /feed/<user_id>/listening-now`
→ `routes/feed.py::listening_now()` calls
→ `services/feed_service.get_friends_listening_now(user_id)`, which loads the user, computes
a `cutoff` time (`now - RECENT_THRESHOLD`), collects the user's friend IDs, queries
`ListeningEvent`s from those friends since the cutoff (newest first), deduplicates so each
friend appears once (their most recent song), and returns a list of
`{friend, song, listened_at}` dicts
→ the route wraps it as `{"feed": [...], "count": N}`.

### Patterns I noticed

- **Strict route→service delegation.** Routes never touch the DB directly (except the tiny
  `GET /users/<id>` lookup); business logic is entirely in `services/`. To debug an endpoint,
  trace to its service.
- **Services return plain dicts** (via `to_dict()`), not model objects, so serialization
  concerns stay out of the routes.
- **UUID string primary keys** everywhere (`generate_uuid`).
- **Timezone-aware UTC timestamps** (`datetime.now(timezone.utc)`); some code defensively
  re-attaches `tzinfo` when reading values back from SQLite (which stores naive datetimes).
- **Explicit ordering via `playlist_entries.position`** — playlists are position-ordered, not
  insertion-ordered.

---

## Root Cause Analysis

Bugs fixed (required 3): **#5 playlist**, **#1 streak**, **#4 notification**.

> Note on **Issue #3 (search duplicates):** I chose not to count this as one of my three,
> because it cannot be reproduced in this environment. The query does contain a real join
> fan-out (`outerjoin(song_tags)` returns one row per tag — verified: 3 raw rows for a
> 3-tag song), but `db.session.query(Song)` returns *mapped entities*, and SQLAlchemy 2.0's
> `Query` de-duplicates entities by primary key via the identity map before `.all()` returns.
> So the duplicate rows collapse to one `Song` and never reach the caller; all three
> `test_search_no_duplicates_*` tests pass. Reproducing before fixing saved me from "fixing"
> an unobservable bug. (See stretch section if I write it up formally.)

### Issue #5 — The last song in a playlist never shows up
- **How I reproduced it:** Ran `get_playlist_songs()` against the seeded playlists (each has
  7 entries) in a `flask` shell / script. It returned **6** songs, always dropping the last
  one. The pytest suite confirms it: `test_playlist_returns_all_songs` fails (`5 != 4`) and
  `test_playlist_returns_songs_in_order` fails (missing `"Track 5"`).
- **How I found the root cause:** Traced from the route `GET /playlists/<id>/songs`
  (`routes/playlists.py::get_songs`) to `playlist_service.get_playlist_songs`. I read the
  function top-down: the query joins `playlist_entries`, filters by `playlist_id`, and orders
  by `position` — all correct, and the count of rows fetched was right (7). The discrepancy
  had to be *after* the query, so I looked at the single remaining line — the return
  statement — and saw the slice `songs[:-1]`. That was the moment it clicked.
- **The root cause:** The return statement was
  `return [song.to_dict() for song in songs[:-1]]`. The slice `songs[:-1]` means "all
  elements except the last," so after correctly fetching every song in `position` order, the
  function discarded the final (highest-position) song before returning. This is why the
  *last* song specifically was always missing, and why an N-song playlist returned N−1. It
  also directly contradicts the function's own docstring ("returns all songs in the
  playlist").
- **Fix and side-effect check:** Changed the slice to iterate the full list:
  `[song.to_dict() for song in songs]`. Side-effect check: I re-ran `tests/test_playlists.py`
  — all 3 pass, including `test_empty_playlist_returns_empty_list`. That empty-playlist case
  is the boundary I specifically worried about: with the old `[:-1]`, a *one-song* playlist
  returned an empty list and an *empty* playlist returned empty only by accident; my fix
  makes both correct (1 → 1, 0 → 0) without changing ordering.

### Issue #1 — My listening streak keeps resetting
- **How I reproduced it:** Called `update_listening_streak(user, now)` with controlled dates:
  a user who listened Saturday (streak 5) then listens Sunday. Expected streak 6 (consecutive
  day); got **1**. A control run Monday→Tuesday correctly gave 6, isolating the failure to
  Sunday. `test_streak_increments_on_sunday` fails (`1 != 2`).
- **How I found the root cause:** Started at `POST /songs/<id>/listen`
  (`routes/songs.py::listen`) → `streak_service.record_listening_event` →
  `update_listening_streak`. I read the branch logic against the documented streak rules in
  the docstring. Three branches: same day (no change), consecutive day (increment), otherwise
  (reset). The consecutive-day branch had an extra clause — `and today.weekday() != 6` — that
  the documented rules never mention. I confirmed with a quick check that `weekday()` returns
  6 for Sunday, which matched the "only on Sundays" symptom exactly.
- **The root cause:** The increment branch read
  `elif days_since_last == 1 and today.weekday() != 6:`. In Python, `datetime.weekday()`
  returns 0 for Monday through 6 for Sunday. So whenever the current listen falls on a Sunday,
  `today.weekday() != 6` is `False`, the consecutive-day branch is skipped, and execution
  falls into the `else`, which resets `listening_streak` to 1 — even though the user listened
  the day before (a genuine consecutive day). The bug is dormant Monday–Saturday and only
  manifests when the new listening day is a Sunday. Correct behavior requires that *any*
  one-day gap increments the streak regardless of weekday; the weekday check has no basis in
  the streak rules and was the entire cause.
- **Fix and side-effect check:** Removed the spurious clause so the branch is simply
  `elif days_since_last == 1:`. Side-effect check: I re-ran `tests/test_streaks.py` — all 5
  pass. I specifically verified the *other* boundaries the streak logic depends on weren't
  disturbed: new user starts at 1, same-day listening doesn't double-count
  (`days_since_last == 0` branch, untouched), and a skipped day still resets
  (`days_since_last >= 2` → `else`). Only the Sunday consecutive-day case changed behavior.

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it
- **How I reproduced it:** In a shell, recorded nova's notification count (1), then had a
  friend (darius) rate a song nova had shared via `rate_song(darius, nova_song, 5)`. nova's
  notification count stayed at **1** — no notification was created for the rating, even though
  the playlist-add path does create one.
- **How I found the root cause:** The hint said this was architectural, not a typo, so instead
  of hunting for a broken line I compared the two sibling functions in
  `notification_service.py` that both represent "a friend interacted with your shared song":
  `add_to_playlist` and `rate_song`. Reading them side by side, `add_to_playlist` ends with a
  `create_notification(...)` call guarded by `if song.shared_by != added_by_user_id`, while
  `rate_song` ends at `db.session.commit(); return rating`. I also grepped the codebase for a
  `song_rated` notification type and found none — confirming the notification wasn't broken,
  it was never written.
- **The root cause:** `rate_song` persists the rating correctly but is **missing the
  notification step entirely**. Its sibling `add_to_playlist` notifies the song's original
  sharer; `rate_song` never calls `create_notification`, and no `song_rated` notification type
  exists anywhere. So structurally, one of the two "interaction" paths simply omits the
  behavior the feature requires — the sharer is notified on playlist-adds but never on
  ratings. This is architectural: the fix isn't correcting a wrong value, it's adding a
  missing step to bring `rate_song` in line with the established notification pattern.
- **Fix and side-effect check:** After the existing `db.session.commit()`, I added a
  `create_notification(user_id=song.shared_by, notification_type="song_rated", body=...)`
  call, guarded by `if song.shared_by != user_id` so users aren't notified about rating their
  own songs — mirroring `add_to_playlist`'s self-check. Side-effect checks: (1) a friend
  rating a shared song now increments the sharer's notification count by exactly one, with the
  right body/type; (2) a user rating their **own** song produces no notification (self-check
  works); (3) `rate_song` still returns the `Rating` object the route serializes with
  `to_dict()` — verified the return path and score are unchanged; (4) the full `pytest` suite
  (13 tests) still passes.
