# Submission

## AI Usage

- I asked Claude to explain to me specific functions or lines of code. For example, what `outerjoin` does, how the models are defined throughout the app, and the db session initialization. Understanding what each thing did helped me ensure that I was looking for the bug in the right place and better understand how certain functions worked.

## Description of Each File

### routes

- `feed.py`: Opens the feed service on the app, allows user to see friends that are listening and activity.
- `playlists.py`: Opens the playlist service on the app, allows user to edit, create, and view playlists.
- `songs.py`: Opens the song service on the app, allows user to view, search, rate, and listen to songs.
- `users.py`: Opens the user service on the app, allows user to view notifications, streaks, and user information.

### services

- `feed_service.py`: Returns information about listening activity of the user's friends.
- `notification.py`: Returns when a user's friend interacts with a song that user shared.
- `playlist_service.py`: Creates new playlists and returns songs in the playlist or metadata.
- `search_service.py`: Returns songs searched by title, artist name, or UUID.
- `streak_service.py`: Updates and returns the listening streak of a user.

### tests

- `test_playlists.py`: Tests that playlist is returned in order and with all songs included.
- `test_search.py`: Tests that songs are able to be searched correctly and tags work as intended.
- `test_streaks.py`: Tests that streak counts work as intended.

### other

- `app.py`: Initializes all routes' connections to the app.
- `models.py`: Generates tables for user, tag, song, listening_event, rating, playlist, and notification which are filled with data by the routes. There are also association tables and relationships created that allow the services to work as intended.

## Data Flow for a User Searching a Song and Adding It to a Playlist

- `GET routes/songs` is opened and the route calls `search_songs()` in `search_service.py`. The song table is queried through the shared `db.session` and a list of songs is returned. The user chooses one of the songs and sends a separate request to `POST routes/playlists`. `add_song()` validates the `song_id` and `added_by` fields and uses `add_to_playlist()` in `notification_service.py` to append the song to the playlist and notify the original sharer.

## Patterns Noticed

- The notifications route handles most updates to ratings or playlists. These are stored here to make the app interactive between users, as opposed to storing update methods within their respective routes.

## Bugs

### Bug 1 — My listening streak keeps resetting (`streak_service.py`)

- **How I Reproduced Error:** Ran `pytest tests/tests_streaks.py` to see the error message and walk through each `if` statement in `streak_service.py`.
- **How I Found Root Cause:** I opened `streak_service.py` and read through each line to understand what it was doing. I asked Claude to explain to me what `now.date()` and `.weekday()` meant. Based off of this I ran the tests and found the bug on line 73.
- **Root Cause:** Line 73 in `streak_service.py` included `and today.weekday() != 6` in the `elif` statement. This made it so that the user's streak would update only if the weekday was not a Saturday and the user listened yesterday. Otherwise, the `else` statement would be executed and the user's streak would be reset.
- **Fix and Side-Effect Check:** I deleted the part of the if-else statement shown above and reran the pytest to make sure that all tests passed.

### Bug 2 — Friends Listening Now shows people from yesterday (`feed_service.py`)

- **How I Reproduced Error:** Thought about a datatable with listening times extending multiple days. Noticed the `RECENT_THRESHOLD` variable being set to 24 hours and thought about how this would cause the bug if, for example, the user was to check listening now in the morning.
- **How I Found Root Cause:** Opened `feed_service.py` and read through the `get_friends_listening_now()` function. Noticed the `RECENT_THRESHOLD` variable was always set to 24 hours.
- **Root Cause:** Line 13's `RECENT_THRESHOLD` variable caused the cutoff on line 32 to check the last 24 hours instead of the current day.
- **Fix and Side-Effect Check:** Deleted `RECENT_THRESHOLD` and changed the cutoff to be set to midnight of the current day. Ran the `get_friends_listening_now()` function to ensure it runs correctly and made sure the variable is not used elsewhere.

### Bug 3 — The same song keeps showing up twice in search (`search_service.py`)

- **How I Reproduced Error:** Looked at `search_songs()` in `search_service` and noticed that there was a join being done before the filter. I searched what an `outerjoin` did with `song_tags.c.song_id` and found that this meant it would join rows without tags, marking this field as null.
- **How I Found Root Cause:** Searched what each function in the extraction of the results did and noticed that the `outerjoin` was an unusual pattern. I looked closer into what this function does and saw that it joined tables based on the id without taking tags into account. This made me realize that songs with multiple tags would be counted multiple times since the table is only filtered by artist and song title.
- **Root Cause:** The `outerjoin` on song ids on line 27 made it so that songs with multiple tags were joined with each tag in a different row.
- **Fix and Side-Effect Check:** Deleted line 27, since the filter below selects songs based on artist and song title, which is enough to find a song. `.all()` at the end performs a join-like function by returning all of the rows that were selected by the filter.

### Bug 4 — I got notified when a friend added my song to a playlist but not when they rated it (`notification_service.py`)

- **How I Reproduced Error:** Opened `notification_service` and saw that there was a `create_notification()` method at the top that was called by `add_to_playlist()`. When I scrolled down, I saw that the `rate_song()` method was not calling this function, meaning that no notification would be sent.
- **How I Found Root Cause:** Looked at functions in `notification_service` and noticed that the function that is responsible for creating a notification was not called in the `rate_song()` function.
- **Root Cause:** `create_notification()` not called in `rate_song()`, which means that no notification is created when a user rates a song.
- **Fix and Side-Effect Check:** Added a call to the `create_notification()` method to `rate_song()` with `song.shared_by`, `"song_rated"`, and a short description as parameters before the rating is returned. No check is included before the function call since there is a check on line 97 to check if the user has already rated the song.

### Bug 5 — The last song in a playlist never shows up (`playlist_service.py`)

- **How I Reproduced Error:** Opened `playlist_service.py` and looked at the `get_playlist_songs()` function and read through each line, thinking about how the function would be called and what the output would look like.
- **How I Found Root Cause:** Reading through each line of the function allowed me to see the error in the return statement of the function on line 66.
- **Root Cause:** The list that was returned on line 66 looped through `songs[:-1]`, which meant that the last element was always excluded.
- **Fix and Side-Effect Check:** I deleted the `:-1` inside the brackets on line 66. Since nothing else changes, there are no side effects.
