# Article Library Schema

The adapter value for every article-style library is `article`
(and `type` is `library`). This covers the UWorld Notes / AMBOSS-style
libraries and any package that ships a `Docs` + `TOC` pair.

Related docs: [Qbank Schema](schema-qbank.md) · [UpToDate Schema](schema-uptodate.md) ·
[Lexi-Drugs Schema](schema-lexidocs.md) · [Flashcard Schema](schema-flashcard.md)

## Folder layout

```
local-databases/
  my-library/
    manifest.json            <- optional, but required for attribution (see below)
    icon.png                 <- optional, shown on the library card
    my-library.db            <- the primary database
    Images/                  <- optional, images referenced by Docs
```

## Tables

### `Docs` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `title` | TEXT | Article title |
| `doc` | TEXT | Article content (HTML) |
| `path` | TEXT | Optional (UWorld Notes and AMBOSS libraries use it) |

### `TOC` — required

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER | Primary key |
| `name` | TEXT | TOC entry name |
| `docId` | INTEGER | Foreign key → `Docs.id`. `NULL` or `0` for a category node. (`leaf` is unreliable for this — AMBOSS marks categories as `leaf = 1`.) |
| `parentId` | INTEGER | Parent entry (`0` = root) |
| `leaf` | INTEGER | `1` for leaf entries, `0` for parents |
| `section` | TEXT | Optional — groups entries (uworld/amboss libraries ship it) |

### `Search` — optional (full-text content index)

| Column | Type | Notes |
|---|---|---|
| `text` | TEXT | Title or snippet |
| `content` | TEXT / FTS content | `MATCH` target (FLAT FTS table → `MATCH`; plain table → `LIKE`) |
| `contentId` | INTEGER | Foreign key → `Docs.id` |
| `type` | INTEGER | Optional — `1` = title, `2` = content, `3` = section (UWorld/AMBOSS libraries) |
| `typeText` | TEXT | Optional — e.g. `Title`, `Explanation` |
| `section` | TEXT | Optional — section grouping |

**Search behaviour.** The app picks the search strategy per database:

- an FTS virtual `Search` table → `MATCH` query (`Search_content` tables with `%spell` FTS indexes are also accepted);
- a plain `Search` table → `LIKE` scan;
- no `Search` table → it builds an FTS5 index over `Docs` on demand and uses that (so a library whose content is in `Docs` still searches correctly, e.g. AMBOSS 2025).

### `images` — optional

AMBOSS-style libraries ship a one-column `images` table (`imageName`). It is
only read for capability detection (`hasImages`); image assets are resolved
from the media folder by file name, not from this table.

### `toc_fts` — automatic

If a TOC table exists, the app maintains a full-text `toc_fts` index over
`TOC(name)`, ranked by bm25. A shipped index is picked up; otherwise it is
built on demand. It is not required in the package.

## Minimum viable library

```sql
SELECT COUNT(*) FROM Docs;
SELECT id, title FROM Docs WHERE id = ?;
SELECT id, name FROM TOC WHERE parentId = 0 ORDER BY id;
SELECT id, name FROM TOC WHERE parentId = ? ORDER BY id;
```

## Verify

Open the resource after import: the library card shows the manifest display
name, description, publisher, version and tags, and the TOC renders.

## Note on size

The TOC of a large library can hold hundreds of thousands of rows. Do not
add a fixed limit in any custom tooling; the app pages the root entries
with a keyset cursor (`id > ?`) so nothing is hidden.
