## Codebase Map

**Structure:** Flask app with a classic routes → services → models layering.
Routes handle request parsing and response formatting only; all business
logic lives in `services/`. `models.py` defines 7 SQLAlchemy models plus
3 association tables.

**Models:**
- `User` — has a `listening_streak` counter and `last_listened_at` timestamp
  updated on every listen.
- `Song` — has a `shared_by` field (the original sharer), tags via the
  `song_tags` join table.
- `ListeningEvent` — one row per listen, tied to user + song + timestamp.
- `Rating` — one row per (user, song) pair, unique constraint enforced.
- `Playlist` — songs attached via `playlist_entries`, which stores an
  explicit `position` column (not just insertion order).
- `Notification` — generic type/body/read fields, created by service
  functions, not automatically.

**Data flow — user rates a song:**
`POST /songs/<id>/rate` (routes/songs.py) → `notification_service.rate_song()`.
This validates the score, finds or creates a `Rating` row, and commits.
Despite living in `notification_service.py`, this function never calls
`create_notification()` — worth tracing carefully against
`add_to_playlist()`, which does notify.

**Data flow — friends listening now:**
`GET /feed/<id>/listening-now` → `feed_service.get_friends_listening_now()`.
Pulls all `ListeningEvent`s for the user's friends within a fixed time
window (`RECENT_THRESHOLD`), then deduplicates to one song per friend
(most recent only).

**Pattern noticed:** every route wraps its service call in a try/except
ValueError → 404. Services never touch `request` or `jsonify` — they're
pure functions returning models or dicts. Cross-service imports are
sometimes done inline inside functions (e.g. `notification_service.
add_to_playlist` imports `Playlist` and `playlist_service` inline) to
avoid circular imports.

## Root Cause Analysis

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:**
In `flask shell`, I set a user's `listening_streak` to 12 and `last_listened_at`
to a Saturday timestamp. I then called `update_listening_streak()` directly
with a Sunday timestamp (confirmed `sunday.weekday() == 6`, Python's Sunday
value). Expected the streak to increment to 13 since the listens were on
consecutive calendar days; instead it reset to 1.

**How I found the root cause:**
I opened `streak_service.py` and read `update_listening_streak()` line by
line. The branch that handles consecutive-day listens is:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
The `and today.weekday() != 6` condition stood out immediately as unrelated
to the stated streak rules (which only mention "listened yesterday" vs.
"more than one day passed" — nothing about weekdays). Confirming
`weekday() == 6` maps to Sunday verified this was the exact trigger.

**The root cause:**
Python's `datetime.weekday()` returns 6 for Sunday. The streak-increment
branch requires `days_since_last == 1` **and** `today.weekday() != 6`. So
even when a user listens on consecutive calendar days, if today happens to
be a Sunday, the condition is false, and execution falls through to the
`else` branch, which resets the streak to 1 instead of incrementing it. The
weekday check has no legitimate purpose in this logic — the stated rule is
purely about consecutive calendar days, not which day of the week it is.

**My fix and side-effect check:**
Removed the `and today.weekday() != 6` clause entirely, leaving:
```python
elif days_since_last == 1:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
I re-ran the reproduction steps for a Sunday listen (12 → 13, correct) and
also verified Monday–Saturday transitions still increment correctly (they
were unaffected by this bug already, since `weekday() != 6` was true for
them). The skip-a-day reset case (`days_since_last > 1`) is untouched.

---

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:**
In `flask shell`, I recorded a song sharer's notification count via
`get_notifications()` (1), called `rate_song()` as a different user, confirmed
the `Rating` saved via `.to_dict()`, then re-checked notifications. The count
was still 1 — no notification was created for the rating.

**How I found the root cause:**
I compared `add_to_playlist()` and `rate_song()` in `notification_service.py`
side by side, since both should notify the song's original sharer.
`add_to_playlist()` ends with a check and a call to `create_notification()`
after the playlist update commits. `rate_song()` ends immediately after
`db.session.commit()` — there is no corresponding call to
`create_notification()` anywhere in the function.

**The root cause:**
`rate_song()` never calls `create_notification()`. The notification-creation
step was implemented for the playlist-add flow but was simply never added
to the rating flow — the rating itself is saved correctly, but nothing in
the code path triggers a notification for it.

**My fix and side-effect check:**
Added a notification call at the end of `rate_song()`, mirroring the pattern
in `add_to_playlist()` (only notify if the rater isn't the song's own sharer):
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} stars.",
    )
```
placed after `db.session.commit()`. I re-ran the reproduction steps and
confirmed the notification count now increases by 1 after rating. I also
checked that rating your own shared song does not create a self-notification
(matching the existing `add_to_playlist` behavior), and confirmed existing
playlist-add notifications still fire unchanged.

---

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**
In `flask shell`, I compared `len(playlist.songs)` (7, via the model
relationship) against `get_playlist_songs(playlist.id)` (6). The counts
didn't match — the service was dropping one song.

**How I found the root cause:**
I opened `playlist_service.py` and read `get_playlist_songs()`. The query
itself correctly selects and orders all matching songs by `position`. The
final line, however, is:
```python
return [song.to_dict() for song in songs[:-1]]
```
`songs[:-1]` slices off the last element of the list before returning it.
Since songs are ordered ascending by position, the last element is always
the most recently added song — matching darius's exact report that the
newest song is always the one missing.

**The root cause:**
The function queries all songs correctly but then explicitly truncates the
last item from the list with `songs[:-1]` before converting to dicts and
returning. This silently drops the most recently added song every time,
regardless of playlist size — even a 1-song playlist would return an empty
list.

**My fix and side-effect check:**
Removed the slice, returning all songs:
```python
return [song.to_dict() for song in songs]
```
Re-ran the reproduction check and confirmed `get_playlist_songs()` now
returns 7 songs, matching `len(playlist.songs)`. I also checked
`get_playlist()` and `get_user_playlists()` — neither calls
`get_playlist_songs()`, so they were unaffected by this bug and unaffected
by the fix.

## AI Usage

I used Claude throughout this project for codebase navigation, hypothesis
testing, and understanding SQLAlchemy/Flask patterns I hadn't worked with
before. Specific ways I used it:

**Codebase orientation:** Before touching any bug, I pasted `README.md`,
`models.py`, and all files in `routes/` and `services/` and asked Claude to
help me understand the data flow — specifically how a rating and a playlist
addition each move from route to service. This became the basis of my
codebase map above.

**Reproducing bugs before fixing:** For each bug, Claude gave me `flask shell`
snippets to trigger the reported behavior directly (calling service functions
with controlled inputs) rather than guessing from reading code alone. This
caught real, measurable before/after differences for Issues #1, #4, and #5.

**Root cause tracing:** For Issues #1, #4, and #5, Claude pointed me toward
the exact lines to compare (e.g., `add_to_playlist()` vs. `rate_song()` for
Issue #4; the `[:-1]` slice for Issue #5) but I read the code myself and
confirmed the logic before writing the fix, rather than accepting a diagnosis
without checking it against the actual file contents.

**Where I verified over AI's initial suggestion:** The Issue #3 investigation
above is the main example. In general, I treated Claude's suggestions as
hypotheses to test in `flask shell` rather than conclusions, which is what
caught the version-drift issue instead of me implementing an unnecessary fix.