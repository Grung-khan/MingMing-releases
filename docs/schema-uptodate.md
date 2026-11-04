# UpToDate Library Schema

The adapter value for the UpToDate library is `uptodate` (and `type` is
`library`). Same data family as the [Article Schema](schema-article.md)
with a small set of extras that the UpToDate UI mode reads.

Related docs: [Qbank Schema](schema-qbank.md) · [Article Schema](schema-article.md) ·
[Lexi-Drugs Schema](schema-lexidocs.md) · [Flashcard Schema](schema-flashcard.md)

## Folder layout

```
local-databases/
  uptodate/
    manifest.json            <- optional, but required for attribution (see below)
    icon.png                 <- optional, shown on the library card
    uptodate-medical-library.db   <- the primary database
```

UpToDate articles are ~8 MB of HTML each. Images are stored in a `media/`
folder of the optional supplemental SQL database in this 2025 build; no
supplemental media database is required for the reader to open topics.

## Tables

### `Docs` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `title` | TEXT | Article title |
| `doc` | TEXT | Article content (HTML) |

(`path` is not used by this package.)

### `TOC` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `name` | TEXT | Entry name |
| `docId` | INTEGER | Foreign key → `Docs.id`. `NULL` or `0` for a category node. |
| `parentId` | INTEGER | Parent entry (`0` = root), default `0` |
| `leaf` | INTEGER | `1` for leaf entries, `0` for parents, default `0` |
| `audience` | TEXT | Target audience, e.g. `adult` / `patient` (default `adult`) |

The root holds 28 categories and ~20,648 articles with a `section` +
`audience` surface. The reader pages the articles of a category with a
keyset cursor and shows categories in full.

### `Search` — required

A plain FTS (non-virtual) table. The app matches it with `LIKE` and the
TOC is searched through an FTS index over `TOC(name)`.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `text` | TEXT | Title or snippet |
| `content` | TEXT | `MATCH` target |
| `contentId` | INTEGER | Foreign key → `Docs.id` |

(The shipped FTS tables in this package, `Search`, are plain — the app does
not call `MATCH` on it.)

### `Graphics` — optional

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `title` | TEXT | Graphic title |
| `imageHtml` | TEXT | Inline graphic HTML |
| `base64Image` | TEXT | Optional base64 image payload |

The reader shows graphics inline where their id is referenced from an
article body.

### `toc_fts` — automatic

The app maintains (or uses a shipped) FTS index over `TOC(name)` and ranks
results by bm25. ~200k entries is fine; do not add `LIMIT`-style truncation.

## Minimum viable library

```sql
SELECT COUNT(*) FROM Docs;
SELECT id, title FROM Docs WHERE id = ?;
SELECT id, name, docId, parentId, leaf, audience FROM TOC WHERE parentId = 0 ORDER BY id;
SELECT id, name, docId, parentId, leaf, audience FROM TOC WHERE parentId = ? ORDER BY id;
```

## Verify

Import the resource and open it: the library card shows the manifest
details and the TOC renders with the `audience` toggle in the reader.
