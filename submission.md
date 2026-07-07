# Mixtape — Codebase Map

Mixtape is a social music app (Flask + SQLAlchemy, SQLite by default). Friends share
songs, rate them, build collaborative playlists, and get notified about activity on
their shared songs. There is no auth layer — every request passes a `user_id` in the
body or URL, so the app trusts the caller to say who they are.

## Architecture at a glance

The app is a three-layer stack with a strict, consistent split of responsibility:

```
HTTP request
   │
   ▼
routes/*.py      ← parse JSON/query args, validate presence, format JSON responses,
   │               translate service ValueErrors into 400/404
   ▼
services/*.py    ← ALL business logic + DB reads/writes/commits
   │
   ▼
models.py        ← SQLAlchemy models + association tables
   │
   ▼
app.db (SQLite)
```

Routes never touch `db` for logic (the one exception: `routes/users.py` does a trivial `db.session.get(User, ...)` lookup directly). Services never touch `request` or `jsonify`.

---

## Main files

### `app.py` - application factory

`create_app(config=None)` builds the Flask app, configures the SQLite URI (overridable via `DATABASE_URL`), initializes the shared `db = SQLAlchemy()` object, registers the 4 blueprints under URL prefixes, and calls `db.create_all()`. The `db` instance defined here is imported by `models.py` and every service.

Blueprint → prefix map:

- `songs_bp` → `/songs`
- `playlists_bp` → `/playlists`
- `users_bp` → `/users`
- `feed_bp` → `/feed`

### `models.py` — data model

Defines **6 models** and **3 association tables**, all using string UUID primary keys
(`generate_uuid()`).

Models:

- **User**: `username`, `email`, plus streak state (`listening_streak`,
  `last_listened_at`). Has a self-referential many-to-many `friends` relationship via the `friendships` table.
- **Tag**: just an `id` + unique `name`; attached to songs via `song_tags`.
- **Song**: `title`, `artist`, `album`, `genre`, and crucially `shared_by` (FK to the
  User who introduced the song) + `share_note`. `shared_by` is what drives notifications.
- **ListeningEvent**: one row per listen (`user_id`, `song_id`, `listened_at`). This is the raw event stream that both the streak logic and the feeds are built on.
- **Rating**: `user_id`, `song_id`, `score` (1–5). A `UniqueConstraint(user_id, song_id)` enforces one rating per user per song, so the service upserts rather than inserting duplicates. Ratings are their own table, not a column on Song.
- **Notification**: `user_id` (recipient), `notification_type` string, `body` text, `read` boolean.

Association tables:

- **`friendships`**: symmetric use ↔ user (stored as `user_id`/`friend_id` pairs).
- **`song_tags`**: song ↔ tag many-to-many.
- **`playlist_entries`**: the interesting one: playlist↔song, but it carries extra
  columns `position`, `added_by`, and `added_at`. Playlists expose `songs` via this secondary table.

### `routes/` — HTTP layer (4 blueprints)

- **`songs.py`**: `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`. Note that "rate" routes into `notification_service`, and "listen" routes into `streak_service`.
- **`playlists.py`**: `POST /playlists/`, `GET /playlists/<id>`,
  `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs` (add a song, which lives in
  `notification_service.add_to_playlist`).
- **`users.py`** — `GET /users/<id>`, `GET /users/<id>/streak`,
  `GET /users/<id>/notifications?unread_only=`, `POST /users/notifications/<id>/read`.
- **`feed.py`** — `GET /feed/<user_id>/listening-now`, `GET /feed/<user_id>/activity`.

### `services/` — business logic (where the bugs live)

- **`streak_service.py`** — `record_listening_event()` writes a ListeningEvent and calls
  `update_listening_streak()`, which increments/resets the streak by comparing calendar
  dates. `get_streak()` reads the cached value off the User.
- **`feed_service.py`** — `get_friends_listening_now()` (events within a 24h
  `RECENT_THRESHOLD`, deduped to one song per friend) and `get_activity_feed()` (most
  recent N events, no recency filter).
- **`search_service.py`** — `search_songs()` (case-insensitive title/artist match with a
  join onto tags) and `get_song()`.
- **`notification_service.py`** — `create_notification()` (the shared writer),
  `add_to_playlist()` (adds song + notifies the sharer), `rate_song()` (upserts a
  Rating), plus `get_notifications()` / `mark_as_read()`.
- **`playlist_service.py`** — `create_playlist()`, `get_playlist_songs()` (ordered by
  `position`), `get_playlist()`, `get_user_playlists()`.

### Support files

- **`seed_data.py`** — populates the DB with test users/songs/etc.
- **`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`.
- **`requirements.txt`**, **`.gitignore`**, **`instance/mixtape.db`** (the live SQLite file).

---

## Data flow: adding a shared song to a playlist → notification

This is the clearest end-to-end flow and the one the notification system is built around.

1. **Request** — `POST /playlists/<playlist_id>/songs` with JSON `{song_id, added_by}`.
2. **Route** (`routes/playlists.py:add_song`) — pulls `song_id` and `added_by` from the
   body, 400s if either is missing, then calls
   `notification_service.add_to_playlist(playlist_id, song_id, added_by)`. It does no
   logic itself.
3. **Service** (`notification_service.add_to_playlist`):
   - Loads the Song, the adding User, and the Playlist; raises `ValueError` (→ 400) if
     any is missing.
   - Appends the song to `playlist.songs` (through the `playlist_entries` table) if not
     already present, and commits.
   - **The notification trigger:** if `song.shared_by != added_by_user_id` — i.e. someone
     _other than the original sharer_ added it — it calls `create_notification()` for
     `song.shared_by` with type `"song_added_to_playlist"` and a human-readable body.
     Adding your own song notifies nobody.
4. **Writer** (`create_notification`) — constructs a `Notification` row for the recipient
   and commits.
5. **Response** — service returns `None`; route returns `201 {"message": "Song added to
playlist"}`.

The recipient later reads it via `GET /users/<id>/notifications`
(`notification_service.get_notifications`, newest-first).

**The key design fact:** notifications are always addressed to `song.shared_by` (the
song's original introducer), not to the playlist owner. The song's provenance — who
shared it — is the anchor for all social feedback.

---

## Patterns worth noting

- **Route → service delegation is near-total.** Routes do exactly three things: extract
  input, catch `ValueError`, format JSON. Every real decision lives in a service. This is
  consistent enough that the README explicitly says to trace bugs from route into service.
- **`ValueError` is the universal error channel.** Services raise `ValueError` for
  not-found / bad-input; routes catch it and choose 400 vs 404 by context. There are no
  custom exception classes.
- **UUID string PKs everywhere**, generated app-side via `generate_uuid()`, not DB
  autoincrement.
- **Events are the source of truth; derived state is cached.** `ListeningEvent` rows are
  the raw log. The streak is _not_ recomputed from events on read — it's stored on the
  User and mutated at write time in `update_listening_streak`. The feeds, by contrast,
  are computed fresh from events on every read.
- **One shared `db` handle** defined in `app.py` and imported everywhere, which is why
  the app factory has to construct it before models/services import it.
- **Ordering is explicit, not implicit.** Playlists rely on the `position` column in
  `playlist_entries` rather than insertion order — the join table carries metadata, it
  isn't a bare many-to-many.

### Suspicious spots I noticed while mapping (this is a bug-hunt repo)

Reading the services against their own docstrings, several behaviors don't match the
stated contract — consistent with the five open issues in the README:

- **`playlist_service.get_playlist_songs`** returns `songs[:-1]`, which silently drops the
  last song even though the docstring says "returns all songs." (Issue 5)
- **`notification_service.rate_song`** upserts the Rating but never calls
  `create_notification`, so rating a song notifies no one — unlike `add_to_playlist`.
  (Issue 4)
- **`search_service.search_songs`** outer-joins `song_tags` without a `DISTINCT`, so a
  song with N tags comes back N times. (Issue 3)
- **`streak_service.update_listening_streak`** has a `today.weekday() != 6` clause that
  resets an otherwise-valid streak on Sundays. (Issue 1)
- **`feed_service`** uses a rolling 24h window (`RECENT_THRESHOLD`) for "listening now,"
  which includes events from yesterday rather than a "today" boundary. (Issue 2)

Each of these lives in the `services/` layer, reinforcing the README's guidance that the
routes are thin and the logic (and the bugs) are one layer down.

---

# Root Cause Analysis

## Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Following Kenji's report, I traced the scenario in the streak
unit tests. The existing test `test_streak_increments_on_sunday` in
`tests/test_streaks.py` already models it exactly: call `update_listening_streak` with a
Saturday datetime (`2024-06-15`, `weekday() == 5`), then again with the next-day Sunday
datetime (`2024-06-16`, `weekday() == 6`). The streak should read 2 (two consecutive
days). Before the fix, running `pytest tests/test_streaks.py` showed this test failing —
the streak came back as 1 instead of 2, confirming the Sunday reset Kenji described.

**How I found the root cause:** I started from the endpoint Kenji hit. The listen action
is `POST /songs/<id>/listen` in `routes/songs.py`, which delegates (per the codebase's
route→service pattern) to `streak_service.record_listening_event()`. That function writes
the `ListeningEvent` and then calls `update_listening_streak(user, now)` in
`services/streak_service.py`. Reading that function's branch logic against its own
docstring (lines 46–50, which say a streak increments on consecutive calendar days and
only resets when a day is skipped) made the culprit obvious: line 73 carried an extra
`and today.weekday() != 6` condition that the documented rules never mention.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday. Line 73 read
`elif days_since_last == 1 and today.weekday() != 6:`. When a user listened on a Sunday
one calendar day after Saturday, `days_since_last == 1` was true (a valid consecutive
day), but `today.weekday() != 6` evaluated to false because Sunday *is* weekday 6. That
made the whole `elif` false, so control fell through to the `else` branch, which sets
`user.listening_streak = 1`. The result: any streak update landing on a Sunday was
treated as a skipped day and reset to 1, regardless of how long the real streak was.

**My fix and side-effect check:** I removed the spurious `and today.weekday() != 6`
clause, leaving `elif days_since_last == 1:`. This restores the documented rule —
increment on exactly one day's gap, reset only when more than one day is skipped. I ran
`pytest tests/test_streaks.py -v`: all 5 tests pass, including
`test_streak_increments_on_sunday`. I specifically confirmed
`test_streak_resets_after_skipped_day` still passes, verifying that a genuine skipped day
(Monday → Wednesday, `days_since_last == 2`) still correctly resets to 1 — the reset path
is unaffected, only the false Sunday reset is gone.

## Issue #2 — "Friends Listening Now" shows people from yesterday

**How I reproduced it:** Following nova's report, I seeded the DB (`python seed_data.py`)
and called `get_friends_listening_now(nova_id)` inside an app context. To reproduce the
exact symptom of an evening listen lingering into the next morning, I gave a friend
(simone) a single ListeningEvent timestamped 30 minutes before today's UTC midnight. That
event was still only about 21 hours old. Before the fix she appeared in the feed even
though her only listen was the previous day, because a rolling 24-hour window still counted
it as under 24 hours old. This matches nova seeing darius's 11pm listen the next morning.

**How I found the root cause:** I traced from the endpoint nova hit,
`GET /feed/<user_id>/listening-now` in `routes/feed.py`, which delegates straight to
`feed_service.get_friends_listening_now()` following the route-to-service pattern. Reading
that function in `services/feed_service.py`, the cutoff computation stood out immediately.
It subtracted a `RECENT_THRESHOLD = timedelta(hours=24)` constant from the current time.
The "hangs around until the same time next day" phrasing in the report is the exact
signature of a sliding window anchored to the current instant rather than to midnight,
which confirmed this line was the cause rather than just a suspicious area.

**The root cause:** The cutoff was `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`
with `RECENT_THRESHOLD = timedelta(hours=24)`, and the query filtered
`ListeningEvent.listened_at >= cutoff`. This is a rolling 24-hour window measured backward
from the current moment rather than a calendar-day boundary. An event at 11pm yesterday
stays above the cutoff until 11pm today, so "listened now" or "listened today" effectively
meant "listened in the last 24 hours." A friend whose last listen was yesterday evening
kept showing until the same clock time the next day.

**My fix and side-effect check:** In `services/feed_service.py` I replaced the rolling
cutoff with today's UTC midnight, using
`cutoff = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)`,
and removed the now-unused `RECENT_THRESHOLD` constant along with the now-unused
`timedelta` import. UTC midnight is the right anchor because every timestamp in the app is
stored and compared in UTC (`models.py`) and there is no per-user timezone. I kept the `>=`
comparison so an event exactly at 00:00:00 today counts as today while 23:59:59 yesterday
does not, and I verified both sides of that boundary directly. I confirmed the three seeded
friends with minutes-ago listens still appear, and that a friend whose only listen was
before today's midnight is now excluded. For side effects, `RECENT_THRESHOLD` is referenced
nowhere else (confirmed by grep), and `get_activity_feed()` in the same file applies no
recency filter, so it is unchanged. I confirmed it still returns all 8 seeded events
regardless of age. The full test suite's only failures are the pre-existing, still-open
Issue #5 playlist tests, which are unrelated to this change.

## Issue #3: The same song keeps showing up twice in search

**How I reproduced it:** Following simone's report, I worked from the existing
`tests/test_search.py`, whose seed fixture deliberately creates one song with no tags
(*Midnight Drive*), one with a single tag (*Block Party*), and one with three tags
(*Crown Heights Anthem* by Borough Kings, the exact song simone named). Running
`pytest tests/test_search.py` before the fix, `test_search_no_duplicates_multi_tag_song`
failed: searching "Crown Heights" returned the three-tag song three times, while the
zero- and one-tag songs each returned once. That reproduced simone's "some appear once,
others two or three times, for a single-song match" symptom and showed the duplicate
count tracked a song's tag count.

**How I found the root cause:** I traced from the endpoint simone hit,
`GET /songs/search?q=` in `routes/songs.py`, which delegates (per the route→service
pattern) to `search_service.search_songs()`. Reading that function in
`services/search_service.py`, the query selected only `Song` but chained an
`.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. The moment I connected "duplicate
count equals tag count" to a join against the many-to-many `song_tags` association table, I
was confident this was the cause rather than a suspicious area: a left join fans out one row
per association row, which is exactly a 3-tag → 3-row mapping.

**The root cause:** `search_songs()` issued
`db.session.query(Song).outerjoin(song_tags, Song.id == song_tags.c.song_id)`. A SQL
`LEFT OUTER JOIN` produces one result row for every matching row on the joined side, so a
song with N `song_tags` rows came back N times; a song with one tag came back once, and a
song with zero tags came back once (the NULL side of the outer join). Because the query
selected only the `Song` entity and never used the joined `song_tags` columns, the filter
matches on `Song.title`/`Song.artist`, and each song's tags are loaded independently through
the `tags = db.relationship("Tag", secondary=song_tags, lazy="subquery")` relationship used
by `Song.to_dict()`, the join contributed nothing but row duplication.

**My fix and side-effect check:** I removed the spurious `.outerjoin(...)` line from
`search_songs()` (and dropped the now-unused `song_tags` import). This eliminates the
row fan-out at its source rather than masking it with `.distinct()`, and returns the
query to plainly selecting songs whose title or artist matches. I ran
`pytest tests/test_search.py -v`: all 5 tests pass, including the multi-tag,
single-tag, and no-tag no-duplicate cases (both sides of the "how many tags" boundary)
and `test_search_returns_matching_songs`, confirming matching songs are still returned
and each appears exactly once. Tag data is unaffected because `Song.to_dict()` still
reads `tags` from the relationship. `get_song()` in the same file never used the join, so
it is unchanged. The full suite's only failures remain the pre-existing, still-open Issue
#5 playlist tests, unrelated to this change.

## Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Following aaliya's report, I exercised both notification paths
against an in-memory DB inside an app context. I created a sharer and a separate friend,
had the friend rate a song the sharer had shared (`rate_song(friend_id, song_id, 5)`),
then read the sharer's notifications with `get_notifications(sharer_id)`. Before the fix
the list came back empty even though the `Rating` row was saved, exactly aaliya's symptom:
the rating persists (shows on the song) but no notification is ever created. As the control,
the playlist-add path (`add_to_playlist`) did produce a notification, confirming the gap was
specific to rating.

**How I found the root cause:** I traced from `POST /songs/<song_id>/rate` in
`routes/songs.py`, which delegates to `notification_service.rate_song()`. The tell was that
both the working and broken behaviors live in the *same file*, `notification_service.py`, so
I compared them line-by-line. `add_to_playlist()` ends with an explicit block (lines 64-70):
after committing, `if song.shared_by != added_by_user_id: create_notification(...)`.
`rate_song()` had the same shape, validate score, load `song` and `rater`, upsert the
`Rating`, `db.session.commit()`, but then simply `return rating`. It never called
`create_notification` at all. This isn't a typo or a wrong comparison; the entire
notification step that the playlist path has was structurally absent from the rating path,
which is why "ratings notifications just don't happen, for anyone."

**The root cause:** `rate_song()` performed only the persistence half of the operation. It
saved (or updated) the `Rating` and returned, with no call to `create_notification`. The
notification side of the "friend interacts with your shared song → notify the sharer"
contract, present and correct in `add_to_playlist`, was missing entirely from
`rate_song`, so no `Notification` row was ever written for a rating and nothing appeared in
`GET /users/<id>/notifications`.

**My fix and side-effect check:** I added the missing notification block to `rate_song()`,
mirroring `add_to_playlist` exactly: after the commit, `if song.shared_by != user_id:` call
`create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score} stars.")`.
I used the type string `"song_rated"`, the value `create_notification`'s own docstring
names as the rating example, and reused the existing shared `create_notification` writer
rather than duplicating insert logic. The `song.shared_by != user_id` guard matches the
playlist path's rule that adding/rating *your own* song notifies nobody. I verified end to
end: a friend's rating now creates exactly one `song_rated` notification addressed to the
sharer with the expected body, and a self-rating creates none (notification count stays put).
For side effects, the rating upsert itself is untouched, the notification runs strictly
after the existing commit, so a repeat rating still updates the same `Rating` row (unique
`(user_id, song_id)` constraint) and doesn't disturb `add_to_playlist`, which shares the
`create_notification` writer. The full suite's only failures remain the pre-existing,
still-open Issue #5 playlist tests, unrelated to this change.

## Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Following darius's report, the two existing playlist tests in
`tests/test_playlists.py` already model it exactly. `test_playlist_returns_all_songs` seeds
a five-song playlist and asserts all five titles come back; `test_playlist_returns_songs_in_order`
asserts the full ordered list. Before the fix, `pytest tests/test_playlists.py` showed both
failing, `get_playlist_songs` returned only `["Track 1"..."Track 4"]`, dropping `Track 5`,
the last (most recently added) song. That reproduced darius's "always exactly one missing,
always the newest" symptom, and the "adding another frees the previous one" behavior follows
directly: whichever song currently sits at the highest position is the one hidden.

**How I found the root cause:** I traced from `GET /playlists/<playlist_id>/songs` in
`routes/playlists.py`, which delegates to `playlist_service.get_playlist_songs()`. That
function builds the correct query, join `playlist_entries`, filter by playlist, order by
`asc(position)`, so the ordering and count were right up to the last line. The return
statement, `return [song.to_dict() for song in songs[:-1]]`, was the moment it clicked: the
`[:-1]` slice discards the final element of an already-correctly-ordered list, and the
function's own docstring says "This function returns all songs in the playlist." The mismatch
between the slice and the documented contract pinpointed the exact cause.

**The root cause:** `get_playlist_songs` correctly queried and ordered every playlist song
ascending by `playlist_entries.position`, but its return statement sliced the result with
`songs[:-1]`, which drops the last element of a list. Because the list is sorted by ascending
position and each newly added song gets the highest position, the dropped element was always
the most recently added song. The playlist's stored count (7) reflected all entries while the
returned list (6) was one short, always missing the newest, exactly the reported behavior.

**My fix and side-effect check:** I changed the return to iterate the full result,
`return [song.to_dict() for song in songs]`, removing the `[:-1]` slice so every song in the
playlist is returned in position order as the docstring promises. This is a one-token change
that leaves the query, ordering, and not-found handling untouched. I ran the full suite:
`pytest` now reports 13 passed, 0 failed, both `test_playlist_returns_all_songs` and
`test_playlist_returns_songs_in_order` pass, confirming all songs return in the correct order.
I verified the boundary on both sides: a populated playlist returns its complete, ordered set
including the newest song, and an empty playlist yields `[]` (slicing was the only thing that
could have masked an empty list, and `[]` iterates to `[]` cleanly). No other function reads
through `get_playlist_songs`, and `add_to_playlist` (which appends entries) is unaffected, so
nothing else depends on the old truncating behavior.
