# Netflix Relational Database — SQL Mini-Project

A normalized SQL Server database built from the [Netflix Movies and TV Shows dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows) on Kaggle, covering schema design, data loading, indexing, and analytical querying.

## Contents

- [Dataset](#dataset)
- [Schema Design](#schema-design)
- [Setup / Run Order](#setup--run-order)
- [Data Loading Process](#data-loading-process)
- [Challenges & Debugging](#challenges--debugging)
- [Indexing](#indexing)
- [Analysis Queries](#analysis-queries)
- [Query Optimisation](#query-optimisation)
- [Next Steps / What I'd Do Differently](#next-steps--what-id-do-differently)

---

## Dataset

Source: Kaggle — Netflix Movies and TV Shows (`netflix_titles.csv`)

![Netflix Dataset Kaggle](netflix_dataset.PNG)

The raw CSV is **not included in this repo** (not my data to redistribute) — download it directly from Kaggle and place it locally before running the load scripts.

Raw structure: one row per title, with `director`, `cast`, `country`, and `listed_in` (genre) stored as comma-separated free text in single columns — this shaped most of the design decisions below.

## Schema Design

![ERD](netflix_erd.PNG)

The raw CSV's multi-valued columns (a title can have multiple directors, cast members, countries, and genres) meant a flat single-table design would violate normal form. I split these into lookup entities + junction tables:

- `Titles` — one row per show (core entity)
- `Directors`, `Actors`, `Countries`, `Genres` — lookup tables, each distinct value stored once
- `TitleDirectors`, `TitleActors`, `TitleCountries`, `TitleGenres` — junction tables resolving the many-to-many relationships

Full DDL: 

```sql
CREATE DATABASE NetflixDB;
GO
USE NetflixDB
GO

CREATE TABLE Titles (
    show_id       VARCHAR(10)   NOT NULL PRIMARY KEY,   -- from source CSV (e.g. s1, s2...)
    title         NVARCHAR(255) NOT NULL,
    type          VARCHAR(20)   NOT NULL,               -- 'Movie' or 'TV Show'
    date_added    DATE          NULL,
    release_year  SMALLINT      NOT NULL,
    rating        VARCHAR(10)   NULL,
    duration      VARCHAR(20)   NULL,
    description   NVARCHAR(1000) NULL
);
GO
 
CREATE TABLE Directors (
    director_id   INT IDENTITY(1,1) PRIMARY KEY,
    director_name NVARCHAR(150) NOT NULL UNIQUE
);
GO
 
CREATE TABLE Actors (
    actor_id      INT IDENTITY(1,1) PRIMARY KEY,
    actor_name    NVARCHAR(150) NOT NULL UNIQUE
);
GO
 
CREATE TABLE Countries (
    country_id    INT IDENTITY(1,1) PRIMARY KEY,
    country_name  NVARCHAR(100) NOT NULL UNIQUE
);
GO
 
CREATE TABLE Genres (
    genre_id      INT IDENTITY(1,1) PRIMARY KEY,
    genre_name    NVARCHAR(100) NOT NULL UNIQUE
);
GO
 
CREATE TABLE TitleDirectors (
    show_id       VARCHAR(10) NOT NULL,
    director_id   INT NOT NULL,
    PRIMARY KEY (show_id, director_id),
    FOREIGN KEY (show_id) REFERENCES Titles(show_id),
    FOREIGN KEY (director_id) REFERENCES Directors(director_id)
);
GO
 
CREATE TABLE TitleActors (
    show_id       VARCHAR(10) NOT NULL,
    actor_id      INT NOT NULL,
    PRIMARY KEY (show_id, actor_id),
    FOREIGN KEY (show_id) REFERENCES Titles(show_id),
    FOREIGN KEY (actor_id) REFERENCES Actors(actor_id)
);
GO
 
CREATE TABLE TitleCountries (
    show_id       VARCHAR(10) NOT NULL,
    country_id    INT NOT NULL,
    PRIMARY KEY (show_id, country_id),
    FOREIGN KEY (show_id) REFERENCES Titles(show_id),
    FOREIGN KEY (country_id) REFERENCES Countries(country_id)
);
GO
 
CREATE TABLE TitleGenres (
    show_id       VARCHAR(10) NOT NULL,
    genre_id      INT NOT NULL,
    PRIMARY KEY (show_id, genre_id),
    FOREIGN KEY (show_id) REFERENCES Titles(show_id),
    FOREIGN KEY (genre_id) REFERENCES Genres(genre_id)
);
GO
```

## Setup / Run Order

Scripts must be run in this order — later scripts depend on tables/data created by earlier ones:

1. `sql/01_create_tables.sql` — creates the database and core schema
2. `sql/02_create_staging.sql` — creates the flat staging table
3. Load the CSV into `Staging_Netflix` (see [Data Loading Process](#data-loading-process))
4. `sql/03_split_and_load.sql` — splits multi-value columns and populates all lookup + junction tables
5. `sql/04_indexes.sql` — adds indexes (run **after** data is loaded, not before)
6. `sql/05_analysis_queries.sql` — the 7 analytical queries

## Data Loading Process

### Why a staging table?

The raw CSV can't be inserted directly into the normalized schema — `director`, `cast`, `country`, and `listed_in` contain comma-separated multi-values that need to be split before they map onto separate lookup and junction tables. The staging table is a flat, permissive landing spot (no constraints, dates imported as text) that mirrors the CSV exactly, so the import itself can't fail on type mismatches or messy data. Splitting happens afterward, in SQL, against this staging copy.

```sql
CREATE TABLE Staging_Netflix (
    show_id       VARCHAR(10),
    type          VARCHAR(20),
    title         NVARCHAR(255),
    director      NVARCHAR(MAX),
    cast_raw      NVARCHAR(MAX),
    country       NVARCHAR(MAX),
    date_added    NVARCHAR(50),   -- text on purpose — converted to DATE after load
    release_year  SMALLINT,
    rating        VARCHAR(10),
    duration      VARCHAR(20),
    listed_in     NVARCHAR(MAX),
    description   NVARCHAR(1000)
);
```

### Loading the CSV

Used `BULK INSERT` rather than the SSMS Import wizards — the Import and Export Wizard wasn't available in this SSMS install (missing SSIS component), and `BULK INSERT` is scriptable and version-controllable, which fits a repo better than a one-off GUI action anyway.

```sql
BULK INSERT Staging_Netflix
FROM 'C:\NetflixData\netflix_titles.csv'
WITH (
    FIRSTROW = 2,
    FORMAT = 'CSV',
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '0x0a',
    CODEPAGE = '65001',
    TABLOCK
);
```

Screenshot: *[row count check confirming staging load matched the CSV row count]*

## Challenges & Debugging

### 1. `BULK INSERT` row terminator mismatch

Initial load failed with a generic `Cannot obtain the required interface ("IID_IColumnsInfo")` error. This message doesn't describe the real problem — the actual cause was a row terminator mismatch: `ROWTERMINATOR = '\n'` didn't match the CSV's actual line endings. Fixed by switching to the hex form `ROWTERMINATOR = '0x0a'`, which matches regardless of `\n` vs `\r\n`.

Screenshot: *[the error message]*

### 2. Duplicate key violation when populating junction tables

```
Violation of PRIMARY KEY constraint 'PK__TitleDir__...'. Cannot insert duplicate key... (s3719, 1801)
```

Cause: a small number of source rows had the same director name repeated within one comma-separated cell. `STRING_SPLIT` faithfully returned both occurrences, and the junction table insert didn't deduplicate at the `(show_id, director_name)` level before joining. Fixed by adding `DISTINCT` inside the splitting CTE, before the join to look up each `director_id`.

Screenshot: *[error + corrected query]*

### 3. `STRING_AGG(DISTINCT ...)` invalid syntax, and the fan-out it was covering for

Attempting a validation query that joined `Titles` to `TitleDirectors`, `TitleGenres`, and `TitleCountries` in one `SELECT` failed on `STRING_AGG(DISTINCT ...)` — T-SQL doesn't support `DISTINCT` inside `STRING_AGG`. Investigating why I'd reached for `DISTINCT` in the first place revealed the real issue: joining three one-to-many relationships in a single query causes a fan-out (a title with 2 directors × 3 genres produces 6 joined rows before aggregation), which silently duplicates values in the aggregated output. Fixed by aggregating each relationship in its own correlated subquery instead, avoiding the fan-out entirely rather than patching over it.

Screenshot: *[before/after query + sample output showing the duplication]*

## Indexing

Indexes added after data load, targeting the three categories the project specifies:

| Column(s) | Table | Why |
|---|---|---|
| `director_id` | TitleDirectors | Reverse-direction JOIN lookup (PK only covers show_id → director_id) |
| `actor_id` | TitleActors | Same reasoning |
| `country_id` | TitleCountries | Same reasoning |
| `genre_id` | TitleGenres | Same reasoning |
| `type` | Titles | WHERE filter used in analysis queries |
| `release_year` | Titles | WHERE filter |
| `date_added` | Titles | ORDER BY column |
| `(release_year, date_added)` INCLUDE `title` | Titles | Composite index for the Step 6 optimisation target |

Full script:

```sql
CREATE INDEX IX_TitleDirectors_DirectorId ON TitleDirectors(director_id);
CREATE INDEX IX_TitleActors_ActorId       ON TitleActors(actor_id);
CREATE INDEX IX_TitleCountries_CountryId  ON TitleCountries(country_id);
CREATE INDEX IX_TitleGenres_GenreId       ON TitleGenres(genre_id);
GO

CREATE INDEX IX_Titles_Type        ON Titles(type);
CREATE INDEX IX_Titles_ReleaseYear ON Titles(release_year);
GO

CREATE INDEX IX_Titles_DateAdded ON Titles(date_added);
GO

CREATE INDEX IX_Titles_ReleaseYear_DateAdded 
    ON Titles(release_year, date_added) 
    INCLUDE (title);
GO
```

## Analysis Queries

Seven queries covering: yearly trend of additions, Movie/TV split, top genres, top directors, country distribution, rating distribution, most frequent actors.

Full script:

```sql
-- Titles added per year (trend)

SELECT YEAR(date_added) AS year_added, COUNT(*) AS titles_added
FROM Titles
WHERE date_added IS NOT NULL
GROUP BY YEAR(date_added)
ORDER BY year_added;

-- Movie vs TV Show split

SELECT type, COUNT(*) AS total
FROM Titles
GROUP BY type;

-- Top 10 genres

SELECT TOP 10 g.genre_name, COUNT(*) AS title_count
FROM TitleGenres tg
JOIN Genres g ON g.genre_id = tg.genre_id
GROUP BY g.genre_name
ORDER BY title_count DESC;

-- Top 10 directors

SELECT TOP 10 d.director_name, COUNT(*) AS title_count
FROM TitleDirectors td
JOIN Directors d ON d.director_id = td.director_id
GROUP BY d.director_name
ORDER BY title_count DESC;

-- Content distribution by country

SELECT TOP 10 c.country_name, COUNT(*) AS title_count
FROM TitleCountries tc
JOIN Countries c ON c.country_id = tc.country_id
GROUP BY c.country_name
ORDER BY title_count DESC;

-- Rating distribution

SELECT rating, COUNT(*) AS total
FROM Titles
WHERE rating IS NOT NULL
GROUP BY rating
ORDER BY total DESC;

-- Most frequently appearing actors

SELECT TOP 10 a.actor_name, COUNT(*) AS appearances
FROM TitleActors ta
JOIN Actors a ON a.actor_id = ta.actor_id
GROUP BY a.actor_name
ORDER BY appearances DESC;

```

**Insight notes** *(fill in with your actual results once you've run them and interpreted the output — don't leave this section as just numbers)*:
- Titles added per year: ...
- Top genre: ...
- [etc.]

## Query Optimisation

Target query:
```sql
SELECT t.title, t.release_year, t.date_added
FROM Titles t
WHERE t.release_year > 2018
ORDER BY t.date_added;
```

**Before:** *[screenshot: execution plan + logical reads, cache cleared via DBCC DROPCLEANBUFFERS / FREEPROCCACHE before measuring]*

**After** adding `IX_Titles_ReleaseYear_DateAdded (release_year, date_added) INCLUDE (title)`: *[screenshot: execution plan + logical reads]*

**Result:** *[state honestly what actually happened — Scan→Seek and reads dropped, or the optimizer still chose a scan at this row volume. Both are legitimate outcomes; explain which one occurred and why.]*

## Next Steps / What I'd Do Differently

- *[e.g. handle the `duration` column more precisely — currently kept as free text ("90 min" / "3 Seasons"), splitting into a numeric value + unit would enable numeric analysis]*
- *[e.g. investigate the volume of unparseable `date_added` values excluded by TRY_CONVERT, rather than silently nulling them]*
- *[e.g. test indexing benefit at a larger synthetic data volume, since the real dataset may be too small to show a clear before/after]*
