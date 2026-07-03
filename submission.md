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

---

### Issue #1 — My listening streak keeps resetting

**File:** [services/streak_service.py](services/streak_service.py), line 73

**How I reproduced it:** I read `test_streak_increments_on_sunday` in `tests/test_streaks.py`, which uses Saturday (June 15, 2024, `weekday() == 5`) and Sunday (June 16, 2024, `weekday() == 6`) as inputs and asserts the streak reaches 2. Running the test before any fix: it failed with streak == 1 on Sunday, confirming that a Saturday → Sunday listen pair was resetting instead of incrementing.

**How I found the root cause:** I went straight to `services/streak_service.py` since the README pointed there. I read `update_listening_streak()` top to bottom. The logic has three branches based on `days_since_last`: 0 (same day), 1 (consecutive), and else (gap). The consecutive-day branch on line 73 read:
```python
elif days_since_last == 1 and today.weekday() != 6:
```
The extra condition `today.weekday() != 6` immediately stood out as the culprit — Python's `datetime.weekday()` returns 6 for Sunday, so this guard literally says "skip the increment when today is Sunday."

**The root cause:** Python's `datetime.weekday()` returns integers 0 (Monday) through 6 (Sunday). The streak increment branch had an appended guard `and today.weekday() != 6`, which evaluates to `False` whenever the current day is Sunday. When a user listens on Saturday and then on Sunday, `days_since_last == 1` is True but `today.weekday() != 6` is False — so the whole `elif` is skipped and execution falls into the `else` branch, resetting the streak to 1. There is no domain reason to exclude Sunday from consecutive-day streaks; the condition has no valid justification.

**Fix and side-effect check:** Removed `and today.weekday() != 6` so the branch is simply `elif days_since_last == 1:`. I checked the surrounding logic: `days_since_last == 0` (same day, no change) and the `else` reset branch are unaffected. All five streak tests pass after the fix: `test_streak_starts_at_1_for_new_user`, `test_streak_increments_on_consecutive_day`, `test_streak_does_not_double_count_same_day`, `test_streak_resets_after_skipped_day`, and `test_streak_increments_on_sunday`.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**File:** [services/feed_service.py](services/feed_service.py), line 13

**How I reproduced it:** The bug is a threshold value, so I traced the logic mentally: if a friend listened at 10 PM on Tuesday and it's currently 9 PM on Wednesday, `datetime.now(UTC) - timedelta(hours=24)` produces a cutoff at 9 PM Tuesday. The friend's 10 PM Tuesday event is 23 hours old, which is >= cutoff, so they appear as "listening now." That's yesterday's data surfacing in a "now" feed. The scenario is reliably reproducible any time a friend listened 1–23 hours ago.

**How I found the root cause:** I opened `services/feed_service.py` and found `RECENT_THRESHOLD = timedelta(hours=24)` at the top of the file. `get_friends_listening_now()` computes `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and filters `ListeningEvent.listened_at >= cutoff`. A 24-hour window is the only thing controlling what "now" means — if the threshold is 24 hours, yesterday's listeners appear today. The constant name `RECENT_THRESHOLD` and the feature name "listening now" made it obvious 24 hours was too broad.

**The root cause:** `RECENT_THRESHOLD = timedelta(hours=24)` makes "listening now" mean "anyone who listened in the past 24 hours." Since 24 hours crosses midnight, events from yesterday are always included unless the user has been active for a full day. A "listening now" feature should reflect the current session, not a full day's history.

**Fix and side-effect check:** Changed `timedelta(hours=24)` to `timedelta(minutes=30)`. A 30-minute window matches the intent of "currently listening." I checked `get_activity_feed()` in the same file — it is a separate function with its own `limit` parameter and no time window, so it is not affected by this constant. No existing tests cover the feed service, so no regressions to check there.

---

### Issue #3 — The same song keeps showing up twice in search

**File:** [services/search_service.py](services/search_service.py), lines 25–37

**How I reproduced it:** I ran `test_search_no_duplicates_multi_tag_song` before any fix. The test seeds a song ("Crown Heights Anthem") with three tags (rap, hip-hop, boom bap), searches for it by title, and asserts the result list contains exactly one entry with that title. The test failed with `len(matching) == 3` — one copy per tag. I also noticed `test_search_no_duplicates_single_tag_song` was passing (one tag → no duplicate), which confirmed the bug was conditional on having multiple tags.

**How I found the root cause:** I read `search_songs()` in `services/search_service.py`. The query joins `song_tags` with `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` before filtering on title/artist. I knew immediately that a SQL join without deduplication returns one row per join match — a song with 3 tags joins 3 rows. SQLAlchemy's `.all()` materializes all rows as separate Song objects (SQLAlchemy deduplicates by identity in the session only when loading the same PK within the same query result, which it does not do here for the same object appearing multiple times). The fix was to add `.distinct()` to collapse duplicate rows before `.all()`.

**The root cause:** The `outerjoin` on `song_tags` produces one SQL row per `(song_id, tag_id)` pair. A song with N tags produces N rows. SQLAlchemy's `.all()` returns one Python object per row, so the same Song appears N times in `results`. The outer join is needed for the tags relationship to load, but without `.distinct()` the SQL `SELECT` returns duplicate song rows that feed through to the Python list.

**Fix and side-effect check:** Added `.distinct()` between `.filter(...)` and `.all()`. This collapses the duplicate rows at the SQL level before SQLAlchemy materializes them. Songs with no tags still appear (the outer join produces a single NULL-tag row, which `.distinct()` keeps). Songs with multiple tags now appear once. I checked `get_song()` in the same file — it uses `db.session.get()` (PK lookup), completely unrelated. All five search tests pass after the fix.

**Regression test:** `tests/test_search.py::test_search_no_duplicates_multi_tag_song` was already in the repo and directly tests this behavior. It fails before the fix and passes after.

---

### Issue #4 — I got notified when a friend added my song to a playlist, but not when they rated it

**File:** [services/notification_service.py](services/notification_service.py), `rate_song()` function

**How I reproduced it:** I read `add_to_playlist()` and `rate_song()` side by side. `add_to_playlist()` calls `create_notification()` after its DB write. `rate_song()` calls `db.session.commit()` and then immediately `return rating` — no notification. To confirm this is the only difference, I traced the route: `POST /songs/<id>/rate` calls `rate_song()` and nothing else. There is no other place that could fire a "song_rated" notification.

**How I found the root cause:** The README hint said "the root cause is architectural, not a typo — look at the pattern used for the working notification and compare it line-by-line to the missing one." I opened `notification_service.py` and read both functions. `add_to_playlist()` (lines 35–70) has this structure: validate inputs → perform DB action → call `create_notification()` → return. `rate_song()` (lines 73–110) has: validate inputs → perform DB action → commit → return. The call to `create_notification()` is simply absent. The infrastructure (`create_notification`, `Notification` model, `get_notifications`) is complete — the notification just was never triggered.

**The root cause:** `rate_song()` saves and commits the Rating row but never calls `create_notification()`. The `add_to_playlist()` function in the same file demonstrates the correct pattern: after the data action, check whether the actor is the song's original sharer, and if not, fire a notification. `rate_song()` skips that step entirely. The omission is not a typo in an existing call — the call simply doesn't exist.

**Fix and side-effect check:** Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `if song.shared_by != user_id` (so users don't get notified when they rate their own song, matching the pattern in `add_to_playlist()`). The notification type is `"song_rated"` and the body names the rater and the score. I checked: the `create_notification()` function does its own commit, which is safe after the Rating commit since they are independent rows. The `get_notifications()` and `mark_as_read()` functions are unaffected — they work on any Notification row regardless of type.

---

### Issue #5 — The last song in a playlist never shows up

**File:** [services/playlist_service.py](services/playlist_service.py), line 66

**How I reproduced it:** I ran `test_playlist_returns_all_songs` before any fix. The test creates a playlist with 5 songs at positions 1–5 and asserts `len(songs) == 5`. It failed with `len(songs) == 4`. I also checked `test_playlist_returns_songs_in_order`, which asserts the titles are `["Track 1", "Track 2", "Track 3", "Track 4", "Track 5"]` — it failed because "Track 5" was missing.

**How I found the root cause:** I read `get_playlist_songs()` in `services/playlist_service.py`. The SQL query (lines 58–64) is correct: it joins through `playlist_entries`, filters by `playlist_id`, and orders by `position` ascending. Then line 66:
```python
return [song.to_dict() for song in songs[:-1]]
```
`songs[:-1]` is a Python slice meaning "all elements except the last." The query returns the right data; the slice discards the final element.

**The root cause:** `songs[:-1]` is Python's "all but last" slice. A query returning N songs becomes N−1 songs before the list comprehension even runs. With 5 songs: returns 4. With 1 song: returns 0 (empty list). The SQL and the join logic are correct; the only problem is `[:-1]` where `[:]` (or no slice at all) was intended.

**Fix and side-effect check:** Changed `songs[:-1]` to `songs` in the list comprehension. The query, join, and ordering are untouched. I verified `get_playlist()` (metadata only, no songs) and `get_user_playlists()` (returns Playlist objects, not song lists) are unrelated. All three playlist tests pass after the fix: `test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`, and `test_empty_playlist_returns_empty_list`.

---

## Regression Test

`tests/test_search.py::test_search_no_duplicates_multi_tag_song` is a regression test for Bug 3. It would have caught the bug before it was introduced: it seeds a song with 3 tags, runs a search, and asserts the result contains exactly one copy of that song (`assert len(matching) == 1`). With the bug present (no `.distinct()`), this test fails with `len(matching) == 3`. With the fix, it passes.

The test is already in the repo. Run it with:
```bash
pytest tests/test_search.py::test_search_no_duplicates_multi_tag_song -v
```

---

## AI Tool Disclosure

Claude Code was used to navigate the codebase (reading files, tracing call chains) and to draft this submission document. Bug identification was done by reading the actual source code. The existing test suite was run before and after each fix to confirm behavior.
