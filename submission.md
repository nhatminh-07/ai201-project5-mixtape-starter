# Analysis of models

models.py defining notification services for a user-defined model itself.

There are significant files and models here:

Classes names:

User: id, username, streak, last_listened at
Tag: category tag used for a song
Song: (database sigma)
ListeningEvent: an event people created (--> registered for the server)
Rating: each rating event in a database
Playlist
Notification

There are 5 significant bugs that exist in README.md and I will write a map of the model itself.

The routes are split to get to process the URL Flask tokens, and these calls immediately the functions in the service/, which implement different classes in the file itself in Python.

## Issue 1
My listening streak keeps resetting.

The issues that exists: `streak_service.py`.

The system assume our services local time and for the purpose of assignments, they are to be set in UTC. The code here gets the issue:

if days_since_last == 0:
        # Already updated today — no change needed
        return
    elif days_since_last == 1 and today.weekday() != 6:
        user.listening_streak += 1
    else:
        user.listening_streak = 1

If today.weekday() != 6, it means it is not Sunday--> add into the streak. But when you listen to the song on Sunday, the elif layer is false, so by the time you listen on Sunday, it returns to 1.

## Issue 2
Friends Listening Now shows people from yesterday 

Files: `feed_service.py`.

Here is the reproduction of the bug:

kenji friends: ['aaliya', 'nova']
kenji feed:  nova — "Midnight Drive" — listened_at = 2026-07-06T03:43:25  (2 hours ago)

darius feed: simone — listened 15 min ago (legitimately recent)
             nova    — "Midnight Drive" — listened 2 hours ago  ← same stale entry


The likely issue is the RECENT_THRESHOLD = timedelta(hours=24) is simply too generous a window for a feature called "Listening Now." A period of 24 hours in the video is simply a correct period, however, playing at something 2+ hours ago is too old to create a "Listening Now" feature. I have moved RECENT_THRESHOLD to only one hour, 

# AI Usage
- I use AI like Claude to read through the code with me and graph the relationship between the code. I tries to verify the relationship in the graph, which calls and 