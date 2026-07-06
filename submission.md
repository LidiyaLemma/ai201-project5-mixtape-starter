# Mixtape Bug Hunt — Submission

## Codebase Map

### Main Files and Their Roles

**app.py** — Flask app factory and database setup. Creates the Flask app instance and initializes SQLAlchemy.

**models.py** — Defines 6 SQLAlchemy models:
- `User` — has username, email, listening_streak, last_listened_at
- `Song` — a shared song with title, artist, album, genre, shared_by
- `Rating` — a user's 1-5 score on a song. Has a UniqueConstraint on (user_id, song_id) meaning one user can only rate each song once
- `ListeningEvent` — records when a user listened to a song
- `Playlist` — a named collection of songs, can be collaborative
- `Notification` — a message to a user about activity (song rated, added to playlist, etc.)

**routes/** — Handle HTTP requests, parse input, call services, format responses. No business logic lives here.
- `songs.py` — song sharing, search, rating
- `playlists.py` — playlist creation and song management
- `users.py` — user profiles, streaks, notifications
- `feed.py` — friends listening now, activity feed

**services/** — All business logic lives here. This is where the bugs are.
- `streak_service.py` — calculates listening streaks
- `feed_service.py` — friends currently listening feed
- `search_service.py` — song search
- `notification_service.py` — creates and retrieves notifications
- `playlist_service.py` — playlist song retrieval

### Data Flow Example — User Rates a Song

1. User sends `POST /songs/<song_id>/rate`
2. `routes/songs.py` receives the request, parses the score
3. Calls `notification_service.rate_song()`
4. That function creates a `Rating` record and a `Notification` for the song's original sharer
5. Response is returned to the user

### Pattern I Noticed
Every route delegates immediately to a service function. Routes only handle input parsing and response formatting — all logic lives in services. This means every bug in the issue tracker will be found in the `services/` folder, not in `routes/`.

### The Five Open Issues
| # | Title | Affected service |
|---|-------|-----------------|
| 1 | Listening streak keeps resetting | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` |
| 3 | Same song shows up twice in search | `search_service.py` |
| 4 | No notification when friend rates a song | `notification_service.py` |
| 5 | Last song in playlist never shows up | `playlist_service.py` |


## Bug Fixes

### Bug #5 — Last song in playlist never shows up

**File:** `services/playlist_service.py`

**How I reproduced it:** Called `GET /playlists/<id>/songs` on any playlist with multiple songs — the last song in the list was always missing from the response.

**Root cause:** `get_playlist_songs()` returned `songs[:-1]` which in Python means "everything except the last item." This silently dropped the final song from every playlist response.

**Fix:** Changed `songs[:-1]` to `songs` to return the complete list.

**One-line summary:** Off-by-one slice error caused the last playlist song to always be excluded.

---

### Bug #3 — Same song shows up twice in search results

**File:** `services/search_service.py`

**How I reproduced it:** Searched for any song that has multiple tags — it appeared once per tag in the results.

**Root cause:** `search_songs()` does an `outerjoin` with the `song_tags` table to enable tag filtering. When a song has multiple tags, the join produces one row per tag, causing the same song to appear multiple times in results.

**Fix:** Added `.distinct()` to the query so each song only appears once regardless of how many tags it has.

**One-line summary:** Missing `.distinct()` on a join query caused songs with multiple tags to appear multiple times.

---

### Bug #2 — Friends Listening Now shows people from yesterday

**File:** `services/feed_service.py`

**How I reproduced it:** The feed was returning listening events older than 24 hours because the timezone-aware cutoff datetime couldn't be compared to the timezone-naive datetimes stored in the database.

**Root cause:** `datetime.now(timezone.utc)` produces a timezone-aware datetime, but SQLite stores datetimes without timezone info (timezone-naive). Comparing timezone-aware vs timezone-naive datetimes causes the filter to behave unpredictably, returning stale results.

**Fix:** Added `.replace(tzinfo=None)` to make the cutoff timezone-naive, matching the format stored in the database.

**One-line summary:** Timezone-aware cutoff couldn't be compared to timezone-naive database datetimes, causing the recency filter to fail.

## AI Usage

I used Claude (AI tool) during this project in the following ways:

1. **Codebase orientation:** Asked the AI to explain what each service file was responsible for after reading them myself. This helped confirm my understanding of the layered architecture (routes → services → models).

2. **Bug investigation:** For each bug, I read the suspicious code first, then asked the AI "what edge cases could cause this function to return wrong values?" For Bug #3, I had already noticed the outerjoin but asked the AI to confirm why joining without distinct causes duplicates. For Bug #5, I spotted `songs[:-1]` and asked the AI to confirm what Python slice notation does with negative indices. For Bug #2, I asked the AI to explain the difference between timezone-aware and timezone-naive datetimes in Python after noticing the cutoff used `timezone.utc` but the database stores naive datetimes.

3. **What I verified myself:** I read every fix myself before applying it and confirmed the specific line that needed changing rather than accepting the AI's suggested fix blindly.