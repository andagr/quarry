# Quarry Implementation Plan

This plan implements version 1 of [SPEC.md](SPEC.md). SPEC.md is authoritative; this document describes how and in which order to build it.

## Solution Layout

```text
src/
  Quarry.slnx
  Quarry/                  CLI executable, packaged as a global .NET tool
    Cli/                   Command definitions, help text, exit codes
    Configuration/         ~/.quarry/config.json model, loading, validation, saving
    Storage/               DuckDB connection, schema bootstrap, repositories
    Indexing/              Discovery, change detection, indexing pipeline
    Parsing/               Markdown, frontmatter and link extraction
    Linking/               Link resolution
    Search/                Query parsing, search, excerpts
    Output/                Text and JSON formatters
  Quarry.Tests/            MSTest + AwesomeAssertions
```

A single application project keeps things simple. Folders can be split into projects later if needed.

## Technology

| Concern | Choice |
|---|---|
| Runtime | .NET 11, C# 15 (union types), nullable enabled |
| Packaging | `PackAsTool`, `ToolCommandName` = `quarry` |
| CLI parsing | System.CommandLine |
| Database | DuckDB.NET.Data.Full |
| Markdown | Markdig (with YAML frontmatter extension) |
| Frontmatter | YamlDotNet |
| JSON | System.Text.Json (source-generated contexts) |
| Tests | MSTest, AwesomeAssertions |

### Union types

Use C# 15 unions where a value is one of several distinct outcomes, for example:

- File index outcome: indexed, unchanged, deleted, failed
- Link resolution: resolved, unresolved, ambiguous
- Search match category: exact phrase, all terms, any term
- Command result: success, no matches / not found, error

## Cross-Cutting Decisions

- **Paths:** stored with `/` separators and compared case-sensitively. Conversion to and from OS paths happens only at the filesystem boundary.
- **Home directory:** `~` in config values is expanded to the user profile directory.
- **Default database:** `~/.quarry/quarry.duckdb`.
- **Database access:** `index` opens read-write; all query commands open read-only. A lock conflict produces a clear error.
- **Parameters:** all user-provided values are passed as DuckDB parameters, never concatenated into SQL. `ILIKE` wildcards in search terms are escaped.
- **IDs:** generated from DuckDB sequences. `note_id` is stable across re-indexing (update in place by `document_key`).
- **Exit codes:**
  - `0` success with results
  - `1` no matches or document not found
  - `2` error (invalid arguments, configuration, database or indexing failure)
- **JSON mode:** errors are written as JSON to stdout with a nonzero exit code; text mode writes errors to stderr.
- **Network:** only `quarry duckdb install-extension` may access the network. Extension autoinstall/autoload is disabled on every connection.

## Phases

Each phase ends with passing tests and a working build.

### Phase 0 — Foundations and spikes

- [ ] Set `OutputType` to `Exe`, enable tool packaging, set language version for C# 15 unions.
- [ ] Create `Quarry.Tests` with MSTest and AwesomeAssertions; add to `Quarry.slnx`.
- [ ] Add `PLAN.md` to the solution items.
- [ ] Spike: confirm C# 15 union types compile and pattern-match with the installed .NET 11 SDK.
- [ ] Spike: confirm DuckDB foreign-key behaviour:
  - Updating non-key columns of a `notes` row referenced by `sections`/`links`.
  - Deleting in the order `links`, `metadata`, `sections`, `notes` inside one transaction.
  - If updates of referenced rows are rejected, record the workaround here before continuing.

### Phase 1 — CLI skeleton and configuration

- [ ] Root command with System.CommandLine, global `--format text|json` handling and exit-code mapping.
- [ ] Configuration model, load/save of `~/.quarry/config.json`, creating the directory when missing.
- [ ] Validation: unique source names, valid source name characters (no `:`), non-empty paths.
- [ ] Configuration commands (see [Command Shapes](#command-shapes)). `source remove --purge` depends on Phase 2 storage.
- [ ] Tests: config round-trip, validation errors, `~` expansion.

### Phase 2 — Schema and storage

- [ ] Schema bootstrap as embedded SQL, with a `schema_version` table for future migrations.
- [ ] Tables as in SPEC.md, including `sections.ordinal`.
- [ ] Connection factory (read-write vs read-only, extension autoinstall disabled).
- [ ] Source synchronisation: upsert configured sources; warn for missing paths and for database sources not in config; never delete them.
- [ ] Tests run against temporary database files.

### Phase 3 — Markdown parsing

- [ ] Title precedence: frontmatter `title`, first H1, filename without extension.
- [ ] Sections by heading hierarchy using Markdig (headings inside code blocks are ignored; ATX and setext headings supported).
- [ ] Preamble section: empty `heading_path`, level 0.
- [ ] Section content is the section's own text only; empty string when it has none.
- [ ] Heading path joined with ` > `; `ordinal` is document order.
- [ ] Frontmatter: scalars as one row, lists as one row per value, nested maps flattened to dotted keys.
- [ ] Link extraction: `[[Wiki]]` and `[text](path.md)`; external URLs ignored; `#anchor` retained in `raw_target`.
- [ ] Tests: table-driven Markdown fixtures covering each rule.

### Phase 4 — Indexing pipeline and `quarry index`

- [ ] Recursive `*.md` discovery with default exclusions (`.git`, `bin`, `obj`, `node_modules`).
- [ ] Change detection: compare `modified_utc`; when different, compare SHA-256; if the hash matches, only update `file_index`.
- [ ] Per-file transaction: upsert note (stable `note_id`), replace sections, metadata and outgoing links, update `file_index`.
- [ ] Deletions: for available sources, remove notes whose files are gone. Null incoming `target_note_id`s, then delete in FK order.
- [ ] Moved files are handled as delete + add.
- [ ] Per-file failures are reported with source and relative path; indexing continues.
- [ ] `--source <name>` limits discovery to one source.
- [ ] Summary output: Indexed, Unchanged, Deleted, Errors; exit code `2` if any file failed.
- [ ] Tests: add, modify, touch-without-change, delete, move, failing file, missing source.

### Phase 5 — Link resolution

- [ ] Re-resolve all links at the end of every index run.
- [ ] Rules (SPEC.md): own source first, then other sources; Markdown links by relative path; wiki links by filename then title; anchors stripped; ambiguous left unresolved.
- [ ] Tests: same-source vs cross-source precedence, ambiguity, links becoming resolved after a target is added, links becoming unresolved after a target is deleted.

### Phase 6 — `quarry search`

- [ ] Query parsing into the full phrase and individual terms.
- [ ] Baseline SQL using `ILIKE` across section content, heading path, title and relative path.
- [ ] Match category per section: exact phrase, all terms, any term.
- [ ] Ordering: category, then most recently modified, then document key, then ordinal.
- [ ] Options: `--source`, `--path <glob-or-prefix>`, `--limit` (default 10), `--format`.
- [ ] Excerpt around the first match (bounded length); matched terms highlighted in text output and listed separately in JSON.
- [ ] Search logic behind an interface so FTS can replace the baseline later without changing the CLI contract.
- [ ] Tests: categories and ordering, filters, wildcard escaping, no-match exit code.

### Phase 7 — `quarry open`

- [ ] Open by document key; reconstruct the document from sections in `ordinal` order (no filesystem access).
- [ ] `--section <heading-path>`; duplicate heading paths addressed as `Path#2`, `Path#3`.
- [ ] JSON output includes note fields, metadata and sections.
- [ ] Tests: full document, section selection, unknown key exit code.

### Phase 8 — `quarry related`, `quarry recent`, `quarry stats`

- [ ] `related`: outgoing links (resolved and unresolved) and backlinks, across sources.
- [ ] `recent`: by `modified_utc` descending, with `--source`, `--limit`, `--format`.
- [ ] `stats`: sources, files, sections, links (resolved/unresolved), tags.
- [ ] Tests for each command.

### Phase 9 — `quarry duckdb`

- [ ] `quarry duckdb version`: prints the embedded DuckDB version.
- [ ] `quarry duckdb install-extension <name>`: runs `INSTALL` via the embedded engine.
- [ ] Helper that loads a required extension and, if missing, reports how to install it. (No extension is required in v1.)

### Phase 10 — Help and documentation

- [ ] `quarry --help` includes the agent workflow: when to use quarry, focused searches, search before filesystem scans, when to open a note, citing document keys and heading paths.
- [ ] README: purpose, installation as a global tool, configuration, commands, the one-line global Copilot instruction pointing to `quarry --help`, extension installation, privacy notes.

### Later (post-v1, per SPEC.md)

- Evaluate DuckDB FTS against the baseline.
- Evaluate embeddings only if real searches show a need.

## Command Shapes

Confirmed configuration commands:

```bash
quarry source add <path> --name <name>
quarry source list [--format text|json]
quarry source remove <name> [--purge]
quarry source relocate <name> <new-path>

quarry config path
quarry config show [--format text|json]
quarry config set <key> <value>

quarry duckdb version
quarry duckdb install-extension <name>
```

Behaviour:

- `source add` requires `--name` and validates that the path exists and the name is unique and valid.
- `source remove` removes the source from config only; indexed data is retained.
- `source remove --purge` also deletes the source's indexed data (links, metadata, sections, notes, file index, source row) in one read-write transaction.
- `source relocate` is explicit; quarry never detects moved sources automatically. It validates that the new path exists and updates config. The next `index` updates `sources.root_path`; relative paths are unchanged, so document keys and `note_id`s are preserved.
- `config set` accepts known keys only (v1: `database`) and rejects unknown keys.
- `config path` prints the location of `~/.quarry/config.json`.

## Risks

| Risk | Mitigation |
|---|---|
| DuckDB FK restrictions on updates | Phase 0 spike before schema work |
| C# 15 unions on a pre-release SDK | Phase 0 spike; fall back to records + pattern matching |
| Concurrent `index` and queries | Read-only query connections; clear lock error |
| Large notes in `open` output | `--section` for targeted reads |
| Link false positives | Only explicit link forms in v1 |
