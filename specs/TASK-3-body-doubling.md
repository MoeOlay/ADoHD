# HDoAD — Run 3: Body doubling

## Goal
Build body doubling, the feature a chat app can't copy: real people working at the same time. Runs 1 and 2 are merged into `master`. Read every file in `.kiro/steering/` before the first step, and obey `adhd-ux.md` and `learning.md` in every step.

Four rules hold throughout:
- Friends only. There is never stranger-to-stranger contact.
- Presence, text, and one-tap check-ins. No voice calls and no video in this run.
- **The app stays in Expo Go.** The developer tests on an iPhone from Windows with no paid Apple Developer account. That means no development build, no custom native modules, and no remote push notifications (Expo Go doesn't support push). Local notifications are fine.
- When this spec is ambiguous, choose the option that adds the least friction for the end user, and record the choice in an ADR.

## Stack additions
- **Presence and session state:** Supabase Realtime (Presence plus broadcast) through `supabase-js`, which is pure JavaScript and works in Expo Go. No new vendor.
- **Reaching a friend whose app is closed:** there's no push, so use React Native's built-in `Share` API to send a text or iMessage containing a 6-character session code. Anything addressed to a user (invites, challenge results, stakes owed) is also stored server-side and shows up in an in-app **Inbox** the next time they open the app.
- The test command for every step remains `python scripts/check.py`.

## Requirements

### B1. Friends
- Add a friend by 6-character friend code, sent through the share sheet or typed in. The other person must accept.
- Each friend row has one-tap remove and one-tap block.
- Anonymous users are asked to upgrade to email only at the moment they first add a friend (not earlier). All existing data is kept.
- New tables `friendships` and `blocks`, with RLS. A blocked user can never see the blocker's presence, invites, or sessions.

### B2. Presence
- Each user publishes one status to friends only: `available`, `focusing`, or `away`.
- While focusing, friends see "focusing on a task" by default. The real task title is shown only if the user turns on sharing for that session.
- Presence disappears within 60 s of the app going to the background (Presence timeout).

### B3. Sessions
Entry points: a "Body double" button on each Today card, a fourth option in the Stuck sheet ("Get a body double"), and the home screen.

**Starting a session**
1. Pick a length: 25 min (default) or 50 min. 50 min is the hard cap.
2. Type a one-line intention ("start the outline"). In a friend session, both people see each other's intention by default. A one-tap toggle hides yours.
3. Choose one of three ways to bring a friend in:
   - **Invite:** friends who are online right now get a live in-app invite; the first to accept joins.
   - **Text a friend:** opens the share sheet with a message like "Body double with me? Code: K7Q2MD", for friends who aren't in the app.
   - **Open room:** friends see it and can drop in.
4. Joining by code: Home → "Join with code" (6 characters, case-insensitive). A code expires when the session ends.

**During the session**
- A shared countdown uses the server start timestamp, so every device shows the same time.
- A one-tap check-in strip: "started", "still going", "done".
- Short text messages, 140 characters max.
- One preset reaction per check-in (👍 🔥 💪).
- A Leave button is always visible.
- A local notification fires at 5 minutes left and at the end, so the user can leave the app during the session.

**Ending the session**
- Each person answers "Did you start?" with Yes, Sort of, or No. All three answers get a neutral reply; none shows a judgment.
- A finished intention can be marked Done on the thread with one tap.

**Session state machine:** `pending → active → ended`, plus `cancelled`. The state is stored server-side and enforced by a scheduled job. Clients render it; they never decide it.

**Idle handling**
- WHEN every participant has been backgrounded for 5 minutes, THE SYSTEM SHALL end the session.
- No session can outlive its timer by more than 1 minute.

### B4. When no friend is available: co-focus count + AI body double
The user runs a solo session with the same flow (length, intention, countdown, end debrief), plus two companions. Both are on by default, and each has a toggle on the session screen.

1. **Co-focus count**
   - "12 people focusing right now": a live count of app users currently in any session.
   - No names, no chat, no identity. Only the aggregate number leaves the server.
2. **AI body double**
   - Uses the Run 2 `Coach` interface and output validator, in a new `checkin` mode.
   - Check-ins:
     - **Start:** confirms the intention back in one line.
     - **Midpoint:** "Still on <intention>?" with one-tap replies: On it, Drifted, Switched.
     - **5 minutes before the end:** a heads-up.
     - **End:** the "Did you start?" debrief.
   - Frequency is chosen per session: off, midpoint only (default), or every 10 minutes.
   - Check-ins arrive in the session screen. If the app is in the background, they arrive as local notifications scheduled at session start and cancelled if the user leaves.
   - Text by default. Voice is optional (STT in, `expo-speech` out).
   - **"Drifted"** gets one line plus one option: either "make it smaller" (the S1 flow) or "keep going". Never a lecture.
   - The AI never adds tasks, never extends the session, and never sends a check-in after the user leaves.
   - **Handoff to a friend:** if a friend becomes available during a solo session, show one dismissible banner, "<Friend> is free — invite?" At most once per session.

### B5. External motivators (all opt-in; never leaderboards or history)
**Urgency**
- The shared countdown in every session.
- Challenge and stake deadlines get the Run 2 due reminders (1 h and 15 min before), as local notifications on each participant's own phone.

**Competition**
- **Challenge:** two friends each name one task and a deadline within the next 12 hours. Both see each other's check-off live while the app is open, and in the Inbox otherwise. At the deadline the challenge closes. The result card shows both outcomes with neutral copy, for 24 h only.
- **Race to start:** an opt-in toggle when starting a friend session. The first person to tap "started" gets a 🏁 shown for that session only. Nothing is stored afterwards.

**Stakes**
- **Setting a stake:** on any thread or challenge, the user writes their own consequence ("if I don't, I owe <friend> a coffee") and picks one friend as witness.
  - It must be set at least 10 minutes before the deadline (commit while motivated).
  - It can't be edited after it's set.
  - It can be cancelled with a one-line reason, which the witness sees in their Inbox.
- **At the deadline:**
  - The *user* self-reports Done / Not done. There's no surveillance.
  - On Not done, the witness's Inbox shows "<user> owes you: <stake>", and the witness marks it settled when paid.
  - No money moves through the app, ever.

**Spite / trash talk**
- Preset taunts only: "⏰ tick tock", "👀 waiting on you", "😤 already done".
- Enabled per friendship only when *both* friends switch it on.
- Usable inside challenges and sessions. At most 3 per friend per day.
- No free-text taunts.

### B6. Inbox
- One screen listing everything addressed to the user, newest first: friend requests, invites, challenge updates, stakes, and taunts.
- Its badge shows a dot, never a number.
- Items expire on their own (an invite when its session ends; a result card after 24 h).
- It's updated live through Realtime while the app is open.

### B7. Safety and abuse limits
- At most 5 invites per hour per user, and at most 3 open rooms per day.
- Report a friend: saves a `reports` row and auto-blocks them.
- Message text passes the banned-words filter plus a basic profanity filter.
- Session codes are random, single-session, and rate-limited: at most 10 failed join attempts per hour per user.

### B8. Solo testing tool
`scripts/fake_friend.py` signs in as a test account and lets the developer test body doubling with one phone. It:
- accepts friend requests;
- appears online;
- accepts invites or joins by code;
- posts check-ins at set times;
- replies to challenges.

It reads its environment from `~/.hdoad/` and defaults to the **test** project.

## Non-goals
Voice calls, video, remote push notifications, development builds, stranger pairing or matching queues, public rooms, leaderboards, streaks, app lockouts, calendar / nutrition / gym integrations, production deployment, pushing to the git remote.

## Acceptance criteria
- WHEN user B has blocked user A, THE SYSTEM SHALL never deliver A's presence, invites, or sessions to B, or B's to A. (RLS test plus a Realtime integration test with two clients.)
- WHEN two clients join a session, THE SYSTEM SHALL show both the same remaining time, within 1 s.
- WHEN all participants are backgrounded for 5 minutes, THE SYSTEM SHALL end the session. (Fake-timer test.)
- WHEN a user joins with a valid code before the session ends, THE SYSTEM SHALL add them to it; WHEN the code belongs to an ended session, THE SYSTEM SHALL reject it with neutral copy.
- WHEN a user makes an 11th failed code attempt in an hour, THE SYSTEM SHALL reject it.
- WHEN a user sends a 6th invite within an hour, THE SYSTEM SHALL reject it with neutral copy.
- WHEN no friend is available, THE SYSTEM SHALL show the co-focus count and the solo option, and SHALL expose no other user's identity.
- WHEN the AI body double is set to "midpoint only", THE SYSTEM SHALL deliver exactly one mid-session check-in, and SHALL cancel any pending check-in notifications when the user leaves. (Fake-timer test.)
- WHEN a stake is created less than 10 minutes before its deadline, THE SYSTEM SHALL reject it with neutral copy.
- WHEN the user reports Not done on a staked item, THE SYSTEM SHALL show the stake in the witness's Inbox only, never to anyone else.
- WHEN only one friend has enabled trash talk, THE SYSTEM SHALL not allow taunts in either direction.
- WHEN a user sends a 4th taunt to the same friend in one day, THE SYSTEM SHALL reject it.
- WHEN a race-to-start session ends, THE SYSTEM SHALL keep no record of who won.
- WHEN a session reaches 50 minutes, THE SYSTEM SHALL end it.
- WHEN the Inbox has unread items, its badge SHALL show a dot and no count.
- The session state machine passes fast-check property tests: no illegal transitions, and every session ends.
- `python scripts/check.py` exits 0.

## Final step
- Run the same audit as Run 1 R7.
- Write `docs/RUN3-REVIEW.md`. It must include how to test body doubling using one iPhone plus `scripts/fake_friend.py`, and using a friend's phone running Expo Go on the same Wi-Fi.
- Write `docs/learning/QUIZ-run3.md`.

## Approval gates
`force_approval: true` for any step that calls a real AI model, writes to `hdoad-dev` (including `migrate_dev.py`), or pushes to the git remote. Testing against `hdoad-test` needs no approval. The human creates every account and pastes every key.
