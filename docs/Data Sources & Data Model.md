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

The Character core should contain information intrinsic to the character, rather than source-specific representations. Names and images are modeled separately so that multiple representations can coexist.

Current conceptual direction:

```text
CHARACTER
─────────
id
description
gender
date_of_birth
created_at
updated_at
```

The timestamps describe the lifecycle of the AniCharaDB record. They do not, by themselves, identify the source of an imported value.

This replaces the earlier simplified Character model with its single `name`, `native_name`, and `image_url` fields. It remains a **conceptual model**, not a final physical database schema. Attribute details and how conflicting source values become canonical values remain open.

### Character Names

A character may have multiple names, spellings, aliases, nicknames, native names, or translations. Each name should be represented separately:

```text
CHARACTER_NAME
--------------
id
character_id
name
type
language
```

Potential name types include:

```text
PRIMARY
NATIVE
ALIAS
NICKNAME
ALTERNATIVE
```

These types are provisional, not a final enum. Language representation and rules for selecting a primary name still need to be defined.

CharacterName supports an arbitrary number of names for each character. Fields such as `alternative_name_1`, `alternative_name_2`, and `alternative_name_3` impose a fixed limit and require structural changes whenever more names are needed. Separate name records also allow each name to carry its own type and language.

Conceptually:

```text
Character: Naruto Uzumaki
│
├── Naruto Uzumaki
│   type = PRIMARY
│
├── うずまきナルト
│   type = NATIVE
│
└── Number One Hyperactive Knucklehead Ninja
    type = ALIAS
```

### Character Images

Different providers may supply different images of the same character. Images should therefore be represented separately from the Character core:

```text
CHARACTER_IMAGE
---------------
id
character_id
url
source
is_primary
```

This allows AniCharaDB to retain images from multiple providers while selecting one as the primary representation when necessary. The selection policy and how `source` is represented remain open. An image's source alone does not resolve provenance for other character attributes.

### Data Provenance — Open Decision

As a multi-source database, AniCharaDB should likely retain the provenance of imported values. If AniList supplies one value and MyAnimeList supplies another, the system should eventually be able to determine which source supplied each value, even when a canonical value is selected.

This will be important for:

- conflict resolution;
- entity merging;
- data quality;
- debugging ingestion;
- refreshing data;
- determining which source should take precedence.

External identifiers link entities across providers, but do not identify the origin of every attribute value. The granularity of provenance, how competing values and their history are retained, and how source precedence is decided still need to be designed. No final provenance storage model or conflict-resolution implementation has been chosen.

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

The current conceptual relationships are:

```mermaid
erDiagram
    Anime ||--o{ AnimeCharacter : has
    Character ||--o{ AnimeCharacter : appears_in
    Character ||--o{ CharacterName : has
    Character ||--o{ CharacterImage : has
    Anime |o--o{ ExternalId : identified_by
    Character |o--o{ ExternalId : identified_by
```

AnimeCharacter carries the role for each anime appearance. CharacterName and CharacterImage allow multiple representations without adding repeated fields to Character.

Each ExternalId refers to one internal entity, selected conceptually by `entity_type` and `entity_id`: Anime or Character in the current model, not both. The two optional links show these alternative targets; the diagram does not encode their mutual exclusivity or prescribe physical foreign keys.

This is a **conceptual model**, not the final database schema. Field sketches are documented in the entity sections above. The diagram does not define database types, required minimum counts, uniqueness constraints, or a provenance implementation.

---

## 11. Current Decisions

The following decisions are currently established:

- AniList is the initial primary ingestion source.
- AniCharaDB owns its internal entity identifiers.
- External IDs are stored separately.
- Anime and Character have a many-to-many relationship.
- Character role belongs to the Anime–Character relationship.
- The Character core separates intrinsic attributes and record timestamps from names and images.
- CharacterName supports an arbitrary number of names, with a type and language for each name; the name types remain provisional.
- CharacterImage retains images from different sources and allows a primary representation to be selected; selection rules remain open.
- Secondary sources will enrich existing data progressively.
- Cross-source entity resolution will be introduced incrementally.
- The internal model must remain independent from the schema of any external API.

## 12. Open Day 1 Questions

The separation of Character, CharacterName, and CharacterImage establishes the current conceptual direction. Further decisions are needed before a physical schema or implementation can be defined:

- Which additional intrinsic Character attributes are needed, and how should incomplete or conflicting values be represented?
- What are the final name types, language conventions, and primary-name selection rules?
- How should image sources be represented, and how should the primary image be selected?
- How should value provenance, competing values, and source precedence be modeled, including during refreshes?
- How should full cross-source merging and deduplication work beyond known external-ID mappings?
- How should voice actors and other planned entities relate to characters and anime?
- What are the limitations, coverage, reliability, and licensing constraints of each source before integration?
- Which database and ingestion technologies should be selected, and what should the API and synchronization architecture be?

Database selection remains open. These conceptual decisions do not establish a final physical schema, migrations, or application implementation.
