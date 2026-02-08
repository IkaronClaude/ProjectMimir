# Mimir Project Tracker

## Current Phase: Core Features Complete
SHN + raw table import, SQL query/edit/shell, constraint validation, and SHN build all working.
1234 tables import successfully. Building out definitions system and tooling.

---

## Architecture

```
[Server Files] --import--> [JSON Project] --edit/query--> [JSON Project] --build--> [Server Files]
   SHN/txt                  git-tracked       SQL/CLI                       SHN/txt
```

**JSON as the canonical intermediate format.** Each source table becomes a single `.json`
file containing header (metadata), column definitions, and all row data. These files live
in a git repo, are human-readable, and diffable. The CLI `import` command converts server
files into a Mimir project, and `build` converts back. All editing/querying happens against
the project via SQLite.

A Mimir project directory looks like:
```
my-server/
  mimir.json                 # project manifest (table name → file path)
  mimir.definitions.json     # table definitions + constraint rules
  data/
    shn/
      Shine/
        ItemInfo.json         # { header, columns, data: [{...}, ...] }
        MobInfo.json
        Shine/View/
          ItemViewInfo.json
    rawtable/
      Shine/MobRegen/
        AdlVal01_MobRegenGroup.json
      Shine/NPCItemList/
        AdlAertsina_Tab00.json
      Shine/Script/
        AdlF_Script.json
      Shine/World/
        ChrCommon_Common.json
    rawtable-define/
      ServerInfo/
        ServerInfo_SERVER_INFO.json
      Shine/
        DefaultCharacterData_CHARACTER.json
```

Directory structure mirrors the source 9Data layout.

**Definitions file** (`mimir.definitions.json`):
```json
{
  "tables": {
    "ItemInfo": {
      "idColumn": "ID",
      "keyColumn": "InxName",
      "columnAnnotations": {
        "AC": { "displayName": "Armor Class", "description": "Defense stat, +N when equipped" }
      }
    }
  },
  "constraints": [
    {
      "description": "NPC shop items reference ItemInfo",
      "match": { "path": "rawtable/Shine/NPCItemList/**", "column": "Column*" },
      "foreignKey": { "table": "ItemInfo" },
      "emptyValues": ["-", ""]
    }
  ]
}
```
- **Table definitions**: `idColumn` (INTEGER PRIMARY KEY), `keyColumn` (UNIQUE index, nullable),
  `columnAnnotations` (displayName + description for each column)
- **Constraints**: Glob patterns on path/table/column, FK targets with shorthand
  (omit column → keyColumn, `@id` → idColumn), `{file}` template
- **FK enforcement**: Constraints become real SQLite FOREIGN KEY clauses at load time,
  with topological table ordering and empty value → NULL mapping

---

## Completed

### P0 - Foundation
- [x] Solution structure, Directory.Build.props, all csproj files
- [x] `Mimir.Core` - Models (ColumnType, ColumnDefinition, TableSchema, TableFile, TableEntry), IDataProvider, IProjectService, DI
- [x] `Mimir.Shn` - ShnCrypto (XOR cipher), ShnDataProvider (read/write), DI
- [x] `Mimir.Sql` - SqlEngine (SQLite in-memory, load/query/extract), DI
- [x] `Mimir.Cli` - import/build/query/dump/analyze-types/validate/edit/shell commands
- [x] Switched from JSONL to single .json per table (header+columns+data)
- [x] Fixed type 10 (0x0A) = 4-byte padded string (map keys like Rou, Eld)
- [x] Fixed type 29 (0x1D) = uint64 flags (targets, activities)
- [x] Fixed type 26 reading: proper null-terminated string (not hacky rowLength calc)
- [x] Updated IDataProvider to return `IReadOnlyList<TableEntry>` (multi-table files)
- [x] **Full import: 218/219 SHN tables** (only QuestData.shn fails - different format)

### P1 - Raw Tables
- [x] Identified 2 text formats across all of 9Data
- [x] `#table/#columntype/#columnname/#record` parser (MobRegen, NPCItemList, Script, World, etc.)
- [x] `#DEFINE/#ENDDEFINE` parser (ServerInfo, DefaultCharacterData configs)
- [x] `RawTableDataProvider` with auto-detection
- [x] DI registration
- [x] **Full import: 1234 total tables** (218 SHN + ~1016 raw tables) from all of 9Data
- [x] Directory structure preserved in project layout

### P2 - Definitions & Constraints
- [x] `ProjectDefinitions` model (table keys, column annotations, constraint rules)
- [x] `DefinitionResolver` (glob matching, `{file}` template, pattern resolution, key column resolution)
- [x] `validate` CLI command (resolves constraints, checks via SQL, reports violations)
- [x] Constraint-backed SQLite: FOREIGN KEY clauses in CREATE TABLE
- [x] Topological sort for table load order (referenced tables first)
- [x] Empty value → NULL mapping (so FK checks skip "-" and "" values)
- [x] Deferred index creation after bulk load (graceful fallback on duplicate data)
- [x] `idColumn` / `keyColumn` designation per table
- [x] Column annotations with `displayName` and `description`
- [x] FK shorthand: omit column → uses keyColumn, `@id` → uses idColumn

### P3 - Edit & Shell
- [x] `edit` command: single-shot SQL modification with automatic save back to JSON
- [x] `shell` command: interactive SQL REPL (sqlcmd-style)
  - `.tables` - list all loaded tables
  - `.schema TABLE` - show column definitions with annotations
  - `.save` - save all tables back to JSON
  - `.quit` - quit with dirty-check prompt
  - SELECT/PRAGMA/EXPLAIN → query display; other SQL → execute with row count
  - Dirty tracking (prompts to save on quit if changes made)

### Build
- [x] `build` command for SHN: JSON → SHN binary output
- [x] Build preserves original directory structure from manifest paths

---

## Active Work

### Raw Table Write
- [ ] `RawTableDataProvider.WriteAsync` (currently `NotImplementedException`)
- [ ] Round-trip test: import .txt → JSON → build .txt

### Round-trip Verification
- [ ] Byte-identical SHN round-trip testing
- [ ] Unit tests with synthetic data (no copyrighted game data in git)

---

## Planned

### Definitions per Project Type
> Currently definitions are per-project (`mimir.definitions.json` in the project root).
> In the future, definitions should be per **project type** and shipped with the application.
> e.g. "Fiesta Online" project type includes all table key/column metadata out of the box.
> The definitions file can also track the SHN encryption key if different from default.

### SHN Type Refinements
- [ ] Deep analysis: check signedness of types 20/21/22 (signed vs unsigned)
- [ ] Deep analysis: string empty patterns (dash=key vs empty=text)
- [ ] Investigate QuestData.shn (different format, crypto buffer overflow)

### File Exclusions
- [ ] "gitignore"-style exclusion patterns in definitions file
- [ ] Exclude files that look like tables but shouldn't be editable (e.g. ServerInfo.txt)

### Metadata Build-out
- [ ] Go through all SHN/txt files and build column annotations for every table
- [ ] Document what each column means across all game tables

---

## SHN Type Codes

| Type | Hex  | Size | Count | Meaning |
|------|------|------|-------|---------|
| 1    | 0x01 | 1    | 310   | byte (bools, small enums) |
| 2    | 0x02 | 2    | 525   | uint16 (IDs, rates) |
| 3    | 0x03 | 4    | 319   | uint32 (IDs, indices) |
| 5    | 0x05 | 4    | 3     | float (scales) |
| 9    | 0x09 | var  | 364   | padded string (null-term, zero-padded to length) |
| 10   | 0x0A | 4    | 1     | padded string (always 4-byte map keys: Rou, Eld, Urg) |
| 11   | 0x0B | 4    | 166   | uint32 flags (0 = none) |
| 12   | 0x0C | 1    | 8     | byte (mode/type) |
| 13   | 0x0D | 2    | 7     | uint16 (type IDs) |
| 16   | 0x10 | 1    | 1     | byte flags |
| 20   | 0x14 | 1    | 17    | sbyte (likely signed, TBD) |
| 21   | 0x15 | 2    | 144   | int16 (likely upgrade indices, 0=none) |
| 22   | 0x16 | 4    | 65    | int32 (likely time values) |
| 24   | 0x18 | 32+  | 19    | padded string (string keys) |
| 26   | 0x1A | var  | 3     | null-terminated variable-length string |
| 29   | 0x1D | 8    | 5     | uint64 flags (targets, activities) |

SHN indices are 1-based. Types 1-4 are unsigned, types 20-22 are likely signed counterparts.
String columns: `-` when empty = key/index, `""` when empty = free text.

---

## Stretch Goals

### CI/CD
- [ ] Dockerfile for `mimir` CLI
- [ ] Example GitHub Actions workflow: validate + build on push
- [ ] Exit non-zero on validation failure

### Data Validation / Linting
Built-in rules that detect common mistakes and inconsistencies:
- **Orphaned items** - Items in ItemInfo not in any drop table AND not sold by any NPC
- **Incomplete sets** - Sets missing a piece for a class, or items breaking set naming convention
- **Broken references** - Drop tables referencing nonexistent item IDs, quests rewarding invalid items
- **Stat anomalies** - Items with stats wildly out of range for their level bracket
- **Missing localizations** - Items/quests/NPCs with empty or placeholder name strings
- **Duplicate entries** - Same item ID appearing multiple times, duplicate drop table entries
- **Unreachable content** - Maps with no warp points, mobs that don't spawn anywhere

### Scripted Workflows
Composable CLI commands for common multi-step operations:
- Clone map
- Clone mob type (including skills, spawn groups, etc.)
- Clone armor set (duplicate items, adjust names/stats, add to drop tables)
- Calculate stat formulas from given data, apply with multiplier to new sets
- Quick drop group creation & assignment (e.g. apply to all mobs with level > 120)
- Quick enhancement stat adjustment, enhancement groups
- Quick NPC shop editing (e.g. "add armor set to NPC X")
- Multi-layer sets (e.g. "SetGroup XYZ, Rarity Orange, Set Bonus, Level 115, Formula +10 offset")
- Quick mob cloning for dungeons/instances/KQs (like Nest Boogie for Leviathan's Nest)
- Time limit editing (e.g. remove time limit from skins)
- Visual monster spawn group editor

### Client-side Asset Pipeline
- Custom folder for cloned set textures/nifs
- On build, copy client-side data to correct paths
- "Edit overrides" - if cloned set has a matching gcf/psd, use that instead
- Possible automated recoloring (find layer, dye, export to png on build)

### Quest System
- Reverse engineer quest file format
- Quest reader/writer integrated into Mimir
- Cross-reference validation (quest rewards vs item DB, quest mobs vs mob DB)

### Scenario Scripting
- Document the custom scripting language used for instance scripts
- Build a parser / AST for the scenario language
- Compiler / syntax checker
- Testing tools (dry-run a scenario script against game data)

### IDE Tooling (Far Future)
- VSCode extension with language support for scenario scripts
- Intellisense / autocomplete (item IDs, mob IDs, map names from loaded data)
- Hover info showing resolved references (hover item ID → show item name/stats)
- Linting integration (red squiggles for broken references)

---

## Notes
- SHN format: 32-byte header + XOR cipher + binary column/row data (see FiestaLib source)
- Existing SHN Editor source at: `Z:/Odin Server Files/Fiesta Tool Project Source/SHN Editor/`
- Spark Editor reference: https://github.com/Wicious/Spark-Editor
- Server data at: `Z:/Odin Server Files/Server/9Data/` (~220 .shn + hundreds of .txt files)
- Server and client share some files (ItemInfo.shn shared, ItemViewInfo.shn client-only, MobRegen server-only)
- Quest files use a different format from SHN - TBD
- Scenario files use a custom language (not Lua) - TBD
- .NET 10 SDK available on build VM (10.0.102)
- GitHub: https://github.com/IkaronClaude/ProjectMimir
