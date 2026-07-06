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

*(To be filled in as bugs are fixed)*

---
