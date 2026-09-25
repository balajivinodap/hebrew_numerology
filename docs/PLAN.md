# Family Numerology App — Plan

Status: draft for review. No app code yet. The decisions marked **OPEN** need an answer before the build starts.

## 1. Goal

Build a private numerology app for the family. It runs on a computer, an iPhone, and an iPad. It does these things:

1. It calculates the numbers for a name, a full date of birth, and any date.
2. It shows interpretations that come from the books and the text database that you supply.
3. It keeps a profile page for each person, with the numbers, life history, and notes.
4. It uses AI (Claude) to research, fill in profile data, and look for patterns. You approve each AI result before the app saves it.
5. Only family members can sign in.

## 2. What the current code does

| Item | Current state | Problem |
|---|---|---|
| Method | Chaldean letter values (1–8), name total, and a "pyramid" of adjacent sums reduced to one digit | The repo name says "Hebrew", but the code uses Chaldean values. There is no Hebrew gematria. |
| Interpretations | Hardcoded text for 8 numbers (10, 24, 33, 41, 6, 13, 15, 51) | Every other number gets generic filler text. Some text ("perfect for algorithmic trading") appears for every person, whatever the number. |
| DOB | The form asks for it | The code uses only the birth year, in a sentence. It calculates no DOB number. |
| Storage | Firebase Firestore, collection `numerology`, no sign-in | The page calls this a "global database". Anybody who opens `numerolo.raajvir.com` can read, add, and delete all entries, unless the Firestore rules block it. I cannot verify the rules from the repo. |
| Pyramid bug | `pyramidNumber` is an array | `addSpecialNumberStyling` and the interpretation lookup compare an array to a number, so they never match. |

Conclusion: keep the calculation logic, and rebuild everything else.

## 3. Recommended architecture

| Layer | Choice | Reason |
|---|---|---|
| App type | Progressive Web App (PWA) | One code base for Mac/PC, iPhone, and iPad. "Add to Home Screen" gives an app icon. No App Store, no Apple developer fee. |
| Front end | React + TypeScript + Vite | Typed calculation code, easy to test. |
| Sign-in | Supabase Auth, email magic link, with an allowlist of family emails | Only allowlisted accounts can read or write. |
| Database | Supabase Postgres with row-level security | Profiles, events, and notes are relational data. The same database holds the book search index (pgvector). |
| Book and text search | Chunk the books, make embeddings, store in pgvector | The app finds the passages that apply to a number and cites the book and page. |
| AI | Claude API, called only from a Supabase Edge Function | The API key stays on the server, never in the browser. |
| Hosting | Keep GitHub Pages at `numerolo.raajvir.com` for the static app | The data is behind sign-in, so a public static page is not a data leak. |

Alternative: stay on Firebase (Auth + Firestore + Cloud Functions). This works, but Firestore vector search and relational queries are weaker. Supabase is simpler for this data.

## 4. Data model

| Table | Main fields |
|---|---|
| `person` | id, full name, other names (birth name, nickname, name changes with dates), DOB, birth time and place (optional), relation |
| `calculation` | person id, method (chaldean, pythagorean, gematria …), input, result JSON, method version |
| `life_event` | person id, date, category (health, career, money, family, travel …), description, source (manual or AI-suggested and approved) |
| `note` | person id, text, author, date |
| `source_doc` | title, author, type (book, text file), your own-copy flag |
| `source_chunk` | doc id, page or section, text, embedding |
| `interpretation` | method, number, text, source chunk ids |
| `prediction` | person id, date made, target date or period, claim, number basis, outcome, hit or miss |
| `ai_run` | person id, prompt type, input, output, status (draft, approved, rejected) |

## 5. Features, in build order

| Phase | Feature | Done when |
|---|---|---|
| 1 | Calculation engine as a tested module: Chaldean name number, pyramid, DOB numbers (life path, birthday, compound), personal year/month/day, any-date number | Unit tests match hand-calculated examples from your books |
| 1 | Sign-in with family allowlist. Move off the open Firestore collection | A non-family account gets no data |
| 2 | Person profile page: numbers, pyramid, name history, timeline of life events, notes | Profile works on iPhone, iPad, and desktop |
| 2 | Import of your text database | All entries map to `interpretation` rows |
| 3 | Book ingestion (PDF/EPUB/text → chunks → embeddings) | A search for a number returns cited passages |
| 3 | Interpretations on the profile from your sources, with citations | Each statement shows its book and page |
| 4 | AI research: Claude reads the profile and the cited passages, then drafts insights and suggests missing events | You approve or reject each draft. Only approved items save |
| 4 | Date forecast: favorable and unfavorable dates for a person, from the rules in your books | Calendar view per person |
| 5 | Prediction log and scorecard | See section 6 |

## 6. Pattern finding — a caution and a test

Numerology has no demonstrated predictive validity in controlled studies. The AI will find "patterns" in any life history, because a large set of numbers and events always contains coincidences. This is the Texas sharpshooter problem and confirmation bias.

The app can test the method instead of only confirming it:

1. The app records each prediction with a date **before** the event.
2. The app records the outcome later.
3. The scorecard compares the hit rate with a base rate (for example, the same claim for a random date).

If you want the "patterns" feature, I recommend this design. Without it, the AI output is interpretation, not evidence.

## 7. Books and copyright

- Only use books that you own.
- Keep the book text in the private database, never in this Git repo or on the public site.
- The app shows short cited passages, not full chapters.

I cannot tell from here whether this repo is public. If it is public, never commit book files to it.

## 8. OPEN decisions

| # | Question | My default |
|---|---|---|
| 1 | Which systems: Chaldean only, or also Pythagorean and Hebrew gematria? | Chaldean first. Add others as plug-in methods. |
| 2 | In what format are the books and the text database (PDF, scanned PDF, EPUB, Word, Excel, plain text)? Scanned PDFs need OCR. | — (need your answer) |
| 3 | Supabase or stay on Firebase? | Supabase |
| 4 | Who are the users, and who can edit vs. only read? | All family members can read and edit. Only you can delete. |
| 5 | Is it acceptable to send profile data to the Claude API for the AI features? | Yes, only on a button press, never automatic |
| 6 | Keep the existing Firestore history, or delete it? | Export it once, import the family entries, then close the open collection |
| 7 | Which exact "pyramid prediction" rule do your books use? The current code uses the second-to-last row. | Confirm against a worked example from the book |

## 9. Next steps

1. Answer the OPEN questions in section 8.
2. Send 2–3 worked examples from the books (name, DOB, expected numbers). These become the unit tests.
3. Send a sample of the text database.
4. Then I build phase 1.
