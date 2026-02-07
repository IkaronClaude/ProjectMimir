# Mimir Project Tracker

## Current Phase: Foundation
Building the core library with SHN support, JSONL project format, and SQL query engine.

---

## Architecture

```
[Server Files] --import--> [JSONL Project] --edit/query--> [JSONL Project] --build--> [Server Files]
   SHN/txt                   git-tracked         SQL/CLI                        SHN/txt
```

**JSON as the canonical intermediate format.** Each source table becomes a single `.json`
file containing header (metadata), column definitions, and all row data. These files live
in a git repo, are human-readable, and diffable. The CLI `import` command converts server
files into a Mimir project, and `build` converts back. All editing/querying happens against
the project via SQLite.

A Mimir project directory looks like:
```
my-server/
  mimir.json              # project manifest (table name → file path)
  data/
    shn/
      ItemInfo.json        # { header, columns, data: [{...}, ...] }
      MobInfo.json
      ActiveSkill.json
      ...
    tables/
      DropTable_Zone01.json
      DefaultCharacterData.json
      ...
```

Each table file:
```json
{
  "header": { "tableName": "ItemInfo", "sourceFormat": "shn", "metadata": {...} },
  "columns": [ { "name": "ID", "type": "UInt16", "length": 2, "sourceTypeCode": 2 }, ... ],
  "data": [ { "ID": 1, "InxName": "Sword", ... }, ... ]
}
```

**Why this format?**
- Schema co-located with data (self-describing files)
- Human-readable, diffable in git
- No binary files in the repo
- Multiple people can work on different tables simultaneously
- CI/CD can validate and build automatically (`mimir build` in GitHub Actions)
- SQLite with foreign keys provides data integrity enforcement

---

## Active Work

### P0 - Foundation
- [x] Solution structure, Directory.Build.props, all csproj files
- [x] `Mimir.Core` - Models (ColumnType, ColumnDefinition, TableSchema, TableData, TableFile), IDataProvider, IProjectService, DI
- [x] `Mimir.Shn` - ShnCrypto (XOR cipher), ShnDataProvider (read/write), DI
- [x] `Mimir.Sql` - SqlEngine (SQLite in-memory, load/query/extract), DI
- [x] `Mimir.Cli` - import/build/query/dump/analyze-types commands
- [x] Smoke test: imported 215/220 SHN tables successfully on first run
- [ ] **IN PROGRESS**: Switch from JSONL to single .json per table (header+columns+data)
- [ ] Fix type 10 (0x0A) = 4-byte padded string, type 29 (0x1D) = two packed uint32s
- [ ] Fix QuestData.shn (different format, needs investigation)
- [ ] Update ProjectService for new JSON format
- [ ] Update CLI commands for new format
- [ ] Unit tests (must generate synthetic SHN files - no copyrighted game data in git)
- [ ] Verify round-trip: SHN → import → JSON → build → SHN → byte-compare

### SHN Type Codes (from analysis)
| Type | Hex  | Size | Count | Meaning |
|------|------|------|-------|---------|
| 1    | 0x01 | 1    | 310   | byte (bools, small enums) |
| 2    | 0x02 | 2    | 525   | uint16 (IDs, rates) |
| 3    | 0x03 | 4    | 319   | uint32 (IDs, indices) |
| 5    | 0x05 | 4    | 3     | float (scales) |
| 9    | 0x09 | var  | 364   | padded string |
| 10   | 0x0A | 4    | 1     | padded string (4-char codes) |
| 11   | 0x0B | 4    | 166   | uint32 (enum indices) |
| 12   | 0x0C | 1    | 8     | byte (mode flags) |
| 13   | 0x0D | 2    | 7     | int16 (IDs) |
| 16   | 0x10 | 1    | 1     | byte (bitmask) |
| 20   | 0x14 | 1    | 17    | sbyte |
| 21   | 0x15 | 2    | 144   | int16 |
| 22   | 0x16 | 4    | 65    | int32 |
| 24   | 0x18 | 32+  | 19    | padded string |
| 26   | 0x1A | var  | 3     | variable-length null-terminated string |
| 29   | 0x1D | 8    | 5     | two packed uint32s (enum pairs) |

Full analysis: see `shn_type_analysis.txt` on shared drive.

### P1 - Raw Tables
- [ ] `Mimir.RawTables` - Text-based file parsing (drop tables, configs)
- [ ] Identify all text formats in 9Data and document their structure
- [ ] Import/build support alongside SHN tables

### P2 - CI/CD
- [ ] Dockerfile for `mimir` CLI
- [ ] Example GitHub Actions workflow: validate + build on push
- [ ] `mimir validate` command that runs all lint rules and exits non-zero on failure

---

## Stretch Goals

### Data Validation / Linting
Built-in rules that detect common mistakes and inconsistencies:
- **Orphaned items** - Items in ItemInfo not in any drop table AND not sold by any NPC
- **Incomplete sets** - Sets missing a piece for a class, or items breaking set naming convention
- **Broken references** - Drop tables referencing nonexistent item IDs, quests rewarding invalid items, NPCs selling missing items
- **Stat anomalies** - Items with stats wildly out of range for their level bracket
- **Missing localizations** - Items/quests/NPCs with empty or placeholder name strings
- **Duplicate entries** - Same item ID appearing multiple times, duplicate drop table entries
- **Unreachable content** - Maps with no warp points, mobs that don't spawn anywhere, KQ definitions with no trigger

### Scripted Workflows
Composable CLI commands for common multi-step operations:
- "Clone set X for class Y" - duplicate items, adjust names/stats, add to drop tables
- "Create KQ reward table" - gather items, build drop table, assign to KQ definition
- "Audit zone Z" - report all mobs, drops, NPCs, quests in a zone with completeness checks

### Quest System
- Reverse engineer quest file format (find source code online or RE together)
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
- Server data at: `Z:/Odin Server Files/Server/9Data/Shine/` (~220 .shn files)
- Quest files use a different format from SHN - TBD
- Scenario files use a custom language (not Lua) - TBD
- .NET 10 SDK available on build VM
