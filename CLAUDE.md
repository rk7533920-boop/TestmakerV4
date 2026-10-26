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
- **Notes sync (build index-31):** notes are kept in the online account at `meta/notes1`, `meta/notes2`, ... (80 notes per document, covered by the existing `users/{uid}/{document=**}` rule) and merged by edit time (`u`) on app start, after every note change (2.5 s later) and on every cloud refresh. A deleted note stays as a tombstone (`del`) for 90 days so deletions reach every device. Code: `notesCloudSync` / `notesCloudSoon`. Folder/Drive backups still carry notes too.
- **Settings sync (build index-31, extended in index-32):** `meta/settings` is no longer overwritten as a whole. `persistSettings` writes only the settings that changed (merge), using `_setBase` (what the cloud last held), and `settingsPull` (run on every refresh) takes over settings changed on another device; if both changed the same setting, this device's value wins; `tpos` is merged entry by entry. Since build index-32 these also sync: marking scheme and exam preset (`marking`, `markPreset`), skip-similar switch (`simOn`), "Not similar" choices (`notSim`, union-merged), backup-reminder off (`remOff`), welcome flag (`welcomed`), custom one-pager prompt (`opPrompt`), all through `syncPrefsFromSettings` (settings are never written online before they were read: `prefPersist`). The planner's seen-only set lives in `meta/seen1..` (`seenCloudSync`, union merge). Device-only by design: sign-in nudge and snooze timers, Drive/backup-folder state, device id.
- Test progress saves continuously so a refresh loses nothing. The screen stays awake during a test.
- Bug/problem reports go to the admin automatically in the background; users are told up front.
- Subject and topic lists show in the order they were added. For a book, topics are saved in the book's own order: `uid()` hands out strictly increasing ids, so topics saved together in one go keep their book order (a topic is placed by its oldest question's id time; ties used to fall back to A-Z). Topics saved before build index-29 that share one millisecond still show A-Z. Across sessions: topics that come from a PDF store the book (file name key) and the page where they start in `settings.tpos` (`subject\u0001topic` -> `{b, p}`, earliest page wins). In a subject, topics of one book sit together (where the first of them was added) and follow the book's page order, whatever session each was added in (`creationOrder`/`topicCmp`). Topics saved before build index-30 have no page note and keep the order they were added.

## Product decisions
### Progress chart
- The Progress tab chart "New vs revised" is a line chart (since build index-30): a bold Total line with the total number of questions written at every day's point, plus thin dashed New and Revised lines. 7/14 days fit the screen; 30 days scrolls sideways and opens on the latest days.

### Library and test screen (build index-31)
- Library: tapping a topic opens a strip under it with three equal buttons, Notes / Export / Delete (line icons, no emoji); one topic is open at a time. The notes marker (icon plus count) sits at the far right of the row. A subject has a 3-dots button at the far right that opens the same strip for the whole subject; tapping the subject name still expands its topics. Code: `renderBankBreakdown`, `bankActs`.
- Test screen header: one row with the breadcrumb (up to 2 lines) and the timer chip, then ONE row of three equal-size buttons (Issue / Archive or Bookmark / Flag) that never wraps. Flag and explanation (bulb) use line icons instead of emoji so Android and iPad draw them the same. Duel screens still use the old `.flagbar`.

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

### Planner (v3, build index-28; all code is in index.html, functions start with `pl`)
- A Planner tab replaced the Duels tab in the bottom bar. Duels stay reachable from the "Challenge a friend" tile on Home, and the friend-request badge sits on that tile.
- Owner found the planner too complicated, so v3 is simpler without losing features. **Two modes** (Simple / Advanced switch at the top of the Planner screen, stored as `settings.planner.mode`; plans that already have reviews open in Advanced):
  - **Simple**: per subject only the number of questions (student decides) and a Reading schedule (start and end dates). A "Days given" tracker shows on how many different days each subject was practised against the days planned for its reading (from `history`), warning when a subject uses more days than planned. No reviews, study style, Days ahead strip or seconds-based time balance. Pace = Daily goal, no limit on blocks per day.
  - **Advanced**: everything from before (study style and split, Days ahead strip, seconds-based Time balance, reviews, shift).
- Reviews (Advanced only) are never shown all at once: a subject opens with only its Reading stage, and the student adds Review 1, 2, 3 one at a time with the round **+** button under the stages. Each added review gets its own progress bar and dates.
- The study window (from/to time) and the "Time to study" banner were removed from the UI and logic. Old `from`/`to` values may remain in saved blocks and are ignored.
- **Reading can be ticked manually** ("Mark reading as done", any coverage, tap again to undo) because progress data was once wiped. It sets `done` on the Reading block, which unlocks reviews and moves the subject on. Only Reading has this tick for now; reviews still use "Mark complete" once 90% is reached.
- The two rules are fixed in code (`PL_MIN=5`, `PL_PASS=90`), not user settings: a skipped question counts after 5 seconds on screen, and 90% covered counts as complete. There are no boxes for them anywhere.
- Plans live in `settings.planner` (version 2): `mode`, `style`, `split`, `subs[subject] = {n, blocks[]}` (`min` and `pass` are kept only for old data). Blocks: one Reading (`t:'r'`) and up to three Reviews (`v1`, `v2`, `v3`) with `s`/`e` dates and `done` (date marked complete).
- Budget: reuses the Daily goal (`dayGoal()`), set only on Home. If the goal is under 100 when a plan is first made, setup asks the student to confirm or raise it. The setup sheet's only Planner-wide options are the start date (and, in Advanced, style/split).
- Stage bars: Reading = attempted at least once (or seen 5 s) out of the subject's size. Review k = attempted at least k+1 times, out of (those) + (flagged questions that have not yet reached that attempt). Reviews practise all flagged questions that have fewer than k+1 attempts. A review completes automatically when no flagged question is waiting, or by "Mark complete". Nothing new is stored per question.
- A question is covered if it was answered, or skipped but on screen for at least 5 seconds. Seen-only questions are kept in their own set (`gc_seen`, this device only), so accuracy, flags and spaced revision are never touched.
- Mixed tests: every question counts to its own subject's progress and quota. The result screen shows "Counted: History +140, Polity +60" and today's total against the daily goal.
- Behind schedule: "Shift the rest by N days" extends the current block and moves later blocks by the same amount (asks to confirm).
- Home: the "Today's plan" tile replaces "Today's goal" while the plan has quotas for today. Quotas are fixed when the day's plan is first built. "Start today's test" builds one test from the quotas.
- The Details button on Home's "My big goal" tile was removed; the per-subject stage bars replace it. The owner did not want a separate "Study plan" summary tile on Home.

### One-pagers (build index-32)
- Purpose: a compact final-revision sheet per subject ("one-pager" is a name, not literally one page): only weak-area facts, keyword one-liners merged across questions, small tables, chronological order, two columns split by a vertical line, printable.
- Library subject 3-dots menu: Notes (with count; the notes icon no longer sits on the subject row), Export, Delete, then full-width Make one-pager and Upload/Replace one-pager. Topic rows are unchanged.
- Make (`opMakeOpen`, `opBuild`): sheet with toggles (notes, flagged, other wrong, all remaining), topic picker, editable AI instructions (`OP_PROMPT`, custom text in `settings.opPrompt`), size estimate, Download / Share (Web Share, phones) / Copy message / Open Claude. The JSON holds the prompt in `instructions`.
- Keep (`opStore`, `opOpen`): HTML preferred (prompt asks the AI for one self-contained printable HTML file); PDF (rendered with pdf.js) and images (shrunk to 2400 px JPEG) also accepted. Stored in IndexedDB (`gc_op_idx`, `gc_op_<id>`), and when under 3 MB also in the account: `meta/onepagers` (list, merged by `u`, deletions are tombstones for 90 days) and `meta/op<id>_<n>` (base64 pieces of 700 KB). A subject with a one-pager shows an icon before its name that opens the viewer (Print / Save / Remove).
- Not in the folder/Drive backups on purpose (backup code untouched to keep flagged data safe).

### Option shuffle (build index-32)
- Profile switch `settings.shuffleOpts` (synced, default off). Only questions with `attemptCount>0` and no positional option text ("All of the above", "None of these", "above", "below"...) are shuffled.
- Only the on-screen order changes; option keys/letters stay original, so explanations like "option B" and all scoring stay valid. The order is made lazily when a question is first shown (`ordOpts`), kept in `session.ord` (saved in the session snapshot), and used by the test screen, the review of wrong answers and keys 1-4 (which follow position). Library, editing, duel and exports keep the original order.
- The one-pager prompt (`OP_PROMPT`) was replaced with the user's improved version: under 10 pages, notes decide, SVG diagrams required, render-check only if possible.
