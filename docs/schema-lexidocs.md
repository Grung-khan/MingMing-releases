# Lexi-Drugs Schema

The adapter value for a Lexi-style drug reference is `lexi-drugs`
(and `type` is `library`). This is nothing like the `Docs`/`TOC` family —
it is Lexicomp's own schema, and it cannot be mapped onto it.

Related docs: [Qbank Schema](schema-qbank.md) · [Article Schema](schema-article.md) ·
[UpToDate Schema](schema-uptodate.md) · [Flashcard Schema](schema-flashcard.md)

## Folder layout

```
local-databases/
  lexi-drugs-2024/
    manifest.json            <- optional, but required for attribution (see below)
    icon.png                 <- optional, shown on the drug-reference card
    1_patch_f.sqlite         <- the primary database (name may differ)
    fsearch.db               <- optional: Lexicomp full-text index (typeahead)
    fcontentsearch.db        <- optional: content search; wired via manifest "searchDb"
    media/                   <- images, referenced by filename from article HTML
```

The primary database does not have to be called `1_patch_f.sqlite`; the
folder's `manifest.primaryDb` decides the entry point.

## Tables

### `document` — required (the monograph)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `globalid` | INTEGER | Stable id across releases, used for cross-release links |
| `chapter_id` | INTEGER | Foreign key → `chapter.id` |
| `title` | TEXT | Monograph title (e.g. drug name) |

### `chapter` — required (the navigation)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `view_id` | INTEGER | FK → `view.id` (the "chapter view" a letter belongs to) |
| `title` | TEXT | Letter or section heading |

### `field` — required (the monograph body)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `fieldtype_id` | INTEGER | Which field/section this is. `14` is the full monograph body. Field-type labels are mapped by the app (see below). |
| `document_id` | INTEGER | Foreign key → `document.id` |
| `content` | TEXT | Content HTML |

The app reads `field` where `fieldtype_id = 14` for the monograph body and
`viewfield` for the field order and labels.

### `view` — required (field-order groups)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `name` | TEXT | Internal view name |

### `viewfield` — required (field order and labels)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `view_id` | INTEGER | Foreign key → `view.id` |
| `fieldtype_id` | INTEGER | Foreign key → `field.fieldtype_id` |
| `label` | TEXT | Display label, e.g. `Dosing & Indications` |
| `sequence` | INTEGER | Ordering of the field inside the view |

### `indexitem` — optional (the A–Z index)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `indextype_id` | INTEGER | FK → `indextype.id` (e.g. `1` drugs, `2` drug classes) |
| `name` | TEXT | Index term |
| `section` | TEXT | Optional grouping |

### `indexitem_document` — optional (index → monograph map)

| Column | Type | Notes |
|---|---|---|
| `indexitem_id` | INTEGER | FK → `indexitem.id` |
| `document_id` | INTEGER | FK → `document.id` |

### `indextype` — optional

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `name` | TEXT | Index type name |
| `usedocname` | INTEGER | Use the monograph name as the display label |
| `createlinks` | INTEGER | Auto-link contents |
| `stripids` | INTEGER | Strip numeric ids |

### `media` — optional (inline images)

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `filename` | TEXT | File name under `media/` |
| `filesize` | INTEGER | Bytes |
| `data` | BLOB | Inlined image (used when the media file is not on disk) |

### `version` — optional (build stamp)

| Column | Type | Notes |
|---|---|---|
| `productcode` | TEXT | Product code |
| `version_id` | INTEGER | Version id |
| `laststatement_id` | INTEGER | Last-statement id |
| `build` | INTEGER | Build number |

## Companion search databases — optional

`fsearch.db` (glossary/typeahead) and `fcontentsearch.db` (full-text) ship
with the real distribution. The app only reads `fcontentsearch.db` if the
manifest declares `"searchDb": "fcontentsearch.db"`. If neither exists, the
monograph is still readable.

## Minimum viable database

```sql
SELECT COUNT(*) FROM document;
SELECT COUNT(*) FROM chapter;
SELECT id, globalid, title, chapter_id FROM document WHERE chapter_id = ? ORDER BY title COLLATE NOCASE;
SELECT d.id, d.globalid, d.title, d.chapter_id, c.title as chapter_name
  FROM document d JOIN chapter c ON d.chapter_id = c.id
 WHERE d.id = ? LIMIT 1;
SELECT content FROM field WHERE document_id = ? AND fieldtype_id = 14 LIMIT 1;
SELECT id, title, view_id FROM chapter ORDER BY id;
```

## Verify

Import the resource and open it: the card shows the manifest display name,
description, publisher and tags. Browse the index — monographs render with
field labels in `viewfield` order.
