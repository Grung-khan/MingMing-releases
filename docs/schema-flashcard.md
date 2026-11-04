# Flashcard Schema

Flashcard resources use the same physical layout as a question bank: a
qbank-format SQLite database, with `type: "flashcard"` and
`adapter: "flashcard"` in the manifest. There is no separate flashcard
format.

Related docs: [Qbank Schema](schema-qbank.md) · [Article Schema](schema-article.md) ·
[UpToDate Schema](schema-uptodate.md) · [Lexi-Drugs Schema](schema-lexidocs.md)

## Folder layout

```
local-databases/
  my-flashcards/
    manifest.json            <- optional, but required for attribution (see below)
    icon.png                 <- optional, shown on the deck card
    my-flashcards.bin        <- the deck database (.bin/.db/.sqlite — any of these)
    media-E/                 <- optional, images referenced by cards
```

## Tables

A flashcard deck needs the same tables as a question bank, but only two are
read by the flashcard player.

### `Subjects` — required (the decks)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `name` | TEXT | Deck / subject name (e.g. `Step 1`), used as the deck name |
| `count` | INTEGER | Number of cards in the deck |

### `Questions` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `subId` | INTEGER | Foreign key → `Subjects.id` — used to list cards in a deck |
| `title` | TEXT | Short title / topic line |
| `question` | TEXT | The front of the card (HTML) |
| `explanation` | TEXT | The back of the card (HTML) |
| `sysId` | INTEGER | Optional; systems are not used by the flashcard player |
| `mediaName` | TEXT | Optional image filename (front) |
| `otherMedias` | TEXT | Optional image filenames (back/extra) |

Unlike a question bank, `corrAns`, `Answers` and `refLink` are not read.
A full qbank database works without changes — a flashcard resource may carry
them because it is usually a re-packaged bank.

### `Answers`, `Refs`, `Tests`, `flags`, `logs` — not read

These ship with real flashcard packages (they are question-bank databases at
heart) and are simply ignored by the flashcard player.

### `Search` — not used

Deck search inside a resource goes over `Questions` directly; a full-text
index is not required.

## Minimum viable deck

```sql
SELECT COUNT(*) FROM Questions;
SELECT id, name, count FROM Subjects ORDER BY name;
SELECT id FROM Questions WHERE subId = ? ORDER BY id;
SELECT id, question, explanation, mediaName, otherMedias FROM Questions WHERE subId = ? ORDER BY id;
```

## Verify

Import the resource and open the Flashcards tab. The deck appears as a card
on the landing page — one card per installed resource — and opens the deck
list.
