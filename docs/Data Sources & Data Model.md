# Data Sources & Data Model

This document describes the initial data strategy and data model decisions for AniCharaDB.

The goal is not to reproduce an existing anime database, but to build an independent and extensible character database capable of aggregating data from multiple sources over time.

## 1. Data Source Strategy

AniCharaDB is designed around a multi-source architecture.

Rather than depending structurally on a single external database, AniCharaDB maintains its own internal entities and uses external sources to populate, enrich, and cross-reference them.

### Primary Source — AniList

AniList will be used as the primary ingestion source during the first development phase.

It provides structured access to:

- Anime
- Characters
- Anime–character relationships
- Character roles
- Alternative titles
- Images
- Character metadata
- Voice actors
- MyAnimeList identifiers for many anime

Its GraphQL API also makes it possible to efficiently navigate relationships between anime and characters.

For the initial version of AniCharaDB:

```text
AniList → Primary ingestion source
```

AniList identifiers will be stored as external identifiers and will **not** become AniCharaDB's internal identifiers.

---

## 2. Secondary Sources

Additional sources may later be used to enrich the database.

### MyAnimeList / Jikan

Jikan provides access to public MyAnimeList data through an API.

It may be used to:

- enrich existing entities;
- recover information missing from AniList;
- cross-reference AniList and MyAnimeList entities;
- increase coverage.

Whenever possible, existing AniList → MyAnimeList mappings should be used before attempting automatic entity matching.

### Wikidata

Wikidata can provide another layer of entity resolution and external identifiers.

It may help connect AniCharaDB entities with identifiers from platforms such as:

```text
AniList
MyAnimeList
AniDB
Kitsu
Anime-Planet
Anime News Network
...
```

Wikidata is therefore considered primarily as a future **cross-reference and enrichment source**, rather than the initial ingestion source.

### Future Sources

The architecture should allow additional sources to be integrated without modifying the core identity of existing AniCharaDB entities.

Potential sources include:

```text
AniDB
Kitsu
Anime-Planet
Anime News Network
Wikidata
Other anime databases
```

The relevance, accessibility, licensing constraints, and data quality of each source will be evaluated before integration.

---

## 3. General Architecture

The initial ingestion strategy can be represented as:

```text
                  ┌──────────────┐
                  │   AniList    │
                  │ PRIMARY DATA │
                  └──────┬───────┘
                         │
                         ▼
                ┌────────────────┐
                │   AniCharaDB   │
                │ Canonical Data │
                └───────┬────────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Jikan      Wikidata    Future
           /MAL                    Sources
             │          │
             ▼          ▼
        Enrichment   ID Mapping
```

AniCharaDB remains the canonical representation.

External databases are considered **data providers**, not the source of AniCharaDB's internal identity.

---

## 4. Core Entities

The initial model revolves around two primary entities:

```text
Anime
Character
```

These entities have a many-to-many relationship.

An anime can contain many characters.

A character can appear in many anime.

This relationship therefore requires an intermediate entity:

```text
Anime
   │
   ▼
AnimeCharacter
   │
   ▼
Character
```

---

## 5. Anime

Initial conceptual representation:

```text
ANIME
─────
id
title
title_english
title_native
format
status
episodes
start_date
end_date
image_url
```

`id` represents an internal AniCharaDB identifier.

External identifiers such as AniList or MyAnimeList IDs must not be used as primary database identifiers.

---

## 6. Character

Initial conceptual representation:

```text
CHARACTER
─────────
id
name
native_name
description
gender
date_of_birth
image_url
```

This model is intentionally minimal at this stage.

Character attributes will be reviewed separately before defining the final database schema.

---

## 7. Anime–Character Relationship

The relationship between an anime and a character contains information of its own.

Initial representation:

```text
ANIME_CHARACTER
───────────────
anime_id
character_id
role
```

For example:

```text
Naruto Uzumaki
│
├── Naruto
│   └── role = MAIN
│
├── Naruto Shippuden
│   └── role = MAIN
│
└── Boruto
    └── role = SUPPORTING
```

For this reason, `role` belongs to the Anime–Character relationship rather than directly to the Character entity.

---

## 8. External Identifiers

AniCharaDB must remain independent from external database identifiers.

Instead of:

```text
Character
id = AniList character ID
```

AniCharaDB should use:

```text
Character
id = AniCharaDB internal ID
```

External identifiers are stored separately.

Conceptually:

```text
EXTERNAL_ID
───────────
entity_type
entity_id
source
external_id
```

Example:

```text
Character: Naruto Uzumaki

AniCharaDB ID → internal identifier
AniList       → external identifier
MyAnimeList   → external identifier
Wikidata      → external identifier
AniDB         → external identifier
...
```

This allows AniCharaDB to integrate multiple databases without becoming dependent on any single one.

---

## 9. Entity Resolution

One of the long-term challenges of AniCharaDB will be determining whether entities originating from different sources represent the same anime or character.

For example:

```text
AniList Character #X
        +
MAL Character #Y
        +
Wikidata Entity #Z
        │
        ▼
One AniCharaDB Character
```

During the initial development phase, AniList will act as the primary source of entities.

Known mappings such as AniList → MyAnimeList should be preferred over heuristic matching.

More advanced deduplication and entity-resolution mechanisms can be introduced later.

---

## 10. Initial Data Model

The current conceptual model is:

```text
┌──────────────────┐
│      ANIME       │
├──────────────────┤
│ id               │
│ title            │
│ title_english    │
│ title_native     │
│ format           │
│ status           │
│ episodes         │
│ start_date       │
│ end_date         │
│ image_url        │
└────────┬─────────┘
         │
         │
┌────────▼─────────┐
│ ANIME_CHARACTER  │
├──────────────────┤
│ anime_id         │
│ character_id     │
│ role             │
└────────┬─────────┘
         │
         │
┌────────▼─────────┐
│    CHARACTER     │
├──────────────────┤
│ id               │
│ name             │
│ native_name      │
│ description      │
│ gender           │
│ date_of_birth    │
│ image_url        │
└────────┬─────────┘
         │
         │
┌────────▼─────────┐
│   EXTERNAL_ID    │
├──────────────────┤
│ entity_type      │
│ entity_id        │
│ source           │
│ external_id      │
└──────────────────┘
```

This is a **conceptual model**, not yet the final database schema.

---

## 11. Current Decisions

The following decisions are currently established:

- AniList is the initial primary ingestion source.
- AniCharaDB owns its internal entity identifiers.
- External IDs are stored separately.
- Anime and Character have a many-to-many relationship.
- Character role belongs to the Anime–Character relationship.
- Secondary sources will enrich existing data progressively.
- Cross-source entity resolution will be introduced incrementally.
- The internal model must remain independent from the schema of any external API.

## 12. Next Decision

Before defining the physical database schema, the Character entity must be explored in more detail.

The next step is to determine:

**What information should AniCharaDB store about a character?**

Once the Character model is established, the conceptual model can be refined before selecting the final database implementation.