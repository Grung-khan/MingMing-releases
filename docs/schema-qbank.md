# Question Bank Schema

The adapter value for every question bank shipped as a resource is `qbank`
(and `type` is `qbank`). One adapter serves every vendor — the per-vendor
UWorld / AMBOSS / NBME classes were removed in v0.13.1. The packaged
manifest decides routing, so the resource does not need a special vendor name.

Related docs: [Article Schema](schema-article.md) · [UpToDate Schema](schema-uptodate.md) ·
[Lexi-Drugs Schema](schema-lexidocs.md) · [Flashcard Schema](schema-flashcard.md)

## Folder layout

```
local-databases/
  my-qbank/
    manifest.json            <- optional, but required for attribution (see below)
    icon.png                 <- optional, shown on the bank's card
    my-qbank.db              <- the primary database (.db / .sqlite / .sqlite3 / .bin)
    media/                   <- optional, images referenced by Questions
```

## Tables

### `Questions` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `question` | TEXT | Question stem (HTML) |
| `explanation` | TEXT | Explanation (HTML) |
| `corrAns` | TEXT | Correct answer letter(s), e.g. `A` or `B,D` |
| `subId` | TEXT | Comma-separated Subject ids, e.g. `3,5` |
| `sysId` | TEXT | Comma-separated System ids, e.g. `2,7` |
| `title` | TEXT | Short title / topic line (used for session search) |
| `mediaName` | TEXT | Primary image filename, resolved relative to the media folder |
| `otherMedias` | TEXT | Additional media filenames |
| `pplTaken` | REAL/INTEGER | Total attempts (NBME-style stats; may be float) |
| `corrTaken` | REAL/INTEGER | Correct attempts (NBME-style stats; may be float) |
| `parentQId` | INTEGER | Optional parent question id (full-strength / follow-up questions) |
| `questionType` | INTEGER | Optional question type code |
| `questionFormatType` | INTEGER | Optional question format code |
| `lastUpdated` | TEXT | Optional last-updated stamp |

### `Answers` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `qId` | INTEGER | Foreign key → `Questions.id` |
| `answerId` | INTEGER | 1-based option number |
| `answerText` | TEXT | Option text (HTML) |
| `correctPercentage` | INTEGER | % of users who chose this option (NBME-style) |

### `Subjects` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key; referenced by `Questions.subId` |
| `name` | TEXT | Subject / discipline name |
| `count` | INTEGER | Optional question count (used by the flashcard adapter) |

### `Systems` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key; referenced by `Questions.sysId` |
| `name` | TEXT | Body system name |
| `count` | INTEGER | Optional question count |

### `Refs` — optional

Per-question references, used by the explanation panel.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `qId` | INTEGER | Foreign key → `Questions.id` |
| `refTitle` | TEXT | Reference title |
| `refLink` | TEXT | URL |
| `refType` | INTEGER | Optional type code |

### `Search` — optional (full-text content index)

Shipped by every real bank. The app matches `Search` with `MATCH` when the
table is an FTS virtual table, or with `LIKE` when it is a plain table. Four
rows per question (title / question / explanation / one per answer choice),
so the app uses `db_search_questions`, which groups rows by question and
reports the true match count.

| Column | Type | Notes |
|---|---|---|
| `text` | TEXT | The searchable text |
| `content` | TEXT / FTS content | `MATCH` target |
| `contentId` | INTEGER | Foreign key → `Questions.id` |
| `type` | INTEGER | 0 = title, 1 = question, 2 = explanation, 3 = answer choice |
| `typeText` | TEXT | Human-readable type label (e.g. `Explanation`) |

### `Tests` — optional

Session bookkeeping tables (`Tests`, `flags`, `logs`) ship with some banks;
they are not read for question delivery.

## Indexing guidance

- Index `Questions(subId)` and `Questions(sysId)`.
- Index `Answers(qId)`.
- Index `Subjects(name)`.

## Minimum viable bank

```sql
SELECT COUNT(*) FROM Questions;
SELECT id, question, explanation, corrAns, subId, sysId, title FROM Questions WHERE id = ?;
SELECT id, qId, answerId, answerText FROM Answers WHERE qId = ?;
SELECT id, name FROM Subjects;
SELECT id, name FROM Systems;
SELECT refTitle, refLink FROM Refs WHERE qId = ? ORDER BY id;   -- optional
```

## Verify

Open the resource after import: the card shows the manifest's display name,
description, publisher and tags, and the bank opens in session view.
