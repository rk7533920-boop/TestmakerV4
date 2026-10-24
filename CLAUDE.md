# Test Maker: notes for Claude (read this first)

This file travels with the app. It records the owner's decisions and working preferences so any Claude session, on any account, can continue the work without asking again. Update it whenever a new decision is made. Keep secrets out (API keys, passwords): the site is hosted from this repository, so every file here may be publicly readable.

## Repository files
- `index.html`: the whole app (single file, plain HTML/CSS/JS, no build step)
- `admin.html`: admin dashboard (Google sign-in for the admin; the admin account is set inside the file and in the Firestore rules)
- `sw.js`, `manifest.webmanifest`, `icon.png`, `CNAME`: offline/PWA and hosting
- `CLAUDE.md`: this file

## How to work with the owner (preferences)
- Make only the requested change. Never rewrite the whole file; use targeted edits.
- For bigger features, discuss the logic and show a visual mockup first. Build only after the owner says go ahead.
- Keep the app clutter-free. The same option must never appear in two places (audit duplicates before adding anything).
- Buttons at the same level or with a similar job have the same shape and size; only the emphasised one differs in colour.
- No key-hint labels inside icons or buttons. Keyboard keys are listed once before a test (OK / Don't show again).
- Flagged-question data is "very very important" (used for future revision). Never put it at risk in any change.
- The owner has an Android/iPad/desktop mix; test on iPad Chrome behaviour when touching inputs and navigation.
- Pop-up messages always have a close (cross) button.
- Prefer free solutions (Firestore quota is limited; no paid notifications, no server for silent renewals, no TTL policies; the app cleans up its own expired data).

## Architecture
- Hosted on GitHub; Firebase (Firestore) is shared by all students. Every user is linked.
- Question bank: expected 10,000 to 50,000 questions; one subject can have about 5,000. IndexedDB per-question records are the working memory.
- Progress (flags, attempts, review schedule, history) is stored separately from question content and merged by timestamp. Timestamp-based conflict handling decides sync across devices.
- Backups: per-topic files inside subject folders, capped at 500 questions per file (Buddhism 1, Buddhism 2), plus separate state, history and progress files. Backups run on triggers; manual backup is always available.
- Google sign-in gives automatic cloud sync. Google Drive backup renews quietly; if it cannot, ask for one tap at most about once a day. The sign-in reminder appears once a week.
- Test progress saves continuously so a refresh loses nothing. The screen stays awake during a test.
- Bug/problem reports go to the admin automatically in the background; users are told up front.
- Subject and topic lists show in the order they were added.

## Product decisions
### Home and navigation
- Home shows: My big goal (details behind a Details button), Today's goal, Revision, Needs more practice, quick tiles (Add questions, Challenge a friend).
- Goal buttons live only on Home. The exam date is set only on Home (green days-left counter). Light/dark exists in the header and in Profile. Backup/restore and sign in/out live only in the header storage sheet. "Delete all questions" lives only at the bottom of Profile. Profile is a menu of options.
- The main navigation, streak and backup buttons hide only during an active test session, not on the whole Practice tab.

### Practice and tests
- Exam mode: marking scheme chosen by exam (UPPCS, UPSC CSE, JEE and so on), answers can be changed until submit, no explanations while attempting.
- Explanations: stored for all questions; hidden during tests; in Practice a button (E key) opens them; never in exam mode; shown under wrong questions after a test and under flagged questions in History.
- Notes: two-page document icon everywhere; notes are tied to subject/topic; all notes of a subject are reachable from any topic; only an Edit notes button (no separate Add); N key in practice.
- Hide options after first reading: options stay visible only when the correct answer is a "none / more than one / all of the above" option. Test setup shows how many selected questions are hideable.
- Recall: after pressing R and answering correctly, a third round button lets the student say "my recall was wrong". It flags the question, counts as half a miss, and halves the streak.
- A wrong answer does not flag live; wrong questions go to the flagged list when the test ends.
- Focus keyword practice (More options when making a test) searches question text and options only, no cooldown.
- Similar-questions review: small count in More options, groups of up to 50, one copy kept, tags and attempt history moved to the kept copy, "why matched" label and a "Not similar" option.
- Duplicate remover lives in Profile; auto-unflag after 2 correct in a row is a Profile switch (default off).

### Spaced repetition and revision
- Wrong or flagged questions return next day, then day 7, then day 30 (editable schedule in Profile, on/off switch there). One Revision tile, no pop-up, a single total, choose 100 / 200 / custom each time, optional daily budget.

### Duels
- Live head-to-head, nickname#tag handle, friends list with requests, rematch, draw outcome, winner/loser screens. The creator sets the total test time; one countdown; exam-style; auto-submit when time ends or both submit. Duel data is stored in chunk documents and cleaned up by the app itself.

### Parser and books
- Shared book profiles improve parsing as more people upload; admin approves each profile. "Teach the parser" in the admin pool section takes up to 10 PDFs and a rules file. Old rules must never break.
- Automatic PDF page ranges cut at topic boundaries so a topic is never split across windows.

### Planner (built; all code is in index.html, functions start with `pl`)
- A Planner tab replaced the Duels tab in the bottom bar. Duels stay reachable from the "Challenge a friend" tile on Home, and the friend-request badge now sits on that tile.
- Setup sheet ("Plan subjects"): per subject, the number of questions the student plans to practise and the date to finish the first pass by. Leaving both empty keeps a subject out of the plan. One "same date for all" shortcut. Global settings: review days (3), seconds before a skipped question counts (5), percent covered needed to mark complete (90).
- Plans are stored in `settings.planner` (syncs with the other settings). Size of a subject = the larger of the planned number and the questions actually in the bank, so the plan grows as questions are added.
- A question is covered if it was answered, or skipped but on screen for at least 5 seconds. Seen-only questions are kept in their own set (`gc_seen`, this device only), so accuracy, flags and spaced revision are never touched. Nothing in the per-question progress records was changed.
- First pass: complete automatically at 100% covered, or by the student's "Mark complete" tap once at least 90% is covered (Undo is available). The review phase then starts the next day and runs for the review days; its progress is questions answered a second time, and it is marked complete the same way.
- Pace: planned progress runs in a straight line from the plan's starting coverage to 90% at the deadline. Status is ahead, on track, behind or overdue (within 10% of the subject's size counts as on track).
- Balance: time share over the last 14 days is estimated from each test's per-topic time. Fair share is proportional to subject size. Over = above 1.5x fair share, low = below 0.6x, shown only after at least 30 minutes of practice and two or more planned subjects.
- Home: the "Today's plan" tile replaces "Today's goal" while the plan has quotas for today (otherwise Today's goal shows as before). Quotas are fixed when the day's plan is first built (remaining questions divided by days left, behind subjects listed first). A subject that is over its time share and not behind schedule gets zero new questions that day. "Start today's test" builds one test from the quotas.
- After each test, the result screen shows how many of its questions were counted in the planner (percentage), the new coverage per subject, and how many were skipped in under 5 seconds. Discarded unfinished tests count too.
- The owner did not want a separate "Study plan" summary tile on Home; this is why Today's plan replaces Today's goal instead of adding a tile.
