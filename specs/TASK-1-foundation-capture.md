# HDoAD — Run 1: Foundation, capture, and the notebook

## Goal
Build the first end-to-end slice of HDoAD, an organization app for people with ADHD. A user opens the app, brain-dumps by voice or text in diary style, and the app turns what they said into tagged threads (focus tasks, don't-forget items, carryovers) without asking a single question at capture time. Every thread gets a red / yellow / green priority color. The Today screen shows at most 3 items, highest priority first, and a separate priority list shows everything. Everything runs locally. Nothing is deployed and nothing is pushed to a remote.

When this spec is ambiguous, choose the option that adds the least friction for the end user, and record the choice in an ADR.

## Product context (applies to every step)
HDoAD digitizes a proven pen-and-notebook method. Each morning the user writes a diary-style entry, and inline symbols turn sentences into tasks:

- "Today I want to teach myself the course, but first I need to review more about how loops work in Javascript, especially the for loop [!]"
- "Today I can't forget that I have training at 4 PM [<>]"
- "Today I want to go for a longer walk to make up for what I couldn't do yesterday [-]"

Later, the user writes follow-ups that start with the same symbol, and they attach to the same thread:

- "[!] I just understood the for loops better but I think it's better not to continue the classes, and to practice more. to be well-prepared."
- "[<>] During the training, they gave me a lot of useful information, such as: instructor number: 123456789; Monthly fee: €56 (1x per week), €100 (2x per week); how it will work from now on, where I meet people."
- "[-] Well, I couldn't do the walk today either, I'll have to wait until tomorrow"

A task is a **thread** (its origin line plus every follow-up), not a ticket with a status field.

The hard UX rules live in `.kiro/steering/adhd-ux.md` (created in step 1 from Appendix A). They are constraints, not suggestions. Every step must comply.

## Stack (decided; do not substitute)
- Monorepo inside the existing git repository (default branch `master`).
- Mobile: React Native with Expo (latest stable SDK), TypeScript in strict mode, Expo Router. The app must keep working in **Expo Go** for this entire run, so use only Expo Go–compatible modules (e.g. `expo-sqlite`, `expo-audio`, `expo-secure-store`).
- Backend: FastAPI on Python 3.12, async, Pydantic v2, dependencies managed with `uv`.
- Data, auth, realtime: Supabase (Postgres, Auth, Realtime). Local development uses the Supabase CLI (`npx supabase start`, which needs Docker). Enable anonymous sign-ins in `supabase/config.toml`. All schema changes go through migrations in `supabase/migrations/`.
- AI: the Anthropic Claude API parses entries, behind a `Parser` interface. OpenAI speech-to-text transcribes voice, behind a `SpeechToText` interface. Model names come from env vars (`PARSER_MODEL`, default `claude-haiku-5-5`). Both interfaces have deterministic fakes, and **every test uses the fakes**. Real providers are called only when `USE_REAL_AI=1`.
- The developer's machine is **Windows 11** (Git Bash). Every script must be cross-platform: Python scripts or npm scripts. No Makefiles, no bash-only scripts.

## Repo layout
```
apps/mobile/          Expo app
services/api/         FastAPI app (src layout) + tests/
supabase/             config.toml, migrations/, seed.sql, RLS tests
evals/tagging/        golden set + eval runner
scripts/check.py      the single quality gate
docs/adr/             architecture decision records
docs/learning/        learning log + quizzes
.kiro/steering/       steering files
```

## Quality gate (the test command for every step)
`python scripts/check.py` must exit 0. It runs these in order and stops at the first failure:
1. Backend: `ruff check`, `ruff format --check`, `mypy --strict`, `pytest` (unit + integration).
2. Supabase: migrations apply cleanly to the local stack, and the RLS tests pass.
3. Mobile: `tsc --noEmit`, ESLint (including a React Native accessibility plugin), Jest with React Native Testing Library and `fast-check` property tests, and the **copy lint** (banned words from Appendix A, run over every user-facing string).
4. Contract: regenerate TypeScript types from FastAPI's OpenAPI schema and fail if they differ from the committed types.

Integration tests run against the local Supabase stack. If Docker is not running, `check.py` must fail with a one-line message telling the developer to start Docker Desktop. It must never skip silently.

## Requirements

### R1. Foundation and steering
- Create `.kiro/steering/adhd-ux.md` and `.kiro/steering/learning.md` with the content in Appendix A, verbatim.
- Create `.kiro/steering/product.md` from "Product context", `tech.md` from "Stack" and "Quality gate", and `structure.md` from "Repo layout". Keep them short; they are rules for the agent, not docs for humans.
- Create `scripts/check.py`, `.env.example` files for the API and mobile app (placeholders only, never real secrets), and a GitHub Actions workflow that runs `python scripts/check.py`. Extend the existing `.gitignore` only if something new needs ignoring.
- Write ADR 0001 recording the stack decisions above and the reason for each.

### R2. Data model (Supabase migrations)
Every table has `id`, `user_id uuid references auth.users`, `created_at`, row-level security enabled, and a policy of `user_id = auth.uid()`.

- `profiles`: display_name, timezone, morning_window_start / morning_window_end (default 06:00–10:00 local), quiet_hours_start / quiet_hours_end (default 22:00–07:00), daily_notification_cap (default 3), onboarded_at.
- `symbols`: glyph (e.g. `[!]`), label, kind (`focus` | `time_bound` | `carryover` | `note`), sort_order. Seed every new user with `[!]` = focus, `[<>]` = time_bound ("don't forget"), `[-]` = carryover. The user can edit, add, and remove symbols.
- `entries`: raw_text, source (`voice` | `text`), audio_path, local_date, client_id (idempotency key), parse_status (`pending` | `parsed` | `failed`), parsed_at.
- `threads`: symbol_id, title, origin_entry_id, origin_excerpt, status (`open` | `done` | `later` | `archived`), due_at, due_source (`user` | `ai`), due_confidence (0–1), importance (`high` | `normal`, default `normal`), importance_source (`user` | `ai`), priority_override (`red` | `yellow` | `green` | null), activation_energy (1 = low effort … 3 = high), visible_from (date), last_touched_at, completed_at, reward_text.
- `thread_notes`: thread_id, entry_id, text, kind (`followup` | `info`).
- `clarifications`: thread_id, field, question, options (jsonb), status (`open` | `answered` | `expired`), answer. Created in this run; shown to the user in Run 2.
- `app_events`: name, props (jsonb). Used later for the return-after-gap metric.

Enable Realtime on `entries`, `threads`, and `thread_notes`.

### R3. Capture API
- `POST /v1/entries` accepts text or an audio file plus a `client_id`. It stores the raw entry (and audio, in Supabase Storage) and returns `201` without waiting for AI. Re-posting the same `client_id` returns the existing entry and creates no duplicate.
- Every request verifies the Supabase JWT. Anonymous users are allowed.
- Parsing runs as a background job: (voice → transcript) → the parser receives the raw text, the user's symbols, and the user's open threads (id, title, symbol) → it returns segments. Each segment is one of `new_thread`, `followup(thread_id)`, or `info(thread_id, bullets)`, with an excerpt, a symbol, an optional due_at plus confidence, an importance guess plus confidence (high when the user's own words signal it, e.g. "can't forget", "need to", "important", "has to"), and an activation_energy guess. Results are written to `threads` / `thread_notes`, and the client sees them through Realtime.
- Parsing rules:
  - A symbol the user typed always beats the AI's guess.
  - A line that *starts* with a symbol (e.g. "[!] I just understood…") is a follow-up to the most recent open thread with that symbol, unless the text clearly names a different one.
  - Any field with confidence below 0.6 is left empty, and one `clarification` row is created for it. The user is never asked anything at capture time.
  - A segment whose excerpt is not an exact substring of the raw text is dropped and logged. The parser never creates a thread for something the user didn't say.
  - Text that is only feelings or reflection, with no intention, creates no thread. It stays in the notebook as written.
- Threads created from an entry saved **outside** the user's morning window get `visible_from` = the next morning, unless they have a due_at today; those show on Today right away. (Mornings are for new tasks; the rest of the day is for follow-ups, and anything new waits for tomorrow unless it's due.)
- If parsing fails, the entry stays saved with `parse_status = failed` and its raw text stays visible. Retry with backoff, 3 attempts.

### R4. Parser quality
- Hypothesis property tests for the validator: excerpt-is-substring, typed-symbol override, confidence below threshold → clarification, feelings-only → zero threads.
- `evals/tagging/golden.jsonl` contains the six example lines from "Product context" with their expected output, plus at least 30 more realistic diary-style lines covering: several tasks in one sentence, vague times ("after class", "tonight"), feelings with no task, follow-ups without a symbol, info bullets, and slang/typos.
- `python -m evals.tagging` runs the real parser only when `USE_REAL_AI=1` and reports symbol accuracy, follow-up linking accuracy, invented-thread count (target: 0), and p50/p95 latency. It is **not** part of `check.py`.

### R5. Mobile app
- **First launch:** anonymous sign-in, so the first capture happens before any account exists. Settings offers an upgrade to email OTP that keeps all data.
- **Onboarding:** at most 3 screens and 60 seconds. (1) Your symbols: prefilled defaults, editable. (2) Morning window and quiet hours. (3) Your first brain dump, right now.
- **Home screen:** during the morning window, home is the capture screen, framed for new tasks ("What's today?"). Outside it, home is Today, with a capture button reachable in one tap from anywhere, framed for follow-ups ("Add to a thread, or save for tomorrow"). Both kinds of capture work at any hour; only the framing changes.
- **Capture screen:** one large record button and a text field visible together. Saving needs zero extra fields. Saving is optimistic: the entry appears instantly, is persisted in a local offline queue (`expo-sqlite`; survives app kill and airplane mode), and syncs when online.
- **Notebook view:** entries grouped by day, diary-style, with symbol chips inline wherever a thread was detected. Tapping a chip lets the user change the symbol, change the due time, or unlink it, each in one tap, with no confirmation dialogs.
- **Thread view:** the origin line verbatim, then follow-ups and info bullets in chronological order. The user can add a follow-up here at any time of day.
- **Priority colors** (a pure function, unit- and property-tested):
  - urgent = due_at is within the next 24 h, or has passed while the thread is still open;
  - important = importance is `high`;
  - **red = urgent and important; yellow = exactly one of them; green = neither.**
  - A `priority_override` always wins.
  - Each color also carries a shape and a label, so color never carries meaning alone: red ● "Now", yellow ▲ "Soon", green ■ "Whenever".
  - Tapping the priority chip cycles red → yellow → green → auto, in one tap each.
- **Red cap:** when a user action (override to red, or marking something important) would make a 4th open thread red, show the current reds and ask which one drops to yellow, in one tap. Cancel leaves the new thread yellow. Reds that appear because time passed never trigger the prompt.
- **Today view:** at most 3 open threads whose `visible_from` is today or earlier, ordered by red, then yellow, then green; within a color, earliest due_at, then lowest activation_energy, then oldest last_touched_at. A "Swap" control replaces one item with the next candidate. Each item has Done (one tap, with a 5-second undo) and an "I'm stuck" button that shows a placeholder until Run 2.
  - Items due today show a live countdown ("1h 12m left").
  - Once the due time passes, the card reads "Time's up — move it?" with one-tap Later today / Tomorrow / Drop the time. Never "overdue" and never late styling.
- **Priority list:** a separate screen, one tap from Today, showing every open thread grouped Red / Yellow / Green. Red is expanded; yellow and green start collapsed. Each row can be recolored, marked Done, or opened.
- **Accessibility:** respects system font scaling, reduce motion, and dark mode; touch targets at least 44 pt; WCAG AA contrast.

### R6. Learning artifacts
Follow `.kiro/steering/learning.md` in every step. At the end of the run, write `docs/learning/QUIZ-run1.md`.

### R7. Final audit (last step of the run)
- Check every screen and every user-facing string against `.kiro/steering/adhd-ux.md`, and check that every acceptance criterion below maps to at least one real test. Fix whatever fails.
- Write `docs/RUN1-REVIEW.md`: at most 10 things for the developer to try by hand on their phone in Expo Go, each with the expected result, plus the exact commands to start the stack (Supabase, API, Expo).

## Non-goals (do not build in this run)
Stuck flow, reward prompt, daily review screen, notifications, body doubling, friends, calendar / nutrition / gym integrations, streaks / points / badges, stranger matching, app lockouts, deployment to any hosted environment, pushing to the git remote.

## Acceptance criteria
- WHEN the user saves a text entry with no network, THE SYSTEM SHALL show it in the notebook immediately AND sync it within 10 s of reconnecting with no duplicate rows. (Jest test of the queue + pytest test of the `client_id` idempotency.)
- WHEN `POST /v1/entries` is called, THE SYSTEM SHALL return 201 before parsing completes. (pytest with a deliberately slow fake parser.)
- WHEN the raw text contains "[<>]", THE SYSTEM SHALL tag the resulting thread time_bound regardless of the parser's guess.
- WHEN a line starts with "[!]" and an open focus thread exists, THE SYSTEM SHALL attach the line as a follow-up to that thread.
- WHEN the parser returns a segment whose excerpt is not in the raw text, THE SYSTEM SHALL drop it and create no thread.
- WHEN a field's confidence is below 0.6, THE SYSTEM SHALL leave the field empty and create exactly one open clarification for it.
- WHEN an entry contains only feelings and no intention, THE SYSTEM SHALL create zero threads.
- WHEN an entry is saved outside the morning window, THE SYSTEM SHALL keep its new threads off Today until the next morning, unless a thread has a due_at today, which shows immediately.
- WHEN a thread is important and due within 24 h, THE SYSTEM SHALL color it red; for exactly one of the two, yellow; for neither, green; and an override SHALL always win. (fast-check property test over all combinations.)
- WHEN a user action would create a 4th red thread, THE SYSTEM SHALL ask which red drops to yellow, and cancelling SHALL leave the new thread yellow.
- WHEN Today has more than 3 candidates, THE SYSTEM SHALL show reds before yellows before greens.
- WHEN a Today item's due time passes, THE SYSTEM SHALL show the "Time's up — move it?" options and no late styling.
- WHEN user A queries any table, THE SYSTEM SHALL return none of user B's rows. (One RLS test per table.)
- WHEN Today renders with 10 eligible open threads, THE SYSTEM SHALL show exactly 3.
- WHEN any user-facing string contains a banned word from Appendix A, THE copy lint SHALL fail.
- `python scripts/check.py` exits 0 on a clean checkout with Docker running.

## Approval gates
Any step that would call a paid API with real keys, link a hosted Supabase project, or push to a git remote: `force_approval: true`. The human creates all third-party accounts and pastes keys into `.env` themselves; the agent never creates accounts.

---

## Appendix A — steering files (write verbatim)

### `.kiro/steering/adhd-ux.md`
```markdown
# ADHD UX rules (hard constraints)

Every rule exists because a specific failure makes ADHD users abandon organization apps.

1. Capture costs at most 1 tap from opening the app and has zero required fields. Never ask a question at capture time. (Thoughts are lost before a form is filled.)
2. Never show more than 3 actionable items at once on Today. (A long list reads as one wall and causes avoidance.)
3. No streaks, points, badges, levels, overdue counters, or any styling that marks something as late. Red means the user's top priority (urgent + important), never lateness. Reds are capped at 3 by user action so the color keeps meaning. (Decorative rewards habituate; failure ledgers cause shame and abandonment; priority flags inflate until they mean nothing.)
4. Unfinished items roll forward silently. Items untouched for 7 days move to Later without a notification.
5. Copy never shames. Banned in user-facing strings (case-insensitive): "overdue", "failed", "missed", "behind", "lazy", "should have", "you forgot", "streak", "again?". The copy lint enforces this.
6. The app schedules only reminders the user turned on: due-time reminders (1 h and 15 min before, on by default, toggleable globally and per thread), the morning alarm, and anything they explicitly accept. If the user declines a suggestion, don't offer it again for that item that day.
7. Every flow has a visible one-tap exit. No lockouts and no blocking other apps. The single deliberate exception is the morning alarm: it is friction, not force. In the app it stops when the user speaks or types any line (including "skipping today because…"). The OS dismiss control is never blocked, and turning the alarm off in Settings is always one tap and immediate.
8. Voice and text are always both available; neither is required.
9. AI output appears as editable suggestions (chips), never as decisions. The AI never invents tasks the user didn't express.
10. Clarifying questions are batched into one daily review (max 3 cards), each answerable in one tap or skippable. Skipped ones expire silently.
11. Returning after a gap of 3+ days opens a fresh start, not a backlog.
12. Notifications: nudges (review, follow-ups, suggestions) stay within the user's daily cap (default 3) and never fire in quiet hours. Due-time reminders and the morning alarm are separate: they are exempt from the cap because the user set the time. Every notification opens directly to the action it names.
13. Motivation is external and opt-in: real people, real deadlines, competition with friends, stakes the user chose, and novelty. Never animations, points, or leaderboards. Celebrations are brief, optional, and respect reduce motion.
14. Accessibility: system font scaling, reduce motion, dark mode, 44 pt touch targets, WCAG AA contrast, and color never the only carrier of meaning.
15. Novelty comes from variety (rotating prompts, sounds, capture formats), never from rewards. Every novelty feature has an off switch, and nothing repeats so often it goes stale.
```

### `.kiro/steering/learning.md`
```markdown
# Learning artifacts (the developer is learning this stack by reading what you build)

Reader profile: knows Python, data structures and algorithms (queues, stacks, hash maps, trees, graphs, BFS/DFS, topological sort), basic HTML/CSS/JS, and basic git. New to web frameworks, databases, auth, React Native, and realtime systems.

After every step, append an entry to `docs/learning/LOG.md` with, in this order:
1. **Definition** — the core concept of the step in plain technical terms, tied to something the reader already knows (e.g. "the offline queue is a FIFO queue persisted to disk"; "an RLS policy is a predicate evaluated per row, like a filter() applied before any query returns").
2. **Worked example** — a short excerpt from this codebase showing the concept in action.
3. **Why** — why this approach beat the obvious alternative. Why matters more than what.
4. **Rebuild it yourself** — 2–4 bullets on how to recreate this step from scratch.

Record every architecture decision not already fixed by the spec as an ADR in `docs/adr/NNNN-title.md`.

At the end of each run, write `docs/learning/QUIZ-runN.md`: 8 multiple-choice questions on the run's decisions, with 4 options each, answers and one-paragraph explanations at the bottom. Test the reasoning behind decisions, not trivia.
```
