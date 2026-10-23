# How to run these specs

Three Task Runner runs, in order. One run can hold at most 50 steps, so the app is split into three runs. Each run builds on the one before it and gets reviewed before the next one starts.

| Run | File | What you get |
|---|---|---|
| 1 | `TASK-1-foundation-capture.md` | The notebook: voice/text brain dump → tagged threads, Today (max 3), offline-safe capture |
| 2 | `TASK-2-stuck-reward-review.md` | Stuck flow (smaller / break / why), rewards, 3-card review, morning alarm (iPhone Clock + Shortcut), 1 h / 15 min due reminders, body check, novelty, fresh start |
| 3 | `TASK-3-body-doubling.md` | Friends, presence, shared sessions with join codes, co-focus count + AI body double, challenges, race-to-start, stakes, trash talk, Inbox, a fake-friend test script |

All three runs stay in **Expo Go**, so everything works on your iPhone for $0. Left out on purpose: voice calls in sessions, and push notifications from friends. Both need a $99/yr Apple Developer account, and you can add them later.

## Before Run 1 (one time)
1. **Create two free Supabase projects** at supabase.com: `hdoad-dev` and `hdoad-test`. The free plan allows exactly 2. In each one, turn on **Anonymous sign-ins** (Authentication settings). Save each project's database password when you create it.
2. **Make a folder outside the repo** (`C:\Users\brave\.hdoad\`, which is `~/.hdoad/` in Git Bash) and put two files in it.

   `.env` (dev project):
   ```
   SUPABASE_URL=
   SUPABASE_PUBLISHABLE_KEY=
   SUPABASE_SECRET_KEY=
   SUPABASE_DB_URL=
   ANTHROPIC_API_KEY=
   OPENAI_API_KEY=
   ```
   `.env.test` (test project):
   ```
   SUPABASE_URL=
   SUPABASE_PUBLISHABLE_KEY=
   SUPABASE_SECRET_KEY=
   SUPABASE_DB_URL=
   ```

   Where to find the values:
   - The URL and keys are on each project's API keys page. Older dashboards call the keys "anon" and "service_role".
   - The DB URL is the connection string under the project's **Connect** button, with your database password filled in.
   - The two AI keys can stay blank for now. Tests use fakes, and only real-AI runs need them.

   The files live outside the repo for two reasons: Crew works in a separate copy of the repo that wouldn't include them, and secrets outside the repo can't get committed by accident.
3. Install Python 3.12, Node 22+, and `uv`.
4. Put the Expo Go app on your iPhone.
5. **Keep the specs inside your repo,** so they're saved with your code and the `kirocrew run` command below can find them.
   1. In File Explorer, open `C:\Users\brave\HDoAD` and make a new folder in it called `specs`.
   2. Drag the four spec files (this README and the three `TASK-` files) from your Downloads folder into `specs`. If Downloads has several copies with names like `(1)` or `(2)`, use the **newest** one of each, and rename it back to the plain name.
   3. In Git Bash, save and upload it:
      ```
      cd ~/HDoAD
      git add specs
      git commit -m "Add Kiro specs"
      git push
      ```

## Each run
```
kirocrew run specs/TASK-1-foundation-capture.md
```
You can also upload the file in the Crew dashboard's **From Spec** tab.

Crew works on its own branch, `kirocrew/task/<id>`, in a separate worktree, so your `master` checkout is untouched. When the run finishes:
1. Open `docs/RUN1-REVIEW.md` and try the listed things on your phone.
2. Take `docs/learning/QUIZ-run1.md`.
3. Skim `docs/learning/LOG.md`.
4. If everything checks out, merge the branch into `master` and push.

Then start the next run.

## Approval gates
Crew pauses for your approval before it uses real AI keys, changes your dev database, or pushes to GitHub. You create every account (Anthropic, OpenAI, Supabase) yourself and put the keys in `~/.hdoad/`. The test project gets wiped and rebuilt on every check, so never put real data in `hdoad-test`.

Free Supabase projects pause after a week of no activity. If checks suddenly say the test project is unreachable, restore it from the Supabase dashboard.
