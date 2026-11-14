# Question Bank Schema

The adapter value for every question bank shipped as a resource is `qbank`
(and `type` is `qbank`). One adapter serves every vendor — the per-vendor
UWorld / AMBOSS / NBME classes were removed in v0.13.1. The packaged
manifest decides routing, so the resource does not need a special vendor name.

Related docs: [Article Schema](schema-article.md) · [UpToDate Schema](schema-uptodate.md) ·
[Lexi-Drugs Schema](schema-lexidocs.md) · [Flashcard Schema](schema-flashcard.md) ·
[VisualDX Schema](schema-visualdx.md)

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
| `id` | INTEGER/TEXT | Primary key (question id). |
| `question` | TEXT | Question stem (HTML). May contain the set item marker (see **Linked question sets**). |
| `explanation` | TEXT | Explanation (HTML). |
| `corrAns` | TEXT | Correct answer letter(s), e.g. `A` or `B,D`. |
| `subId` | TEXT | Comma-separated Subject ids, e.g. `3,5`. |
| `sysId` | TEXT | Comma-separated System ids, e.g. `2,7`. |
| `title` | TEXT | Short title / topic line (used for session search/topic filtering). |
| `mediaName` | TEXT | Primary image filename, resolved relative to the media folder. |
| `otherMedias` | TEXT | Additional media filenames. |
| `pplTaken` | REAL/INTEGER | Total attempts (NBME-style stats; may be float). |
| `corrTaken` | REAL/INTEGER | Correct attempts (NBME-style stats; may be float). |
| `parentQId` | INTEGER | Linked-set group id. `0`/NULL/absent = standalone. See **Linked question sets**. |
| `questionType` | INTEGER | Question/set variant code. See **Linked question sets**. |
| `questionFormatType` | INTEGER | Optional question format code. |
| `lastUpdated` | TEXT | Optional last-updated stamp. |

`parentQId` and `questionType` are **optional/column-guarded**: a bank without them still reads normally (treated as standalone questions).

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

Shipped by many banks. The app matches `Search` with `MATCH` when the table is an FTS virtual table, or with `LIKE` when it is a plain table. Banks commonly store four rows per question (title / question / explanation / one per answer choice), so the in-bank question search groups rows by question and reports the true match count (it does not count the same question multiple times).

| Column | Type | Notes |
|---|---|---|
| `text` | TEXT | Searchable text (naming varies by bank) |
| `content` | TEXT / FTS content | `MATCH` / search target |
| `contentId` | INTEGER | Foreign key → `Questions.id` |
| `type` | INTEGER | 0 = title, 1 = question, 2 = explanation, 3 = answer choice (convention) |
| `typeText` | TEXT | Human-readable type label (e.g. `Explanation`) |

### `Tests` — optional

Session bookkeeping tables (`Tests`, `flags`, `logs`) ship with some banks; they are not read for question delivery.

## Linked question sets (`parentQId`, `questionType`)

Some banks export a **shared-context group** (one vignette split across several questions) as multiple `Questions` rows with the same `parentQId`.

- **Item 1** repeats the vignette (and, for the locked variant, the “items must be answered in sequential order…” notice).
- **Items 2..N** are only `Item N of M` / `Question N of M` plus the stem — **no vignette**. Showing a member alone loses context, so a set is treated as one unit.

### Fields

| Field | Meaning |
|---|---|
| `parentQId` | Group id. `0`/NULL/absent = standalone. Every row of a set carries the same value. |
| `questionType` | Set/variant code. `6` = locked/sequential set; `2`, `3`, `5` = other multi-item variants; `0`, `1`, `4` = standalone. |

`questionType` identifies the locked/sequential variant; grouping is driven by `parentQId`.

### Grouping and item order (as implemented)

- **Grouping key:** `parentQId` **alone**. A group is a set only when it has **≥ 2 members**. A `parentQId` that equals a row’s own id with no siblings is a standalone (some exports do this).
- **Member order:** the in-text marker, **not** id order and **not** row order. Parsers look for the first `Item N of M` / `Question N of M` marker in `question` HTML (UWorld: `<strong>Item 2 of 2</strong>`, MCC: `<h4>Question 2 of 3</h4>`). Id order is only the fallback when no marker is present. Real banks can number a set in reverse of id order, and a referenced parent row may be absent.
- **Set integrity beats filtering:** if any member of a set is selected/matched, the **whole** set is included (even members excluded by status/marked/difficulty), and members stay in marker order inside the set.
- **Session building:** sets are shuffled/placed as **units** (chaos/order never reorders inside a set). A batch/block boundary never splits a set. A QID that targets a set member expands to the full set.
- **Locked variant UI:** `questionType == 6` is preserved/carried, but blocking backwards navigation / freezing answers after proceeding is a UI behavior (not implied by schema alone).

### Set index

To keep sets whole without shipping full question HTML, the app reads a small per-question index: `{ id, parentQId, questionType, itemIndex }` (whole bank, unfiltered). `itemIndex` is derived by parsing the marker from `question` HTML for set members only.

## Indexing guidance

- Index `Questions(subId)` and `Questions(sysId)`.
- Index `Answers(qId)`.
- Index `Subjects(name)`.
- If you ship `parentQId`, keep it present and consistent across all members of a set.

## Minimum viable bank

```sql
SELECT COUNT(*) FROM Questions;
SELECT id, question, explanation, corrAns, subId, sysId, title, parentQId, questionType FROM Questions WHERE id = ?;
SELECT id, qId, answerId, answerText FROM Answers WHERE qId = ?;
SELECT id, name FROM Subjects;
SELECT id, name FROM Systems;
SELECT refTitle, refLink FROM Refs WHERE qId = ? ORDER BY id;   -- optional
```

## Verify

Open the resource after import: the card shows the manifest's display name,
description, publisher and tags, and the bank opens in session view. Linked
sets (where present) should appear as one contiguous unit with members in item
order.
