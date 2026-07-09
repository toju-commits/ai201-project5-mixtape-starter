# AI Usage

I used AI tools as a debugging partner during codebase navigation and documentation, not as a replacement for reproducing or verifying the bugs myself.

First, I used AI to help orient myself in the unfamiliar Flask codebase. I provided the route list from `python -m flask --app "app:create_app" routes` and asked how the endpoints connected to the files in `routes/` and `services/`. This helped me understand the app's route-to-service structure: routes receive HTTP requests and delegate the main business logic to service functions.

Second, I used AI to help trace specific call chains after I had reproduced the bugs. For example, for the rating notification issue, I asked AI to walk through `POST /songs/<song_id>/rate` from `routes/songs.py` into `services/notification_service.py`. That helped me compare `rate_song()` against the working `add_to_playlist()` notification pattern.

Third, I used AI to help turn my debugging notes into root cause analysis entries. I still verified each explanation against the actual code and runtime behavior before committing. For Issue 4, AI suggested that `rate_song()` needed the same notification pattern as `add_to_playlist()`, but I verified this myself by checking nova's notifications before and after darius rated `Midnight Drive`. The notification count stayed at 1 before the fix and increased to 2 after the fix.

One course-correction was that I initially moved too quickly toward fixing bugs. I slowed down and used the README, route list, baseline test failures, and manual curl checks to make sure each bug was reproduced before treating the AI explanation as correct.

# Codebase Map

## Main files and responsibilities

- `app.py`: Creates the Flask app, configures SQLite through SQLAlchemy, initializes the database, and registers the route blueprints for songs, playlists, users, and feed. The app factory is `create_app()`.
- `models.py`: Defines the SQLAlchemy data models and relationships for users, songs, tags, listening events, ratings, playlists, playlist entries, friendships, and notifications.
- `routes/songs.py`: Handles song search, song detail lookup, listening events, and song rating endpoints. It validates request data and delegates the actual work to service functions.
- `routes/playlists.py`: Handles playlist creation, playlist detail lookup, retrieving playlist songs, and adding songs to playlists.
- `routes/users.py`: Handles user profile lookup, streak lookup, notification retrieval, and marking notifications as read.
- `routes/feed.py`: Handles the friends listening-now feed and the general friend activity feed.
- `services/streak_service.py`: Records listening events and updates user listening streaks.
- `services/feed_service.py`: Builds the listening-now feed and activity feed from friends' listening events.
- `services/search_service.py`: Searches songs by title or artist and returns song dictionaries.
- `services/notification_service.py`: Creates notifications, handles playlist-add notification behavior, handles song rating behavior, retrieves notifications, and marks notifications as read.
- `services/playlist_service.py`: Creates playlists, retrieves playlist metadata, retrieves user playlists, and returns ordered playlist songs.
- `seed_data.py`: Resets and repopulates the database with sample users, friendships, songs, tags, listening events, playlists, playlist entries, ratings, and notifications.

## Data flow for one feature

When a user listens to a song, the client sends `POST /songs/<song_id>/listen` with a `user_id` in the JSON body. `routes/songs.py` reads the request, validates that `user_id` exists, and calls `record_listening_event()` in `services/streak_service.py`. The service loads the user, creates a `ListeningEvent`, calls `update_listening_streak()` to update the user's streak, commits the database session, and returns the listening event as JSON.

## Patterns noticed

The app uses a route-to-service pattern. Route files handle HTTP request/response work, while service files contain the main business logic. The database models live in `models.py`, and most services query or update those models through `db.session`. The README says the known bugs are in the service layer, so debugging should start by reproducing an endpoint issue, tracing the route that handles it, then inspecting the service function called by that route.

# Bug Fixes

## Issue 1: My listening streak keeps resetting

### How I reproduced it

I ran the baseline test suite with `pytest tests/` and saw `test_streak_increments_on_sunday` fail. The test simulated a user listening on Saturday and then again on Sunday. The expected streak was 2, but the actual streak stayed at 1, meaning the Sunday listen reset the streak instead of continuing it.

### How I found the root cause

I traced the listen feature from `POST /songs/<song_id>/listen` in `routes/songs.py`. That route reads the user id from the request body and calls `record_listening_event()` in `services/streak_service.py`. `record_listening_event()` creates the `ListeningEvent` and then calls `update_listening_streak(user, now)`, so I inspected that function because it is where the streak rules are implemented.

The moment that made me confident I had found the root cause was seeing that `update_listening_streak()` calculated `days_since_last == 1`, but then added an extra condition: `today.weekday() != 6`. Since the failing test was specifically Saturday into Sunday, this condition matched the failure exactly.

### The root cause

`update_listening_streak()` correctly calculated that the previous listen was 1 day ago, but the increment condition also checked `today.weekday() != 6`. In Python, `weekday()` returns `6` for Sunday. Because of that comparison, a Saturday-to-Sunday listen skipped the increment branch and fell into the reset branch. This violated the rule in the function's own docstring: if the user listened yesterday, the streak should increment.

### Fix and side-effect check

I removed the Sunday exclusion so any `days_since_last == 1` case increments the streak. The fix changed the condition from checking both `days_since_last == 1` and `today.weekday() != 6` to checking only `days_since_last == 1`.

For side effects, I ran `pytest tests/test_streaks.py` to verify the streak behavior specifically, then ran `pytest tests/` to verify the whole project. This was important because changing streak logic could have broken first-listen behavior, same-day behavior, or reset-after-skipped-day behavior. The full test suite passed after the fix.

## Issue 4: I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

I first queried the seeded users and songs from the database to choose a song owner and a different rater. I used `Midnight Drive`, which was shared by `nova`, and used `darius` as the user rating the song.

Before rating the song, I called `GET /users/<nova_user_id>/notifications`. The response showed `count: 1`, with one existing `song_added_to_playlist` notification saying that darius added nova's song `Midnight Drive` to the playlist `Late Night Vibes`.

Then I called `POST /songs/<midnight_drive_song_id>/rate` with a JSON body containing darius's user id and a score of 5. The response returned a valid rating object, so the rating itself succeeded.

After that, I called nova's notifications endpoint again. The response still showed `count: 1`, and there was no new notification for the rating. This reproduced the bug: rating another user's shared song succeeds, but the original sharer does not receive a notification.

### How I found the root cause

I traced the rating request from `POST /songs/<song_id>/rate` in `routes/songs.py`. The route reads `user_id` and `score` from the JSON body, then calls `rate_song(user_id, song_id, int(score))` from `services/notification_service.py`.

Inside `rate_song()`, the service validates the score, loads the song and rater, checks whether the user already rated the song, then either updates the existing `Rating` or creates a new one. After that, it commits the database session and returns the rating.

I compared this with `add_to_playlist()` in the same service file. `add_to_playlist()` checks whether the actor is different from the original song sharer and then calls `create_notification()` for the original sharer. `rate_song()` did not have any equivalent notification logic. That comparison made me confident that the root cause was not the route or the notification retrieval endpoint; the missing step was inside the rating service itself.

### The root cause

`rate_song()` only handled the rating data change. It created or updated a `Rating`, committed the session, and returned the rating. It never called `create_notification()` for the original user who shared the song. Because of that missing side effect, rating another user's shared song saved correctly but did not notify the song sharer.

### Fix and side-effect check

I added notification creation inside `rate_song()` when the rater is not the original song sharer. The new notification uses the type `song_rated` and tells the sharer who rated their song and what score was given.

To verify the fix, I repeated the manual reproduction. Before rating, nova had `count: 1` notification. After darius rated `Midnight Drive` with score 4, nova's notification count increased to `2`, and the new notification had type `song_rated` with the body `darius rated your song 'Midnight Drive' 4/5.`

I also ran `pytest tests/`, and the full suite passed with `13 passed`. This side-effect check mattered because `notification_service.py` also handles playlist notifications and notification retrieval, so I confirmed the new rating notification appeared without breaking existing notification behavior.

## Issue 5: The last song in a playlist never shows up

### How I reproduced it

I ran the baseline test suite with `pytest tests/`. The playlist tests failed because `get_playlist_songs()` returned 4 songs when the seeded playlist contained 5. The order test also showed that `"Track 5"` was missing from the returned list.

### How I found the root cause

I traced the playlist song endpoint from `GET /playlists/<playlist_id>/songs` in `routes/playlists.py` to `get_playlist_songs()` in `services/playlist_service.py`. The route only called the service and returned whatever list the service produced, so the bug had to be in the service logic rather than the route.

Inside `get_playlist_songs()`, the query joined `Song` with `playlist_entries`, filtered by `playlist_id`, and ordered by `playlist_entries.c.position`. That matched the expected behavior. The moment that made me confident I had found the actual root cause was the return statement: it used `songs[:-1]`, which slices off the final item after the query has already retrieved the correct list.

### The root cause

`get_playlist_songs()` queried the songs in the correct playlist order, but the return statement used `songs[:-1]`. In Python, that slice returns every item except the final one. Because the slicing happened after the query, the database lookup was correct but the response-building step always removed the last playlist song before returning JSON.

### Fix and side-effect check

I changed the return statement to iterate over `songs` instead of `songs[:-1]`. Then I ran `pytest tests/test_playlists.py`, and all playlist tests passed. I also reran `pytest tests/`, which improved the suite from 3 failures to 1 remaining failure at that point.

This was a targeted side-effect check because playlist ordering could have been affected if I changed the query itself. I did not change the join, filter, or `order_by`; I only removed the slice that dropped the last item. The playlist order test confirmed that the songs still came back in the correct position order while also including the final song.

# Final Verification

- `pytest tests/` passed with `13 passed` after the three fixes.
- `git log --oneline` on `bugfix/mixtape` shows three separate fix commits:
  - `fix: return final song in playlist results`
  - `fix: allow Sunday listening streak increments`
  - `fix: notify sharer when song is rated`
