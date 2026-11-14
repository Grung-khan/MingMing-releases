# VisualDX Schema

VisualDX is a **library-type clinical workflow** resource (manifest `type: "library"`, `adapter: "visualdx"`). It is not served by the generic article/library adapter: MingMing opens its databases and folder assets directly through `VisualDxWorkflow.js`.

Unlike other libraries, a VisualDX resource is a **folder of databases, JSON configs, and image assets**. The workflow drives focus → body location → lesion/questionnaire → differential → condition detail/search.

Related docs: [Qbank Schema](schema-qbank.md) · [Article Schema](schema-article.md) · [Flashcard Schema](schema-flashcard.md)

## Folder layout

```
local-databases/
  <resourceId>/
    manifest.json
    icon.png                     <- optional card icon (manifest "icon")
    VisualDX.db                  <- primary clinical database (manifest "primaryDb")
    fcontentsearch.db            <- full-text content index (manifest "searchDb", optional)
    focusAreaList.json           <- focus/workflow entry points
    modules/
      <moduleId>.json            <- per-module questionnaire config (one per workflow module)
    BodyLocation/
      <bodyLocationType>/
        info.js                  <- body-location map/config for that type
    Icons/                       <- finding/distribution icon images referenced by module configs
    Medium/
      <imageId>.jpg              <- condition/example images referenced by Images.imageId
    *.json                       <- additional lesion/distribution configs shipped by the package
```

`manifest.json` fields (relevant): `resourceId`, `type: "library"`, `adapter: "visualdx"`, `primaryDb` (the main DB filename), optional `searchDb`. Assets are read off disk via the `local:` fetch bridge (`local:<resourceId>/<relative path>`), not from the IndexedDB resource store.

## Databases

### `VisualDX.db` (primary)

The workflow registers this as `db_id: "VisualDX"` and queries the following tables/columns.

#### `Diagnoses`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Diagnosis id |
| `dName` | TEXT | Diagnosis name |
| `severityLevel` | INTEGER | Severity |
| `sortPriority` | INTEGER | Sort priority |
| `defaultModule` | INTEGER | Default module for the diagnosis |

Used for: diagnosis listing and metadata.

#### `Modules`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Module id (matches `modules/<id>.json` and `focusAreaList` module references) |
| `shortName` | VARCHAR | Short name |
| `category1` | VARCHAR | Category |
| `category2` | VARCHAR | Category |
| `longName` | VARCHAR | Long name |
| `authors` | TEXT | Authors |
| `desc` | TEXT | Description |

Used for: module info.

#### `DiagnosesModules`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Diagnoses-modules row id (the `diagnosesModulesId` used elsewhere) |
| `diagnosisId` | INTEGER | FK → `Diagnoses.id` |
| `moduleId` | INTEGER | FK → `Modules.id` |
| `imageCount` | INTEGER | Image count |

Used for: resolving `(diagnosisId, moduleId)` → `diagnosesModulesId`.

#### `DiagnosesModulesFindings`

Central join for building a differential from selected findings.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Row id |
| `diagnosesModulesId` | INTEGER | FK → `DiagnosesModules.id` |
| `moduleId` | INTEGER | Module |
| `findingId` | INTEGER | Selected finding id |
| `images` | TEXT | Associated image ids (string/list as stored) |
| `diagnosisTitle` | TEXT | Diagnosis title |
| `severity` | INTEGER | Severity |
| `sort` | INTEGER | Sort order |

Used for:
- differential: `WHERE moduleId = ? AND findingId IN (...) GROUP BY diagnosesModulesId` ordered by `matchCount DESC, severity ASC, sort ASC`;
- category findings: distinct findings for a module within a category id range joined to `Findings`.

#### `Findings`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Finding id (referenced by module configs and `DiagnosesModulesFindings`) |
| `shortName` | VARCHAR | Display name |
| `longName` | TEXT | Long name |
| `leaf` | INTEGER | Leaf flag |
| `parentId` | INTEGER | Parent finding |
| `children` | TEXT | Children |
| `supportedModules` | TEXT | Supported modules |

Used for: finding names and the per-category finding picker.

#### `Documents`

Condition content, keyed by field.

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Row id |
| `diagnosesModulesId` | INTEGER | FK → `DiagnosesModules.id` |
| `fieldId` | INTEGER | Content field id |
| `doc` | TEXT | HTML/text content |

Used for: `WHERE diagnosesModulesId = ? ORDER BY fieldId`. Field ids map to section labels (e.g. 1 ICD codes, 2 Synopsis, 3 Look For, 4 Diagnostic Pearls, 5 Differential Diagnosis & Pitfalls, 6 Best Tests, 7 Management Pearls, 8 Therapy, 9 References, 10 Associated Findings, 11 Associated Medications — as observed in shipped content).

#### `Images`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Row id |
| `diagnosesModulesId` | INTEGER | FK → `DiagnosesModules.id` |
| `imageId` | INTEGER | Image identifier → file `Medium/<imageId>.jpg` |
| `copyRight` | TEXT | Copyright/credit |

Used for: condition image gallery. The on-disk file set in `Medium/` must cover the referenced `imageId` values.

#### `DiagnosesSearch`

FTS (FTS4, porter) diagnosis name search.

| Column | Type | Notes |
|---|---|---|
| `id` | — | Diagnosis id |
| `dName` | TEXT | Diagnosis name |
| `dNameSearch` | TEXT | Search form |

Used for: `WHERE dName MATCH ?` (diagnosis search). (Other FTS/spell/support tables may be present in shipped databases and are not required by the workflow.)

### `fcontentsearch.db` (search, optional but recommended)

Registered as `db_id: "fcontentsearch"` when present.

#### `Search`

FTS (FTS4, porter) content search.

| Column | Type | Notes |
|---|---|---|
| `Text` | TEXT | Result title |
| `SubText` | TEXT | Result subtitle |
| `URL` | TEXT | Encoded reference; parsed as `diagnosisId-moduleId` to open a condition |
| `content` | TEXT | Indexed content |

Used for: `WHERE content MATCH ?`.

## Config and asset files

### `focusAreaList.json`

Defines the focus/workflow entry points and the `moduleId` (and, where applicable, body-location type) selected to start a workflow. Loaded at startup of the VisualDX screen.

### `modules/<moduleId>.json`

One JSON config per workflow module (module id matches `Modules.id` and focus-area selection). The loader tolerates a legacy `var/let/const = …;` wrapper but the canonical on-disk form is plain `.json`.

Top-level keys read by the questionnaire:

| Key | Purpose |
|---|---|
| `visualFindingIconImageList` | Lesion/morphology findings shown as an icon grid. Entries carry `findingId`, `imageName`/`mouseoverImageName` (paths under `Icons/`), optional `imageId`s, and optional `childIcons[]` (same shape). |
| `distributionFindingIconImageList` | Distribution-pattern findings (icon grid), rendered when non-empty. |
| `keyQuestionList` | Additional clinical questions; entries contain findings (e.g. radio/checkbox groups of `findingId`s) rendered by the quiz-question UI. |

Selected `findingId`s (including `childIcons`) drive the differential query against `DiagnosesModulesFindings` for that `moduleId`. Finding display names come from `Findings.shortName`.

### `BodyLocation/<bodyLocationType>/info.js`

Body-location map/config for a location type selected from the focus flow. Parsed as JSON-like config (loader strips wrappers/trailing commas as needed).

### `Icons/` and `Medium/`

- `Icons/...` — `imageName`/`mouseoverImageName` targets referenced by module and body-location configs.
- `Medium/<imageId>.jpg` — condition/example images referenced by `Images.imageId`.

## Notes

- The workflow depends on the specific table/column names above (see `public/assets/VisualDxWorkflow.js`). A package that renames these columns or omits the queried tables will not render correctly.
- `diagnosesModulesId` is the stable key linking `DiagnosesModules`, `DiagnosesModulesFindings`, `Documents`, `Images`, and (via `URL`) search results.
- Category finding ranges are derived from the service’s category id map over `Findings.id`; the exact category set is defined in code and must align with the shipped finding id ranges.
