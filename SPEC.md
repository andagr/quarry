# Personal Knowledge Catalog Using DuckDB

## Goal

Build a lightweight local knowledge system that makes multiple collections of Markdown documents searchable and explorable by AI coding assistants, such as GitHub Copilot, without requiring:

- Vector databases
- Embeddings
- Custom MCP servers
- Cloud services
- External infrastructure

The system should periodically ingest one or more Markdown repositories into DuckDB and expose a simple CLI that an AI agent can invoke to retrieve relevant context.

The design prioritizes:

- Simplicity
- Explainability
- Fast local execution
- Easy future evolution
- Compatibility with a corporate laptop

The initial version intentionally relies on keyword and full-text search instead of embeddings.

## Problem Statement

The knowledge corpus consists of multiple repositories such as:

- Personal notes
- Daily journals
- Investigations
- Design explorations
- Team wiki
- Architecture documentation
- ADRs (Architecture Decision Records)
- Runbooks
- Technical experiments

The structure is highly custom and distributed across multiple locations, making it difficult for AI tools to discover relevant information efficiently.

The objective is to create a unified knowledge catalog that is:

- Searchable
- Source-aware
- Navigable
- Related-content aware
- Efficient for both humans and AI agents

## High-Level Architecture

```text
Personal Notes
Team Wiki
ADR Repository
Runbooks
Other Markdown Sources
        |
        v
     Indexer
        |
        v
      DuckDB
        |
        +-- Sources
        +-- Notes
        +-- Sections
        +-- Metadata
        +-- Links
        +-- File Index
        |
        v
     CLI Tool
        |
        v
 GitHub Copilot / Human Users
```

## Technology Choices

### Language

C# / .NET

Reasons:

- Familiar environment
- Strong filesystem support
- Good Markdown libraries
- Straightforward DuckDB integration
- Easy deployment as a self-contained executable

### Database

DuckDB

Reasons:

- Single-file database
- No service required
- Powerful SQL capabilities
- Strong analytics support
- Future option for vector search

## Design Principles

### 1. Preserve Markdown Structure

Markdown headings already define logical sections.

Instead of treating documents as blobs:

```markdown
# Investigation

## Findings

### Networking

## Open Questions
```

store each section separately.

Avoid arbitrary token-based chunking.

### 2. Preserve Source Ownership

Markdown repositories remain the source of truth.

DuckDB is only an index.

The indexer must never modify source files.

### 3. Support Multiple Knowledge Sources

The system is a knowledge catalog, not a single notes index.

Example sources:

```text
Personal Notes
Team Wiki
Architecture Docs
ADR Repository
Runbooks
```

New sources should be addable through configuration rather than code changes.

### 4. Optimize for Retrieval

Search should return:

- Relevant section
- Heading hierarchy
- Source
- Document path

It should avoid returning entire repositories or unnecessarily large files.

## Database Schema

### Sources

Represents each indexed repository.

```sql
CREATE TABLE sources (
    source_id   BIGINT PRIMARY KEY,
    source_name VARCHAR NOT NULL UNIQUE,
    root_path   VARCHAR NOT NULL,
    source_type VARCHAR,
    weight      INTEGER NOT NULL DEFAULT 100
);
```

Example source types:

- `personal`
- `wiki`
- `adr`
- `runbook`

The weight can later be used when ranking search results.

### Notes

Represents physical Markdown files.

```sql
CREATE TABLE notes (
    note_id       BIGINT PRIMARY KEY,
    source_id     BIGINT NOT NULL,
    document_key  VARCHAR NOT NULL UNIQUE,
    relative_path VARCHAR NOT NULL,
    full_path     VARCHAR NOT NULL,
    title         VARCHAR,
    modified_utc  TIMESTAMP,
    FOREIGN KEY (source_id) REFERENCES sources(source_id)
);
```

#### Document key

The document key is a stable source-qualified identifier, for example:

```text
personal:investigations/sql-retention.md
wiki:architecture/retention-strategy.md
adr:adr-023-duckdb-choice.md
```

This simplifies:

- References
- Backlinks
- Saved searches
- Related-content exploration
- AI citations
- Deduplication

### Sections

The section is the primary retrieval unit.

```sql
CREATE TABLE sections (
    section_id    BIGINT PRIMARY KEY,
    note_id       BIGINT NOT NULL,
    heading_path  VARCHAR,
    heading_level INTEGER,
    content       VARCHAR NOT NULL,
    FOREIGN KEY (note_id) REFERENCES notes(note_id)
);
```

Example heading paths:

```text
Investigation
Investigation > Findings
Investigation > Findings > SQL Server
```

Text before the first heading should also be retained, either as a document preamble section or under the document title.

### Metadata

Stores frontmatter attributes as normalized key-value records.

Input:

```yaml
---
tags:
  - azure
  - sql
status: active
---
```

Schema:

```sql
CREATE TABLE metadata (
    note_id BIGINT NOT NULL,
    key     VARCHAR NOT NULL,
    value   VARCHAR,
    FOREIGN KEY (note_id) REFERENCES notes(note_id)
);
```

Multi-value properties such as tags should be stored as one row per value.

### Links

Stores relationships between documents.

```sql
CREATE TABLE links (
    source_note_id BIGINT NOT NULL,
    target_note_id BIGINT,
    raw_target     VARCHAR NOT NULL,
    link_type      VARCHAR,
    FOREIGN KEY (source_note_id) REFERENCES notes(note_id),
    FOREIGN KEY (target_note_id) REFERENCES notes(note_id)
);
```

`target_note_id` may be null when a link cannot yet be resolved. Retaining `raw_target` allows it to be resolved after another source is added or a target document is created.

Example link forms:

```markdown
[[SQL Retention]]
```

```markdown
See event-store.md
```

```markdown
Related: ADR-023
```

### File Index

Supports incremental indexing.

```sql
CREATE TABLE file_index (
    source_id    BIGINT NOT NULL,
    relative_path VARCHAR NOT NULL,
    content_hash VARCHAR NOT NULL,
    modified_utc TIMESTAMP,
    indexed_utc  TIMESTAMP NOT NULL,
    PRIMARY KEY (source_id, relative_path),
    FOREIGN KEY (source_id) REFERENCES sources(source_id)
);
```

Only changed files should be reprocessed.

## Source Configuration

Sources should be configured outside the database schema so new directories can be added without code changes.

Example configuration:

```json
{
  "database": "knowledge.duckdb",
  "sources": [
    {
      "name": "personal",
      "path": "D:/Notes",
      "type": "personal",
      "weight": 80
    },
    {
      "name": "wiki",
      "path": "D:/Repos/TeamWiki",
      "type": "wiki",
      "weight": 100
    },
    {
      "name": "adrs",
      "path": "D:/Repos/Architecture/ADRs",
      "type": "adr",
      "weight": 120
    }
  ]
}
```

Each source name must be unique because it forms part of `document_key` and is used by CLI filters.

Useful optional source settings may later include:

- Include globs
- Exclude globs
- Default tags
- Whether hidden directories are scanned
- Whether Git-ignored files are excluded
- Source-specific link resolution rules

## Indexing Pipeline

### Step 1: Load configured sources

Read all source definitions and ensure corresponding records exist in the `sources` table.

### Step 2: Discover Markdown files

Recursively find:

```text
*.md
```

Apply configured exclusions for directories such as:

```text
.git
bin
obj
node_modules
```

### Step 3: Detect changes

For each file:

1. Determine the source and relative path.
2. Compare modification metadata with `file_index`.
3. Compute a content hash, such as SHA-256, when required.
4. Skip unchanged files.

The implementation should also detect deleted and renamed files. A rename may initially be treated as one deletion and one addition.

### Step 4: Parse Markdown

Extract:

- Title
- Heading hierarchy
- Document preamble
- Sections
- Frontmatter
- Cross-links

The title can be determined using a documented precedence rule, for example:

1. Frontmatter title
2. First level-one heading
3. Filename without extension

### Step 5: Update DuckDB

For each changed file, update its note, sections, metadata, links, and file-index entry in one transaction.

Replacing all derived rows for a changed document is acceptable for the initial version and is simpler than performing fine-grained section updates.

### Step 6: Handle deleted files

Remove notes and dependent index data for files that no longer exist under a configured source.

### Step 7: Resolve links

Resolve each raw link target against indexed documents.

Resolution should be deterministic and source-aware. Exact document keys and source-local relative paths should take precedence over fuzzy title matching.

Unresolved or ambiguous links should remain visible rather than being silently discarded.

### Step 8: Refresh search indexes

Refresh or rebuild full-text-search indexes only when required by the chosen DuckDB search implementation.

## CLI Design

The CLI is the primary interface for humans and AI agents.

Commands should support predictable, non-interactive output and meaningful process exit codes.

### Index

Index all configured sources:

```bash
notes index
```

Index a specific source:

```bash
notes index --source personal
```

Suggested output:

```text
Indexed: 12
Unchanged: 438
Deleted: 1
Errors: 0
```

### Search

Search all sources:

```bash
notes search "duckdb"
```

Suggested result:

```text
[personal] investigations/data-platform.md
Document key: personal:investigations/data-platform.md
Heading: Investigation > Findings
Score: 87

DuckDB performed better than ...
```

Search only one source:

```bash
notes search "retention" --source wiki
```

Useful options:

```text
--source <name>
--limit <number>
--format text|json
--path <glob-or-prefix>
```

JSON output is important for reliable agent consumption.

### Open

Display a complete note using its stable identifier:

```bash
notes open personal:investigations/sql-retention.md
```

Useful options:

```text
--format text|json
--section <section-id-or-heading-path>
```

### Related

Show directly related documents using resolved links and backlinks:

```bash
notes related personal:investigations/sql-retention.md
```

Results may span all configured sources.

### Recent

Show recently modified notes:

```bash
notes recent
```

Useful options:

```text
--source <name>
--limit <number>
--format text|json
```

### Statistics

Show catalog statistics:

```bash
notes stats
```

Example:

```text
Sources: 4
Files: 1432
Sections: 11284
Links: 3901
Tags: 216
```

## Search Strategy for Version 1

### Initial baseline

Implement a simple, explainable keyword-search baseline using SQL operations such as:

```sql
ILIKE
contains()
regexp_matches()
```

This establishes working retrieval before introducing a full-text-search extension.

### Full-text search

Evaluate DuckDB full-text-search support after the baseline is functional.

The search abstraction should allow the implementation to evolve without changing the CLI contract.

### Query handling

A multi-term query should distinguish between:

- Exact phrase matches
- All-term matches
- Any-term matches

Exact phrases and all-term matches should generally rank above loose any-term matches.

### Ranking

Use an understandable scoring model with positive signals such as:

- Exact phrase match
- Title match
- Heading match
- Filename or path match
- Section body match
- Number of matched query terms
- Source weight
- Optional modification-recency bonus

Example source weights:

```text
ADR Repository     120
Runbooks           100
Team Wiki           90
Personal Notes      80
Daily Notes         40
```

Weights must be configurable and should act as ranking signals rather than completely suppressing lower-weight sources.

### Search result requirements

Each result should include enough information for an agent to decide whether to open it:

- Source name
- Document key
- Relative path
- Title
- Heading path
- Short excerpt
- Score

Search excerpts should highlight or otherwise clearly expose matched terms in human-readable output. JSON output should retain the unformatted excerpt and matched terms separately.

## Link Extraction and Resolution

Support clearly recognizable link forms first:

```markdown
[[Project X]]
```

```markdown
[Retention strategy](architecture/retention.md)
```

```markdown
architecture/retention.md
```

```markdown
ADR-023
```

Link extraction and link resolution should be separate steps.

This enables:

- Forward links
- Backlinks
- Cross-source navigation
- Unresolved-link reporting
- Related-content discovery

Automatic detection of arbitrary filename-like text should be conservative to avoid creating excessive false links.

## Knowledge Graph Benefits

Over time, indexed links create a cross-source graph:

```text
Personal Investigation
    +-- Team Wiki Page
    +-- ADR
    +-- Runbook
```

This enables:

- Related-content discovery
- Knowledge-hub detection
- Cross-repository navigation
- Basic impact exploration

Version 1 only needs direct outgoing links and backlinks. Recursive graph traversal and visualization are future enhancements.

## Agent Usage

The intended agent workflow is:

1. Run `notes search` with a focused query.
2. Review concise section-level results.
3. Run `notes open` for the most relevant document or section.
4. Optionally run `notes related` to discover supporting material.

Example:

```bash
notes search "sql retention partition" --limit 5 --format json
notes open personal:investigations/sql-retention.md --format text
notes related personal:investigations/sql-retention.md --format json
```

The project should include a short agent instruction file explaining:

- When to use the CLI
- How to formulate focused searches
- Why search should precede broad filesystem scans
- When to open a complete note
- How to cite returned document keys and heading paths

## Error Handling

The CLI should:

- Continue indexing other files when one file fails.
- Report the affected source and relative path.
- Avoid leaving partially updated document data.
- Use nonzero exit codes for command-level failure.
- Support machine-readable errors in JSON mode.
- Clearly distinguish no matches from execution failure.

## Security and Privacy

The application is local-first.

Version 1 should:

- Read only configured directories.
- Avoid network access.
- Avoid sending note content to external services.
- Avoid modifying source documents.
- Store the DuckDB database at a configurable local path.
- Exclude the generated database and temporary files from source repositories by default.

The index contains copies of note content and must be treated as having the combined sensitivity of all configured sources.

## Future Analytics

A major benefit of DuckDB is analytical exploration.

### Notes over time

```sql
SELECT
    year(modified_utc) AS year,
    count(*) AS note_count
FROM notes
GROUP BY year
ORDER BY year;
```

### Most common tags

```sql
SELECT
    value,
    count(*) AS usage_count
FROM metadata
WHERE key = 'tag'
GROUP BY value
ORDER BY usage_count DESC;
```

### Most referenced documents

```sql
SELECT
    target_note_id,
    count(*) AS incoming_links
FROM links
WHERE target_note_id IS NOT NULL
GROUP BY target_note_id
ORDER BY incoming_links DESC;
```

### Source size

```sql
SELECT
    s.source_name,
    count(n.note_id) AS note_count
FROM sources s
LEFT JOIN notes n ON n.source_id = s.source_id
GROUP BY s.source_name
ORDER BY note_count DESC;
```

## Future Enhancements

The following are explicitly out of scope for version 1.

### Embeddings

Possible future table:

```sql
CREATE TABLE section_embeddings (
    section_id BIGINT PRIMARY KEY,
    embedding  FLOAT[1536]
);
```

Embeddings should only be introduced after actual searches demonstrate a need for semantic retrieval.

### DuckDB VSS

DuckDB vector similarity search may later complement keyword search.

A future hybrid retrieval model could combine:

```text
Keyword Search
      +
Vector Search
      +
Graph Relationships
      =
Final Ranking
```

### Additional future capabilities

- AI-generated summaries
- Topic clustering
- Graph visualization
- Duplicate-content detection
- Saved searches
- Watch mode for automatic incremental indexing
- Git metadata such as commit date or author
- Additional document-format extractors

## Non-Goals for Version 1

Version 1 will not:

- Modify or reorganize source notes.
- Run a background database service.
- Provide a custom MCP server.
- Generate embeddings.
- Require network or cloud access.
- Implement semantic search.
- Parse every possible Markdown extension.
- Infer arbitrary relationships using AI.
- Replace Git or the original repositories as the system of record.

## Success Criteria

The initial version is successful if:

1. Multiple Markdown repositories can be configured and indexed.
2. Adding another notes directory requires configuration only.
3. Incremental indexing processes changed, added, and deleted files correctly.
4. Markdown sections are extracted using their heading hierarchy.
5. Section-level search returns useful, source-aware results.
6. Every result identifies its source, document key, path, and heading.
7. GitHub Copilot can invoke the CLI and consume predictable text or JSON output.
8. A document can be opened by stable document key.
9. Direct links and backlinks can be queried across sources.
10. No embeddings, custom MCP servers, cloud services, or additional database servers are required.

## Recommended MVP Implementation Order

1. Create the .NET solution and CLI command structure.
2. Add configuration loading and validation.
3. Create the DuckDB schema and migrations/bootstrap logic.
4. Implement source synchronization.
5. Implement recursive Markdown-file discovery.
6. Implement change and deletion detection.
7. Parse titles, preambles, headings, and sections.
8. Persist notes and sections transactionally.
9. Implement `notes index`.
10. Implement basic keyword ranking and `notes search`.
11. Add deterministic text and JSON output.
12. Implement `notes open`.
13. Extract Markdown links and wiki-style links.
14. Resolve links and implement `notes related`.
15. Add frontmatter and metadata extraction.
16. Implement `notes recent` and `notes stats`.
17. Add source weighting and search filters.
18. Evaluate DuckDB full-text search against the baseline.
19. Evaluate whether embeddings are needed based on observed search failures.

## Core Philosophy

Treat **Markdown as the source of truth**, **headings as the natural section boundaries**, and **DuckDB as a searchable knowledge catalog spanning multiple repositories**.

Start with a small, explainable system built on:

- Markdown structure
- DuckDB
- Keyword and full-text search
- Explicit link relationships
- A stable CLI contract

Only introduce embeddings or vector search if real-world usage demonstrates a clear need for semantic retrieval beyond structured search and cross-linking.
