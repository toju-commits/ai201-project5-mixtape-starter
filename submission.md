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

TODO

### How I found the root cause

TODO

### The root cause

TODO

### Fix and side-effect check

TODO

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

TODO

### How I found the root cause

TODO

### The root cause

TODO

### Fix and side-effect check

TODO

## Issue 5: The last song in a playlist never shows up

### How I reproduced it

TODO

### How I found the root cause

TODO

### The root cause

TODO

### Fix and side-effect check

TODO