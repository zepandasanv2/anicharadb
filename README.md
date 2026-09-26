# AniCharaDB

**AniCharaDB** (*Anime Character Database*) is a project designed to build a centralized database containing **as many anime characters as possible**.

The goal is to progressively aggregate information from multiple data sources in order to create a unified, independent, and extensible anime character database.

## Project Goal

AniCharaDB aims to:

- Reference as many anime characters as possible
- Aggregate information from multiple data sources
- Detect and prevent duplicates across different databases
- Link each character to the anime they appear in
- Store different names, aliases, images, and available character information
- Associate characters with their voice actors
- Preserve the origin of collected data
- Allow new data sources to be added over time

The long-term goal is to make **AniCharaDB a unified anime character reference database**, rather than a simple copy of an existing database.

---

## Data Architecture

### General Principle

AniCharaDB maintains its **own internal character repository**.

A character is therefore not defined by their AniList, MyAnimeList, Kitsu, or any other external identifier.

Each character receives a unique AniCharaDB identifier that can then be linked to identifiers from external sources.

Example:

```text
Naruto Uzumaki
│
├── AniCharaDB ID: 12842
├── AniList ID: ...
├── MyAnimeList ID: ...
├── Kitsu ID: ...
└── Other sources: ...
```

This approach allows AniCharaDB to combine information from multiple databases without creating multiple records for the same character.

### Proposed Architecture

```text
External Sources
(AniList, MyAnimeList, Kitsu, others...)
             │
             ▼
       Data Ingestion
             │
             ▼
         Raw Data
          (JSON)
             │
             ▼
       Normalization
             │
             ▼
   Matching / Deduplication
             │
             ▼
         AniCharaDB
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
 Characters Anime  People
```

### Main Entities

AniCharaDB should eventually be able to represent:

- Characters
- Character names and aliases
- Character images
- Anime appearances
- Character roles within each anime (`MAIN`, `SUPPORTING`, etc.)
- Voice actors
- Voice acting languages
- Franchises
- Relationships between characters
- External identifiers from different data sources

### Source Management

Data retrieved from external sources remains separated from the canonical AniCharaDB entities.

```text
External Source
       │
       ▼
  Source Record
       │
       ▼
    Matching
       │
       ▼
AniCharaDB Entity
```

This makes it possible to track where information comes from and update each external source independently.

### Matching and Deduplication

One of the main challenges of AniCharaDB will be determining when characters from different sources represent the same character.

The matching system could progressively use several methods:

```text
Known Cross-Reference
        ↓
Exact Name + Anime
        ↓
Alias + Anime
        ↓
Probabilistic Matching
        ↓
Manual Validation
```

A confidence score could also be used to prevent AniCharaDB from automatically merging characters when a match is ambiguous.

---

## Storage

The currently proposed architecture uses:

**PostgreSQL** for structured data and relationships between entities.

**JSONB** for storing raw data retrieved from external sources.

Original source data can therefore be preserved before being transformed:

```text
API / Dataset
      ↓
Raw JSON
      ↓
Normalization
      ↓
Entity Resolution
      ↓
AniCharaDB
```

---

## Long-Term Vision

AniCharaDB is not intended to be a simple copy of AniList, MyAnimeList, Kitsu, or any other existing database.

The project aims to progressively build an **independent and unified anime character reference database** capable of combining information from multiple sources while preserving the origin of the data.

Each character has their own identity within **AniCharaDB**, independently of the external platforms used to enrich their information.
