# AI Usage

TODO

# Codebase Map

## Main files and responsibilities

- `app.py`: Creates the Flask app, configures SQLite through SQLAlchemy, initializes the database, and registers the route blueprints for songs, playlists, users, and feed. The app factory is `create_app()`.
- `models.py`: Defines the SQLAlchemy data models and relationships for users, songs, tags, listening events, ratings, playlists, playlist entries, friendships, and notifications.
- `routes/songs.py`: Handles song search, song detail lookup, listening events, and song rating endpoints.
- `routes/playlists.py`: Handles playlist creation, playlist detail lookup, retrieving playlist songs, and adding songs to playlists.
- `routes/users.py`: Handles user profile lookup, streak lookup, notification retrieval, and marking notifications as read.
- `routes/feed.py`: Handles the friends listening-now feed and the general friend activity feed.
- `services/streak_service.py`: Records listening events and updates user listening streaks.
- `services/feed_service.py`: Builds the listening-now feed and activity feed from friends’ listening events.
- `services/search_service.py`: Searches songs by title or artist and returns song dictionaries.
- `services/notification_service.py`: Creates notifications, handles playlist-add notification behavior, handles song rating behavior, retrieves notifications, and marks notifications as read.
- `services/playlist_service.py`: Creates playlists, retrieves playlist metadata, retrieves user playlists, and returns ordered playlist songs.
- `seed_data.py`: Resets and repopulates the database with sample users, friendships, songs, tags, listening events, playlists, playlist entries, ratings, and notifications.

## Data flow for one feature

When a user listens to a song, the client sends `POST /songs/<song_id>/listen` with a `user_id` in the JSON body. `routes/songs.py` reads the request, validates that `user_id` exists, and calls `record_listening_event()` in `services/streak_service.py`. The service loads the user, creates a `ListeningEvent`, calls `update_listening_streak()` to update the user’s streak, commits the database session, and returns the listening event as JSON.

## Patterns noticed

The app uses a route-to-service pattern. Route files handle HTTP request/response work, while service files contain the main business logic. The database models live in `models.py`, and most services query or update those models through `db.session`. The README says the known bugs are in the service layer, so debugging should start by reproducing an endpoint issue, tracing the route that handles it, then inspecting the service function called by that route.

# Bug Fixes

## Issue 1: My listening streak keeps resetting

### How I reproduced it

I ran `pytest tests/` and saw `test_streak_increments_on_sunday` fail. The test listened on Saturday and then Sunday. The expected streak was 2, but the actual streak was 1, meaning the Sunday listen reset instead of continuing the streak.

### How I found the root cause

I traced the listen feature from `POST /songs/<song_id>/listen` in `routes/songs.py` to `record_listening_event()` in `services/streak_service.py`. That function creates the listening event, then calls `update_listening_streak()`, so I inspected the streak update rules there.

### The root cause

`update_listening_streak()` correctly calculated that the previous listen was 1 day ago, but the increment condition also checked `today.weekday() != 6`. Since Python uses `6` for Sunday, a Saturday-to-Sunday listen skipped the increment branch and fell into the reset branch. This violated the stated rule that listening on consecutive calendar days should increment the streak.

### Fix and side-effect check

I removed the Sunday exclusion so any `days_since_last == 1` case increments the streak. I then ran `pytest tests/test_streaks.py` to verify the streak behavior and `pytest tests/` to check the whole project. The fix should make the full suite pass without changing same-day or skipped-day behavior.

## Issue 2: Friends Listening Now shows people from yesterday

### How I reproduced it

TODO

### How I found the root cause

TODO

### The root cause

TODO

### Fix and side-effect check

TODO

## Issue 3: The same song keeps showing up twice in search

### How I reproduced it

TODO

### How I found the root cause

TODO

### The root cause

TODO

### Fix and side-effect check

TODO

## Issue 4: I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

I first queried the seeded users and songs from the database to choose a song owner and a different rater. I used `Midnight Drive`, which was shared by `nova`, and used `darius` as the user rating the song.

Before rating the song, I called:

`GET /users/<nova_user_id>/notifications`

The response showed `count: 1`, with one existing `song_added_to_playlist` notification saying that darius added nova's song `Midnight Drive` to a playlist.

Then I called:

`POST /songs/<midnight_drive_song_id>/rate`

with a JSON body containing darius's user id and a score of 5. The response returned a valid rating object with score 5, so the rating itself succeeded.

After that, I called nova's notifications endpoint again. The response still showed `count: 1`, and there was no new notification for the rating. This reproduced the bug: rating another user's shared song succeeds, but the original sharer does not receive a notification.

### How I found the root cause

TODO

### The root cause

TODO

### Fix and side-effect check

TODO

## Issue 5: The last song in a playlist never shows up

### How I reproduced it

I ran the baseline test suite with `pytest tests/`. The playlist tests failed because `get_playlist_songs()` returned 4 songs when the seeded playlist contained 5. The order test also showed that `"Track 5"` was missing from the returned list.

### How I found the root cause

I traced the playlist song endpoint from `GET /playlists/<playlist_id>/songs` in `routes/playlists.py` to `get_playlist_songs()` in `services/playlist_service.py`. The route was only returning whatever the service gave it, so the bug had to be in the service logic.

### The root cause

`get_playlist_songs()` queried the songs in the correct playlist order, but the return statement used `songs[:-1]`. In Python, that slice returns every item except the final one, so the last playlist song was always removed before the response was built.

### Fix and side-effect check

I changed the return statement to iterate over `songs` instead of `songs[:-1]`. Then I ran `pytest tests/test_playlists.py`, and all playlist tests passed. I also reran `pytest tests/`, which improved the suite from 3 failures to 1 remaining failure, confirming the playlist bug was fixed without breaking the search or other playlist behavior.