# Project 5 Submission — Mixtape Bug Hunt

---

## AI Usage

I used Claude during codebase orientation to read and summarize each service file, trace the data flow for features, and identify patterns across files. Specifically:

- Asked Claude to read and explain each service file's responsibility and main functions.
- Used Claude to trace the call chain for features like song rating and playlist retrieval.
- Verified every diagnosis myself by reading the specific lines of code flagged.
- Claude helped narrow down where to look, but I confirmed each root cause by reading the code directly.

---

## Codebase Map

### Main Files and Their Roles

**`app.py`**
Flask application factory. Creates the Flask app, configures SQLite as the database (`instance/mixtape.db`), initializes SQLAlchemy, and registers the four route blueprints (`/songs`, `/playlists`, `/users`, `/feed`). All app setup lives here — nothing runs directly from this file.

**`models.py`**
Defines 6 SQLAlchemy models and 3 association tables:
- `User` — stores username, email, listening streak, and last listened timestamp. Has many-to-many self-referential `friends` relationship.
- `Song` — stores title, artist, album, genre, sharer ID, and a share note. Songs are owned by the user who shared them.
- `Tag` — simple label (name only). Many-to-many with Song via `song_tags`.
- `ListeningEvent` — records each time a user listens to a song, with a timestamp.
- `Rating` — a user's 1–5 score for a song. Enforces one rating per user/song pair via a unique constraint.
- `Playlist` — has a name, creator, and a collaborative flag. Songs are linked via `playlist_entries` with a `position` column for ordering.
- `Notification` — stores a type string, body message, read status, and recipient user ID.

Association tables: `friendships` (User ↔ User), `song_tags` (Song ↔ Tag), `playlist_entries` (Playlist ↔ Song, with `position` and `added_by`).

**`routes/songs.py`** — Handles song sharing, search, and rating endpoints. Delegates all logic to `search_service` and `notification_service`.

**`routes/playlists.py`** — Handles playlist creation and adding songs. Delegates to `notification_service` and `playlist_service`.

**`routes/users.py`** — Handles user profiles, streak updates, and notification retrieval. Delegates to `streak_service` and `notification_service`.

**`routes/feed.py`** — Handles "Friends Listening Now" and activity feed. Delegates to `feed_service`.

**`services/streak_service.py`** — Calculates and updates listening streaks. Core function: `update_listening_streak()` checks days since last listen and increments, holds, or resets the streak.

**`services/feed_service.py`** — Queries recent `ListeningEvent` records for a user's friends. `get_friends_listening_now()` uses a `RECENT_THRESHOLD` of 24 hours and deduplicates to one entry per friend.

**`services/search_service.py`** — Searches songs by title or artist using a case-insensitive `ILIKE` query with an outer join on `song_tags`.

**`services/notification_service.py`** — Creates `Notification` records. `add_to_playlist()` notifies the song's original sharer. `rate_song()` saves a rating. `get_notifications()` retrieves them ordered by most recent.

**`services/playlist_service.py`** — `get_playlist_songs()` queries songs joined through `playlist_entries`, ordered by `position` ascending.

**`seed_data.py`** — Populates the database with 5 users, 13 songs, 3 playlists, and 10 tags for testing.

---

### Data Flow — User Rates a Song

1. Client sends `POST /songs/<song_id>/rate` with a score in the request body.
2. `routes/songs.py` parses `user_id` and `score` from the request.
3. Route calls `notification_service.rate_song(user_id, song_id, score)`.
4. `rate_song()` validates the score (1–5), fetches the Song and User from the DB.
5. Checks if a Rating already exists for this user/song pair — updates it if so, creates a new one if not.
6. Commits to the database and returns the Rating.
7. Route returns the rating dict as JSON.

### Data Flow — User Adds a Song to a Playlist

1. Client sends `POST /playlists/<playlist_id>/songs` with `song_id` and `user_id`.
2. `routes/playlists.py` calls `notification_service.add_to_playlist(playlist_id, song_id, user_id)`.
3. `add_to_playlist()` fetches Song, User (adder), and Playlist from DB.
4. If the song isn't already in the playlist, appends it and commits.
5. If the adder is not the original sharer, calls `create_notification()` to notify the sharer.
6. `create_notification()` creates a `Notification` record and commits.

### Pattern Observed

Every route delegates immediately to a service function. Routes handle request parsing and response formatting only — all business logic and database interaction lives in `services/`. This separation makes the bugs easy to locate: if an endpoint behaves wrong, the bug is in its service.

---

## Root Cause Analysis

---

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**
Called `update_listening_streak()` directly with a controlled Sunday date (`datetime(2026, 7, 5)`). Set `last_listened_at` to the previous Saturday and `listening_streak` to 5. After calling the function with Sunday as "now", the streak reset to 1 instead of incrementing to 6.

**How I found the root cause:**
Read `streak_service.py` and immediately saw the condition on line 73: `elif days_since_last == 1 and today.weekday() != 6`. The `!= 6` clause stood out — it's adding an extra condition that blocks streak increment specifically on Sundays.

**The root cause:**
`datetime.weekday()` returns 6 for Sunday. The condition `today.weekday() != 6` evaluates to `False` on Sundays, which means the `elif` branch (which increments the streak) is never taken on Sundays. Instead it falls to `else` and resets the streak to 1. So any user who listens on a Sunday after listening on Saturday will lose their streak entirely, even though they listened on consecutive days.

**Fix and side-effect check:**
Removed `and today.weekday() != 6` from the `elif` condition in `update_listening_streak()`. The condition is now simply `elif days_since_last == 1`, which correctly increments the streak for any consecutive-day listen including Sundays. Verified that the skip-day reset (`else` branch) still works correctly: simulated a 2-day gap and confirmed streak resets to 1. The fix is a 1-character change and touches only the streak increment branch.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:**
Called `GET /feed/<nova_id>/listening-now`. The feed returned friends whose most recent events were many hours ago (10+ hours). The seed data places some listening events 2, 10, and 18 hours ago — all within the 24-hour window, even though "listening now" should mean actively listening in the last ~30 minutes.

**How I found the root cause:**
Read `feed_service.py` and saw `RECENT_THRESHOLD = timedelta(hours=24)` on line 13. The cutoff is 24 hours, meaning anyone who listened within the past day shows up as "listening now."

**The root cause:**
`RECENT_THRESHOLD` is set to `timedelta(hours=24)`, which is far too large for a "Friends Listening Now" feed. Someone who listened 23 hours ago (yesterday evening) would appear as currently active. The threshold should be a short window (e.g., 30 minutes) to reflect who is actually listening right now.

**Fix and side-effect check:**
Changed `RECENT_THRESHOLD = timedelta(hours=24)` to `RECENT_THRESHOLD = timedelta(minutes=30)`. Verified that listening events from 78+ minutes ago no longer appear in the feed, while events from the past 17 minutes still do. The `get_activity_feed()` function is intentionally unaffected — it has no recency filter and is designed to show all recent activity regardless of time.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:**
Queried the raw SQL for a multi-tag song (Crown Heights Anthem, which has 3 tags: rap, hip-hop, boom bap). The raw query produced 3 identical rows for the same song. With `db.session.query(Song).outerjoin(song_tags, ...)`, songs with multiple tags produce one row per tag in the SQL result set.

**How I found the root cause:**
Read `search_service.py` and saw the `outerjoin(song_tags, Song.id == song_tags.c.song_id)`. An outer join against the `song_tags` association table creates one result row per tag per song. A song with 3 tags produces 3 rows. Without `.distinct()`, the query relies on SQLAlchemy's identity map to avoid duplicates — which is fragile and version-dependent behavior.

**The root cause:**
The `outerjoin` on `song_tags` is needed to allow filtering by tags, but it causes the SQL to produce N rows per song (where N = number of tags). Without `.distinct()`, the results depend on SQLAlchemy's internal deduplication. The correct fix is to add `.distinct()` to the query so the SQL itself guarantees one row per song, regardless of how many tags it has.

**Fix and side-effect check:**
Added `.distinct()` to the SQLAlchemy query in `search_songs()`, between the `.filter()` and `.all()` calls. This pushes `SELECT DISTINCT` to the SQL level, guaranteeing one row per song regardless of how many tag associations it has. Verified that searching for "Borough" (3-tag song) and "Uptown" (3-tag song) each return exactly 1 result, and that songs with 0 tags (no outerjoin rows) still appear correctly.

---

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**
Checked nova's notification count before and after darius rated her song (`Midnight Drive`, score 5) via `POST /songs/<id>/rate`. Notification count stayed at 1 — no new notification was created despite darius rating nova's song.

**How I found the root cause:**
Compared `add_to_playlist()` and `rate_song()` in `notification_service.py` side by side. `add_to_playlist()` calls `create_notification()` after saving the playlist entry. `rate_song()` saves the rating and commits, but never calls `create_notification()` at all.

**The root cause:**
The `rate_song()` function in `notification_service.py` is missing the `create_notification()` call entirely. It validates the score, saves or updates the Rating record, and returns — but never notifies the original song sharer that someone rated their song. The pattern for notifying exists in `add_to_playlist()` but was never added to `rate_song()`.

**Fix and side-effect check:**
Added a `create_notification()` call at the end of `rate_song()`, mirroring the exact same pattern used in `add_to_playlist()`. The notification is only sent if the rater is not the original sharer (`song.shared_by != user_id`). Verified by having darius rate nova's song "Still Waters" — nova's notification count went from 1 to 2, with body "darius rated your song 'Still Waters' 4/5." Checked that rating one's own song does not trigger a self-notification.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**
Called `GET /playlists/<Late Night Vibes id>/songs`. The playlist has 7 songs seeded (positions 1–7), but the response returned only 6. The 7th song ("Free Throws") was consistently missing.

**How I found the root cause:**
Read `playlist_service.py`, specifically `get_playlist_songs()`. The last line reads: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice cuts off the last element of the list.

**The root cause:**
`songs[:-1]` is a Python slice that returns all elements except the last one. The query correctly fetches all songs ordered by position, but the return statement discards the final song in the list. This means the last song in every playlist is always excluded from the response, regardless of playlist size.

**Fix and side-effect check:**
Changed `songs[:-1]` to `songs` in the return statement of `get_playlist_songs()`. Verified that "Late Night Vibes" (7 songs) now returns all 7 including the previously missing last song "Free Throws". Checked the other two playlists ("Friday Energy" — 7 songs, "Study Mode" — 7 songs) and confirmed all songs appear. No other logic in the function was touched.

---
