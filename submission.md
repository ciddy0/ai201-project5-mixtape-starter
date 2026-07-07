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
