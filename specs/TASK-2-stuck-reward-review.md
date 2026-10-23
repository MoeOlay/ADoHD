# HDoAD — Run 2: Stuck flow, rewards, daily review, and the morning anchor

## Goal
Add everything that happens *after* capture: getting unstuck, finishing with a reward, the 3-card daily review, the morning alarm, due-time reminders, a body check, novelty, and gap-proof rollover. Run 1 is merged into `master`. Read every file in `.kiro/steering/` before the first step, and obey `adhd-ux.md` and `learning.md` in every step.

When this spec is ambiguous, choose the option that adds the least friction for the end user, and record the choice in an ADR.

## Stack and quality gate
Unchanged from Run 1 (see `.kiro/steering/tech.md`). The test command for every step is `python scripts/check.py`. New AI features go behind interfaces with deterministic fakes; tests never call real models.

**The app stays in Expo Go for this entire run.** The developer tests on an iPhone from a Windows PC with no paid Apple Developer account, so a development build isn't possible. Rules:
- Use only Expo Go–compatible modules. No `expo-dev-client`, no custom native modules, no config plugins that need a rebuild, and no remote push (Expo Go doesn't support push).
- Local and scheduled notifications (`expo-notifications`) are allowed.
- If a requirement below turns out to be impossible in Expo Go, build the fallback this spec gives for it and record it in an ADR. Never switch build types.

## Requirements

### S1. Stuck flow
"I'm stuck" on any thread opens a bottom sheet with three equal-weight choices and a visible close button:

1. **Make it smaller**
   - Opens the coach: a short conversation. Text input is always shown, a mic button transcribes voice, and an optional speaker toggle reads replies aloud (`expo-speech`).
   - The coach proposes exactly **one** concrete next step that takes 2 minutes or less ("open the doc and reread the outline", not "work on the paper").
   - The user can accept it, edit it, ask for "even smaller", or tap **"Stop breaking this down"**.
   - An accepted step is saved as a `thread_note` with a new kind, `step`, and is shown on the Today card.
   - After accepting, the user sees an *optional* timer button (2 / 5 / 10 min). The coach never starts a timer itself.
   - "Stop breaking this down" switches the coach out of breakdown mode for this thread for the rest of the day. It then offers, once, "a break" or "remind me why". If the user declines both, the coach says one supportive line and closes. It never sets a reminder or timer.
   - If the user's words signal overwhelm, frustration, or exhaustion, the coach drops breakdown and offers one regulation option (60-second breathing, a short walk, water) before anything else.
2. **I need a break**
   - Choices: 5, 15, 30 min, or "until later".
   - For a timed break, ask once, with one tap: "Want a nudge when it's over?" Only a yes schedules anything.
   - No guilt copy, in any wording.
3. **Remind me why**
   - Shows the thread's origin line verbatim, in the user's own words, plus any reflection notes on the thread.
   - Then offers three one-tap options: "Still matters" (back to Today), "Later", or "Let it go" (archive, with neutral copy).

Coach implementation:
- `Coach` interface, model from `COACH_MODEL` env (default `claude-sonnet-5-5`), with a fake for tests.
- The system prompt embeds the adhd-ux rules. Output is validated server-side:
  - at most 2 sentences per message;
  - at most 1 question per message;
  - at most 1 proposed step;
  - no banned words.
  Invalid output is regenerated once, then replaced with a safe fallback line.
- `evals/coach/` holds at least 20 scripted conversations: normal breakdown, "stop breaking it down", overwhelm, "I don't even know why I'm doing this", and a user who refuses everything. A rubric checks the rules above. Runs only with `USE_REAL_AI=1`; not part of `check.py`.

### S2. Done and reward
- Done is one tap with a 5-second undo (kept from Run 1).
- After Done, show one optional line: "Reward?" with chips made from the user's own past rewards plus a free-text field. Skippable, with no penalty and no repeat prompt.
- A chosen reward is saved to `threads.reward_text` and shown once as "You earned: <reward>". Never points, streaks, or totals.
- Settings → "My rewards" lets the user edit the chip list.

### S3. Daily review (3 cards max)
- Appears as a single dismissible card at the top of Today right after the morning capture is saved: "3 quick ones?"
- It contains at most 3 cards total, pulled in this order:
  1. open `clarifications`, oldest first;
  2. one "Still matters?" card for a thread untouched for 3+ days (options: Yes / Later / Let it go).
- Every card is answerable in one tap or skippable. Skipped clarifications expire silently after 2 days.
- Answering a clarification writes the value back to the thread.

### S4. Morning alarm (friction, not force), built on the phone's own Clock alarm
Off until the user turns it on. Onboarding offers it in one tap.

**The ring is the phone's built-in Clock alarm.** It is already a real alarm that breaks through silent mode. The app connects it to Capture with an iOS Shortcuts personal automation instead of shipping its own alarm.

**Setup**
- Settings → Morning alarm → "Set up with iPhone Clock" is a 3-step guide (≤ 1 screen):
  1. Set your wake-up alarm in the Clock app.
  2. In Shortcuts, create a personal automation: *When [your alarm] is stopped → Open URL*, set to **Run Immediately**.
  3. Paste the link, which has a big Copy button.
- The link is built with `Linking.createURL('/capture', { queryParams: { record: '1', source: 'alarm' } })`, so it's correct for whatever host Expo Go is using.
- The app remembers the last link the user copied. If the current link differs (for example the PC's IP changed), Settings shows a one-line notice, "Your alarm link changed — copy the new one". This is a notice on screen, not a notification.
- The user also enters their alarm time in the app; it's used to schedule the re-rings.
- On Android, skip the Shortcut step. The alarm-time notification below is the way in.

**Opening it**
- Opening Capture via the link with `record=1` starts recording **immediately** (microphone permission is granted during setup).
- The alarm is satisfied the moment an entry with at least one word, voice or text, is saved for today.

**If no line is saved:** re-rings are local notifications.
- Send 6 notifications total, using the default notification sound, 10 minutes apart, then silence for the day.
- The first one goes out 10 minutes after the alarm time if the user confirmed the Shortcut is set up (the Clock alarm already rang). Otherwise it goes out at the alarm time. The guide ends with an "I set up the Shortcut" checkbox that records this.
- Each one deep-links to Capture with `record=1`.
- A saved line or the Settings toggle cancels the rest.
- The app never tries to block or hide any OS control.

**Exits**
- "Skip today" sits on the capture screen. It needs one spoken or typed line ("sick today", "slept in"). That line is saved as a normal entry and ends the alarm.
- Settings → Morning alarm off: one tap, takes effect immediately, cancels any pending re-rings.

**Copy:** never shames. The re-ring notification text rotates from a neutral pool ("Still here when you're ready", "One line is enough").

**Notification policy:** every notification the app schedules, including alarm re-rings, goes through a single `NotificationPolicy` module:
- Nudges (review, follow-ups, suggestions) obey the daily cap and quiet hours, and "never re-offer a declined suggestion today".
- Due reminders (S4b) are exempt from the cap and from quiet hours.
- Property tests (fast-check):
  - nudges never exceed the cap and never fire in quiet hours;
  - alarm re-rings stop once a line is saved and never exceed 6;
  - turning the alarm off cancels everything pending.

### S4b. Urgency: due reminders and "Put a clock on it"
**Due reminders**
- Every open thread with a due_at gets reminders 1 hour and 15 minutes before it, if the due time was set by the user or AI-parsed with confidence 0.6 or higher.
- Settings has a global toggle and editable offsets (default 60 and 15 min). Each thread has a mute.
- Each reminder names the thread and has three actions:
  - **Open** (thread view);
  - **Start** (the step from S1 if one exists, otherwise the thread);
  - **+10 min** (pushes both the reminder and due_at back 10 minutes).
- Use notification action buttons if they work in Expo Go. If not, tapping the notification opens the thread with the same three actions as large buttons at the top.
- Editing or removing due_at reschedules or cancels its reminders. Completing the thread cancels them.

**Put a clock on it**
- A one-tap action on any thread or Today card: "in 1 h", "in 2 h", "by 5 pm", "tonight", or custom.
- It sets due_at (source `user`), which turns on the reminders and, if the thread is important, the red color.

### S5. Body check
- An optional row on the morning Capture screen: Sleep (rough / ok / good), Water ✓, Sunlight ✓, Moved ✓.
- Each is one tap. There is no score, no history graph, no streak, and no prompt if it's skipped.
- Store in a new `body_checks` table (RLS as in Run 1).
- If sleep = rough, Today keeps any red items and fills the remaining slots with activation_energy = 1 items, saying so in one line ("Light day picks").

### S6. Rollover, Later, fresh start, metrics
- A daily job (Supabase `pg_cron` or an API scheduler, decided in an ADR) silently moves threads untouched for 7 days to `later`. No notification.
- **Later** is a calm list screen, reachable from Today, where any item can be pulled back with one tap.
- **Fresh start:** when the app opens after 3+ days with no `app_open` event, show a fresh-start screen with one warm line, the capture button, and a small "Peek at Later" link. Today is recomputed from scratch (max 3, lowest energy). Never show counts of what piled up.
- Log `app_open`, `capture_saved`, `thread_done`, `stuck_choice`, and `gap_return` to `app_events`.
- Add a SQL view plus `scripts/metrics.py` that reports the return-after-gap rate: of users silent for 3+ days, the share who come back and save a capture within 7 days.

### S7. Novelty (variety, never rewards)
Every item below has its own off switch in Settings → Variety.

- **Rotating morning prompt:** one line above the capture field, drawn from a pool of at least 60 (e.g. "What's loud in your head?", "One thing that would make today a win?"). No repeats within 30 days. A shuffle button swaps it.
- **Rotating re-ring copy:** the text of each alarm re-ring and due reminder rotates through a neutral pool. Custom notification sounds need a rebuild, so they're out of scope.
- **Lightning round:** an alternate morning capture mode, offered as a small toggle on the capture screen. A 60-second visible countdown, a brain dump by voice, and it saves automatically when time is up.
- **Format of the day:** at most once a week, the morning screen offers one alternate format with a one-tap "not today": "Just 3 words", "Talk like you're texting a friend", or "Start with the thing you're avoiding". Declining hides it until next week.
- Rotation state lives on the device (no server round-trip) and is deterministic under a seed, so it can be tested.

## Non-goals
Development builds, EAS, custom native modules, remote push notifications, custom notification sounds, body doubling, friends, presence, calendar / nutrition / gym integrations, streaks / points / badges / leaderboards, stranger matching, app lockouts, deployment, pushing to the git remote.

## Acceptance criteria
- WHEN the user taps "Stop breaking this down", THE SYSTEM SHALL schedule no reminder or timer and SHALL not propose another breakdown for that thread that day.
- WHEN the coach's output contains 2 or more proposed steps or a banned word, THE SYSTEM SHALL regenerate or fall back before showing it.
- WHEN the user's message expresses overwhelm, THE coach SHALL offer a regulation option before any task step. (Fake-model test of the routing; eval for the real model.)
- WHEN the user picks a timed break and declines the nudge, THE SYSTEM SHALL schedule nothing.
- WHEN "Remind me why" opens, THE SYSTEM SHALL show the origin excerpt character-for-character.
- WHEN the daily review renders with 7 open clarifications, THE SYSTEM SHALL show exactly 3 cards.
- WHEN the user skips the reward prompt, THE SYSTEM SHALL not show it again for that thread.
- WHEN any nudge is scheduled, THE SYSTEM SHALL respect quiet hours and the daily cap. (Property test.)
- WHEN the app opens via the capture link with `record=1`, THE SYSTEM SHALL start recording with no further tap.
- WHEN the current capture link differs from the last one the user copied, Settings SHALL show the "link changed" notice.
- WHEN an entry with at least one word is saved on an alarm day, THE SYSTEM SHALL cancel all remaining re-rings for that day.
- WHEN no line is saved, THE SYSTEM SHALL re-ring at most 6 times, 10 minutes apart, then stop with no further notification.
- WHEN the user turns the alarm off in Settings, THE SYSTEM SHALL cancel all pending alarm events immediately.
- WHEN a thread has a user-set due_at, THE SYSTEM SHALL schedule reminders 60 and 15 minutes before it, exempt from the daily cap; and SHALL cancel them when the thread is done, muted, or its due_at is removed.
- WHEN the user taps "+10 min" on a reminder, THE SYSTEM SHALL move due_at and both reminders 10 minutes later.
- WHEN the morning prompt rotates, THE SYSTEM SHALL not repeat a prompt within 30 days. (Seeded test.)
- WHEN any novelty feature is switched off, THE SYSTEM SHALL stop showing it immediately.
- WHEN the app opens after 3+ days of silence, THE SYSTEM SHALL show the fresh-start screen and no backlog count.
- WHEN a thread is untouched for 7 days, THE SYSTEM SHALL move it to Later without notifying.
- WHEN sleep = rough is recorded, Today SHALL keep all red items and fill every other slot with activation_energy = 1 items where they exist.
- `python scripts/check.py` exits 0.

## Final step
- Run the same audit as Run 1 R7, against all screens.
- Write `docs/RUN2-REVIEW.md`.
- Write `docs/learning/QUIZ-run2.md` (8 multiple-choice questions).

## Approval gates
`force_approval: true` for real-model calls, writes to `hdoad-dev` (including `migrate_dev.py`), and pushing to the git remote. Testing against `hdoad-test` needs no approval.
