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


The likely issue is the RECENT_THRESHOLD = timedelta(hours=24) is simply too generous a window for a feature called "Listening Now." A period of 24 hours in the video is simply a correct period, however, playing at something 2+ hours ago is too old to create a "Listening Now" feature. I have moved RECENT_THRESHOLD to only one hour, so it removed the bugs.

## Issue 4
I got notified when a friend added my song to a playlist but not when they rated it.

This exist in `notification_service.py` 

The gap: unlike add_to_playlist, rate_song never calls create_notification. It has all the ingredients to do so (song.shared_by is right there on the song object it already fetched), but simply doesn't check who shared the song or notify them. This is exactly issue #4 from your bug list — rating a friend's song silently succeeds but the sharer never finds out, while adding to a playlist correctly notifies them.

The fix would mirror the playlist pattern: after saving the rating, if song.shared_by != user_id, call create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.").

Bug reproduction:

cd "d:/nhat_minh/Github_clone/ai201-project5-mixtape-starter" && python -c "
from app import create_app, db
from models import User, Song
from services.notification_service import rate_song, get_notifications

app = create_app({'TESTING': True, 'SQLALCHEMY_DATABASE_URI': 'sqlite:///:memory:'})
with app.app_context():
    db.create_all()

    sharer = User(username='sharer', email='sharer@example.com')
    rater = User(username='rater', email='rater@example.com')
    db.session.add_all([sharer, rater])
    db.session.commit()

    song = Song(title='Great Song', artist='Artist', shared_by=sharer.id)
    db.session.add(song)
    db.session.commit()

    # Case 1: someone else rates the song -> sharer should be notified
    rate_song(rater.id, song.id, 5)
    notifs = get_notifications(sharer.id)
    print('sharer notifications after rating by someone else:', len(notifs))
    for n in notifs:
        print(' ', n['type'], '-', n['body'])

    # Case 2: sharer rates their own song -> should NOT self-notify
    rate_song(sharer.id, song.id, 4)
    notifs = get_notifications(sharer.id)
    print('sharer notifications after self-rating:', len(notifs))

    # Case 3: re-rating (update) by the same non-sharer user -> notifies again (matches update semantics)
    rate_song(rater.id, song.id, 3)
    notifs = get_notifications(sharer.id)
    print('sharer notifications after re-rating (update):', len(notifs))
"


# AI Usage
- I use AI like Claude to read through the code with me and graph the relationship between the code. I tries to verify the relationship in the graph, which calls and 