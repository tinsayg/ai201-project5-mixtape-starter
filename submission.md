# Project 5 Submission — Mixtape Bug Hunt

## Codebase Map

### Main Files and What Each One Does

**`app.py`**
Flask application factory. `create_app(config=None)` initializes SQLAlchemy with a SQLite database (default: `mixtape.db`), then registers four blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`). This is the only place the DB is initialized — every other file imports `db` from here.

**`models.py`**
Defines all SQLAlchemy models and three association tables:

- `friendships` — symmetric many-to-many on User (A→B and B→A stored as separate rows)
- `song_tags` — many-to-many between Song and Tag
- `playlist_entries` — many-to-many between Playlist and Song, but with extra columns: `position` (int, not null) for explicit ordering, `added_by` (FK to User), and `added_at` (timestamp). This is not a simple join table — it carries ordering metadata.

Core models:

| Model | Key fields | Notes |
|-------|-----------|-------|
| User | username, email, listening_streak, last_listened_at | streak + timestamp updated by streak_service |
| Song | title, artist, album, genre, shared_by, share_note | shared_by is an FK to User |
| Tag | name (unique) | genres like "rap", "indie" |
| ListeningEvent | user_id, song_id, listened_at | one row per listen |
| Rating | user_id, song_id, score (1–5) | unique constraint on (user_id, song_id) — one rating per user per song |
| Playlist | name, created_by, is_collaborative | songs relationship goes through playlist_entries |
| Notification | user_id, notification_type, body, read | recipient-centric; created by notification_service |

Every model has a `to_dict()` method used for JSON serialization in routes.

**`routes/songs.py`** — 4 endpoints under `/songs`:
- `GET /songs/search?q=` → `search_service.search_songs()`
- `GET /songs/<id>` → `search_service.get_song()`
- `POST /songs/<id>/rate` → `notification_service.rate_song()`
- `POST /songs/<id>/listen` → `streak_service.record_listening_event()`

**`routes/playlists.py`** — 4 endpoints under `/playlists`:
- `POST /playlists/` → `playlist_service.create_playlist()`
- `GET /playlists/<id>` → `playlist_service.get_playlist()`
- `GET /playlists/<id>/songs` → `playlist_service.get_playlist_songs()`
- `POST /playlists/<id>/songs` → `notification_service.add_to_playlist()`

**`routes/users.py`** — 4 endpoints under `/users`:
- `GET /users/<id>` → direct DB query for User
- `GET /users/<id>/streak` → `streak_service.get_streak()`
- `GET /users/<id>/notifications` → `notification_service.get_notifications()`
- `POST /users/notifications/<id>/read` → `notification_service.mark_as_read()`

**`routes/feed.py`** — 2 endpoints under `/feed`:
- `GET /feed/<id>/listening-now` → `feed_service.get_friends_listening_now()`
- `GET /feed/<id>/activity` → `feed_service.get_activity_feed()`

**`services/notification_service.py`**
Owns notifications end-to-end. `create_notification()` is a generic factory the other service functions call. `add_to_playlist()` both inserts the song into the playlist *and* fires a "song_added_to_playlist" notification to the song's original sharer. `rate_song()` creates or updates a Rating row. `get_notifications()` and `mark_as_read()` handle retrieval and read state.

**`services/streak_service.py`**
Manages the listening streak on User. `record_listening_event()` creates the ListeningEvent and delegates to `update_listening_streak()`. The streak rule: first listen → 1; same calendar day → no change; yesterday → increment; any gap → reset to 1.

**`services/feed_service.py`**
`get_friends_listening_now()` queries ListeningEvents for all friends within a recent time window, deduplicates to one entry per friend (most recent song), and returns them newest-first. `get_activity_feed()` returns the most recent N events from all friends with no time window.

**`services/search_service.py`**
`search_songs()` does a case-insensitive title/artist match using `ilike`. It outer-joins the `song_tags` table to include tag data. `get_song()` is a simple PK lookup.

**`services/playlist_service.py`**
`get_playlist_songs()` joins Song through `playlist_entries` and sorts ascending by `position`. The other functions handle playlist creation, metadata retrieval, and user playlist listing.

**`seed_data.py`**
Creates 5 users, establishes friendships between them, inserts 25 songs with varying tag counts, creates 3 collaborative playlists, and populates listening events spanning two weeks. Run once to get a usable local database.

**`tests/`** — Three test files, each using an in-memory SQLite database:
- `test_streaks.py` — tests the streak increment/reset rules including a Sunday-specific case
- `test_search.py` — tests that search returns correct matches and no duplicates for songs with 0, 1, or 3 tags
- `test_playlists.py` — tests that all songs in a playlist are returned in position order

---

### Data Flow: Rating a Song Triggers a Notification

1. Client sends `POST /songs/<song_id>/rate` with body `{"user_id": "...", "score": 4}`.
2. `routes/songs.py:rate()` extracts `user_id` and `score`, validates they are present, then calls `notification_service.rate_song(user_id, song_id, int(score))`.
3. `rate_song()` in `notification_service.py`:
   - Validates score is 1–5.
   - Looks up the Song (raises ValueError if not found).
   - Looks up the rater User (raises ValueError if not found).
   - Queries Rating for an existing `(user_id, song_id)` pair — because of the unique constraint, a user can only have one rating per song.
   - If a rating exists: updates its `score`. If not: creates a new Rating row and adds it to the session.
   - Commits, returns the Rating instance.
4. The route serializes the rating with `.to_dict()` and returns HTTP 201.
5. (After Bug 4 fix) `rate_song()` also calls `create_notification()` targeting `song.shared_by` with type `"song_rated"` and a body naming the rater and score — so the original sharer gets notified.

---

### Data Flow: Adding a Song to a Playlist

1. Client sends `POST /playlists/<playlist_id>/songs` with body `{"song_id": "...", "added_by": "..."}`.
2. `routes/playlists.py:add_song()` validates both fields are present, then calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. `add_to_playlist()`:
   - Validates song, adder User, and playlist all exist.
   - Checks if the song is already in `playlist.songs`; if not, appends and commits.
   - If `song.shared_by != added_by_user_id`, calls `create_notification()` for the song's original sharer.
4. `create_notification()` inserts a Notification row for the sharer with type `"song_added_to_playlist"`.
5. Route returns HTTP 201 `{"message": "Song added to playlist"}`.

---

### Architectural Patterns

**Routes delegate immediately to services.** Every route function does exactly two things: validate/extract the HTTP input and call one service function. All business logic — DB queries, constraint checking, side effects like notifications — lives in the service layer. Routes handle input parsing and response formatting only.

**Notifications are a side effect of service actions, not a separate call.** When `add_to_playlist()` runs, it both modifies the data *and* fires the notification in one transaction. This is the intended pattern, but it requires every service function that should notify to explicitly call `create_notification()`.

**UUIDs as primary keys everywhere.** All PKs are `String(36)` UUID strings generated at insert time via `generate_uuid()`. There are no auto-increment integer PKs.

**One `to_dict()` per model.** Serialization lives on the model, not in the routes or services. Routes call `.to_dict()` directly on returned instances.

---

## Bug Fixes

### Bug 1 — Streak Resets on Sunday

**File:** `services/streak_service.py`, line 73

**Root Cause:** The streak increment branch has an extra guard: `elif days_since_last == 1 and today.weekday() != 6`. `weekday() == 6` is Sunday. This condition means: "don't increment the streak if today is Sunday." When a user listens on Saturday and then Sunday, `days_since_last == 1` is true, but `today.weekday() != 6` is false — so the condition fails and the streak resets to 1 instead of incrementing. There is no valid reason to treat Sunday differently.

**Fix:** Remove `and today.weekday() != 6`.

**Before:**
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
```

**After:**
```python
elif days_since_last == 1:
    user.listening_streak += 1
```

**Verified by:** `tests/test_streaks.py::test_streak_increments_on_sunday` — this test was already written and fails before the fix, passes after.

---

### Bug 2 — Friends Listening Now Shows Yesterday's Listeners

**File:** `services/feed_service.py`, line 13

**Root Cause:** `RECENT_THRESHOLD = timedelta(hours=24)` defines the window for "listening now." A 24-hour window includes events from yesterday: if a friend listened at 10 PM last night and it's currently 9 PM tonight (23 hours later), they appear as "listening now." The feature is called "listening now," so a 24-hour lookback is far too large. A 30-minute window matches what "now" means for an active listener.

**Fix:** Change `timedelta(hours=24)` to `timedelta(minutes=30)`.

**Before:**
```python
RECENT_THRESHOLD = timedelta(hours=24)
```

**After:**
```python
RECENT_THRESHOLD = timedelta(minutes=30)
```

---

### Bug 3 — Duplicate Songs in Search Results

**File:** `services/search_service.py`, line 27–35

**Root Cause:** `search_songs()` uses `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` to join the tags association table. SQL joins produce one row per matching join pair — a song with 3 tags produces 3 rows, one per tag. SQLAlchemy's `.all()` returns all rows, so a song with N tags appears N times in the result list. A song with no tags appears once (the outer join keeps it with a NULL tag row). The fix is to add `.distinct()` so each Song object is returned only once.

**Fix:** Add `.distinct()` to the query chain.

**Before:**
```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(...)
    .all()
)
```

**After:**
```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(...)
    .distinct()
    .all()
)
```

**Regression test:** `tests/test_search.py::test_search_no_duplicates_multi_tag_song` — already in the repo and fails before the fix, passes after.

---

### Bug 4 — No Notification When a Song Is Rated

**File:** `services/notification_service.py`, `rate_song()` function

**Root Cause:** This is an architectural omission. The `add_to_playlist()` function correctly calls `create_notification()` after performing its action. `rate_song()` performs the same type of social action (someone interacts with a song you shared), but never calls `create_notification()`. The notification infrastructure is all there — the function just doesn't use it. Comparing the two functions line-by-line makes the missing call obvious.

**Fix:** After committing the rating, add a notification for the song's sharer (when the rater is not the sharer).

**Added after `db.session.commit()`:**
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

---

### Bug 5 — Last Song in a Playlist Never Shows Up

**File:** `services/playlist_service.py`, line 66

**Root Cause:** `get_playlist_songs()` queries the database correctly and retrieves all songs in order. But the return statement uses `songs[:-1]` — Python's slice that means "all elements except the last one." A playlist with 5 songs returns 4; with 1 song it returns 0 (an empty list). This is a typo; `songs[:-1]` should be `songs`.

**Fix:** Change `songs[:-1]` to `songs`.

**Before:**
```python
return [song.to_dict() for song in songs[:-1]]
```

**After:**
```python
return [song.to_dict() for song in songs]
```

**Verified by:** `tests/test_playlists.py::test_playlist_returns_all_songs` — this test asserts `len(songs) == 5` and fails before the fix with 4, passes after.

---

## AI Tool Disclosure

Claude Code was used to navigate the codebase (reading files and tracing call chains) and to draft this submission document. All bug identification and analysis was done by reading the actual source code. The bugs were confirmed by running the existing test suite before and after each fix.
