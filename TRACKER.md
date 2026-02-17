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
  data/
    shn/
      Shine/
        ItemInfo.json         # { header, columns, data: [{...}, ...] }
        MobInfo.json
        Shine/View/
          ItemViewInfo.json
    shinetable/
      Shine/MobRegen/
        AdlVal01_MobRegenGroup.json
      Shine/NPCItemList/
        AdlAertsina_Tab00.json
      Shine/Script/
        AdlF_Script.json
      Shine/World/
        ChrCommon_Common.json
    configtable/
      ServerInfo/
        ServerInfo_SERVER_INFO.json
      Shine/
        DefaultCharacterData_CHARACTER.json
  mimir.definitions.json     # table definitions + constraint rules (walks up from project dir)
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
      "match": { "path": "data/shinetable/Shine/NPCItemList/**", "column": "Column*" },
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
- [x] **Full import: 219/219 SHN tables** (QuestData.shn handled by QuestDataProvider)

### P1 - Shine Tables (text formats)
- [x] Identified 2 text formats across all of 9Data
- [x] `#table/#columntype/#columnname/#record` parser (MobRegen, NPCItemList, Script, World, etc.)
- [x] `#DEFINE/#ENDDEFINE` parser (ServerInfo, DefaultCharacterData configs)
- [x] `ShineTableDataProvider` with auto-detection (format: "shinetable" / "configtable")
- [x] Write support for both table and define formats
- [x] DI registration
- [x] **Full import: 1234 total tables** (218 SHN + ~1016 text tables) from all of 9Data
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

### P2.5 - Encoding & Data Fidelity
- [x] SHN string encoding: EUC-KR (code page 949) for Korean strings
- [x] JsonElement handling in SHN write path (ConvertToByte/UInt32/etc.)
- [x] ConvertFromSqlite: direct Int64/Double handling (no checked Convert overflow)
- [x] ConvertFromSqlite: string-to-numeric TryParse fallback for raw table data

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
- [x] `build` command for shine tables: JSON → .txt output (table + define formats)
- [x] Build preserves original directory structure from manifest paths

---

## Priority

> **Guiding principle:** Fully functional basics before cool features. A working pipeline
> (import → edit → build → deploy) that handles all data correctly is worth more than
> half-finished advanced features.

### P1: Text Table String Length Bug
> Configtable #DEFINE STRING columns hardcode length 256, silently truncating longer strings.
> Simple bug, real data fidelity issue — fix ASAP.
- [x] Remove hardcoded 256 limit for configtable STRING columns (changed to Length=0, variable-length)

### P2: Conflicting Table Handling (Client Data)
> 45 tables conflict between server and client (same name, different data). Currently ignored
> during import. This is critical — the program must handle all data correctly before adding
> new features. Import both versions, build the correct one per environment.
- [x] Multi-source import: `--client` option on import command
- [x] Source origin tracking: server / client / shared per table (in metadata)
- [x] Duplicate detection: identical tables marked "shared", mismatches reported as conflicts
- [x] Build command: `--client-output` option writes client/shared tables to separate directory
- [x] TableComparer for data-level duplicate detection
- [x] Client data explored: 203 SHN files in `Z:/Client/Fiesta Online/ressystem/`
- [x] Multi-source import verified: 1336 tables (1210 server, 84 client, 42 shared, 82 conflicts)
- [x] Handle conflicting tables via conflict column splitting (`conflictStrategy: "split"` on merge actions)
- [x] `edit-template` CLI command to set conflictStrategy on merge actions
- [x] Full round-trip verified: init-template → edit-template → import → build → server SHN byte-identical, client data-identical (SHN header metadata from server base is a known limitation)

### P3: Local Docker Test Server
> Stand up a real Fiesta server locally in Docker Compose to test Mimir's output end-to-end.
> Two Windows containers: SQL Server Express (7 game databases) + game server (all 11 processes).
> Server binaries volume-mounted from `Z:/Server/`, game data from Mimir build output.
> Validates that built data files actually work — "server boots and players can log in."
- [x] Docker Compose config (`deploy/docker-compose.yml`)
- [x] SQL Server container with database restore from .bak files (`Dockerfile.sql`, `setup-sql.ps1`)
- [x] Game server container with all 11 processes (`Dockerfile.server`, `start-server.ps1`)
- [x] Volume mounts for server binaries + Mimir build output (9Data)
- [x] ServerInfo.txt override for Docker (ODBC Driver 17, `sqlserver` hostname)
- [ ] Smoke test: server boots, accepts connections with Mimir-built data
- [ ] Integration: `mimir build` → `docker compose restart gameserver` → verify server health

### P4: CI/CD — Push-to-Deploy Pipeline
> The end goal: push a JSON change to a GitHub repo → CI validates + builds → server auto-restarts
> with new data → client patch is packed and ready to download. Builds on the k8s setup from P3 —
> the local cluster is our test target for the full CI/CD loop before going to production.
- [ ] Dockerfile for `mimir` CLI
- [ ] GitHub Actions workflow: validate + build on push
- [ ] Exit non-zero on validation failure
- [ ] Server deploy integration (auto-restart on new build via k8s rollout)
- [ ] Client patch packaging (build client output into distributable zip)

### P5: Client Patcher
> Simple standalone patcher app for players. Loads a patch index JSON from a web server,
> compares against the client's current version, downloads the zip for the current patch,
> and extracts to the client folder (overwriting existing files). This also gives us a real
> target to test client build packaging against — CI builds the client data, packs a patch
> zip, uploads it, and the patcher pulls it down.
- [ ] Patch index format: JSON manifest listing versions, zip URLs, checksums
- [ ] Patcher app: fetch index → compare version → download zip → extract over client dir
- [ ] Version tracking: local version file in client dir, compare against index
- [ ] Progress reporting (download %, extraction %)
- [ ] `mimir pack` CLI command: build client env → zip → generate/update patch index JSON
- [ ] Test loop: `mimir build --all` → `mimir pack` → patcher downloads + applies → client launches

### P6: Edit in External Editor
> Open a single table in an external SHN editor (e.g. Spark Editor) for quick visual edits.
> Mimir builds the SHN to a temp file, opens the system file dialog / launches the editor,
> waits for it to close, then re-imports just that one file back into the project. Useful for
> visual spot-checking or leveraging existing editor UIs for quick tweaks without SQL.
- [ ] `mimir open <project> <table>` CLI command
- [ ] Build single table to temp SHN file
- [ ] Launch with system default app (or configurable editor path)
- [ ] Wait for process exit, then re-import the modified SHN back into the project
- [ ] Diff detection: only update JSON if the file actually changed
- [ ] Optional `--editor <path>` flag to specify editor binary

### P7: Drop Table Consolidation
> Text tables (spawn groups, NPC item lists, drop tables, etc.) are split into hundreds of separate
> tables by source file, making SQL queries across them nearly impossible. Need a merge rule
> (template action or import option) that consolidates all tables of the same schema into a single
> table, with an extra column for the source key (original filename / mob name).
> E.g. all `*_MobRegen` tables → one `MobRegen` table with a `SourceKey` column.
> This makes `SELECT * FROM MobRegen WHERE MobIndex = 123` actually work.
> This is also a merge rule problem like P2 — tackling them together makes sense.
- [ ] Design merge-by-schema template action (or auto-detect same-schema tables from same format)
- [ ] Add source key column during consolidation
- [ ] Ensure build can split back out to individual files
- [ ] Test with MobRegen (spawns), NPCItemList, and drop table families

### P8: QuestData Field Mapping
> Map more of the 666-byte fixed data region beyond QuestID. Generate a binary dump for
> collaborative hand-analysis against Spark Editor / known quest data. Expand FixedData
> into proper named columns as offsets are identified.
- [ ] Generate annotated hex dump of QuestData fixed region for hand-analysis
- [ ] Cross-reference with Spark Editor field definitions
- [ ] Incrementally extract known fields into proper columns

### Nice-to-Have (Do When Convenient)

#### Fixed Test Fixtures for Integration Tests
> Integration tests currently generate SHN/TXT files programmatically via providers. Instead,
> use fixed committed test files as fixtures. High prio for reliability but not blocking anything.
- [ ] Create `tests/fixtures/` directory with committed SHN and TXT test files
- [ ] Migrate SyntheticRoundtripTests to use fixed fixture files instead of WriteShn/WriteTxt
- [ ] Verify byte-identical roundtrip against known-good fixture files

### Completed
- [x] QuestData.shn support (QuestDataProvider — custom binary parser, no XOR, 666-byte fixed region + 3 PineScript strings)
- [x] README update

---

## Shelved (Revisit Later)

### SHN Type Refinements
- [ ] Deep analysis: check signedness of types 20/21/22 (signed vs unsigned)
- [ ] Deep analysis: string empty patterns (dash=key vs empty=text)

### File Exclusions
- [ ] "gitignore"-style exclusion patterns in definitions file
- [ ] Exclude files that look like tables but shouldn't be editable (e.g. ServerInfo.txt)

---

## Planned

### Definitions per Project Type
> Definitions file (`mimir.definitions.json`) is now searched upward from the project directory
> (like `.gitignore`). Currently lives at the repo root, outside the project folder.
> In the future, definitions should be per **project type** and shipped with the application.
> e.g. "Fiesta Online" project type includes all table key/column metadata out of the box.
> The definitions file can also track the SHN encryption key if different from default.

### Column-based Table Type Matching
> Our glob pattern matching for definitions could also match by column signature.
> e.g. if columns = ["RegenIndex", "MobIndex", "MobNum", ...] → type is MobSpawnTable.
> Then Siren_MobRegen matches that type and gets MobSpawnTable constraints applied automatically.
> This would make it easier to define constraints for families of tables with the same structure.

### Table Relationships & Grouping
> Make it more obvious when tables belong together. Example: Siren.txt has MobRegenGroup + MobRegen.
> MobRegenGroup defines groups with a string key "GroupIndex" (IsFamily decides if pulling one pulls all).
> MobRegen uses RegenIndex (FK to MobRegenGroup.GroupIndex) to assign mobs to each group.
> Note: MobRegen has no PK and no ID - row number is the implicit ID.
> No unique constraint on FK either - multiple MobRegen records per MobRegenGroup is valid.

### Constants / Enums in Shell
- [ ] Support named constants in shell mode: `#Classes.Gladiator` → `0x80` (or whatever the value)
- [ ] Load enum definitions from definitions file
- [ ] Replace constant references in SQL before execution

### Shell UX Improvements
- [ ] Tab-completion for table names and column names
- [ ] Column-aligned output (pad columns to fixed width, trim flexible-length columns at 32 chars)
- [ ] When only 2-3 columns in SELECT, print full values without trimming

### Metadata Build-out
- [ ] Go through all SHN/txt files and build column annotations for every table
- [ ] Document what each column means across all game tables
- [ ] Extract set definitions from drop tables (staggered 8, staggered 10, same-level types)
  - Include level, rarity, name "colour", dungeon/instance/enemy association

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

### Client Data & Asset Pipeline
> **Table data (SHN/txt)** is now handled by multi-source import. The import command
> accepts `--client` to import both server and client data, auto-detecting shared tables.
> Build writes to separate server/client output directories based on source origin.
>
> **Binary resources (~10GB textures, nifs, gcf)** still need the hybrid approach:
> Track an external `clientRoot` path plus local `assetOverrides/` folder.
> On build, copy from clientRoot then overlay assetOverrides.

- [x] Import client-side table data (multi-source import with `--client`)
- [x] Track sources in project manifest
- [x] Build to separate server/client output directories
- [ ] Track `clientRoot` path for binary assets
- [ ] Asset override folder for modified textures/nifs
- [ ] On build, copy from clientRoot + overlay overrides
- [ ] "Edit overrides" - if cloned set has a matching gcf/psd, use that instead
- [ ] Possible automated recoloring (find layer, dye, export to png on build)

### Quest System
- ~~Reverse engineer quest file format~~ (done — QuestDataProvider)
- ~~Quest reader/writer integrated into Mimir~~ (done)
- Cross-reference validation (quest rewards vs item DB, quest mobs vs mob DB)
- Map more fixed-data field offsets beyond QuestID (expand FixedData into proper columns)

### Scenario Scripting
> **Note:** QuestData.shn has PineScripts inlined directly into it (StartScript, InProgressScript,
> FinishScript columns). When tackling script editing, quest scripts live here — not in separate
> script files. The QuestDataProvider already extracts them as string columns, so they're
> queryable/editable via SQL today, but a proper script editor would need to parse PineScript syntax.
- Document the custom scripting language used for instance scripts
- Build a parser / AST for the scenario language
- Compiler / syntax checker
- Testing tools (dry-run a scenario script against game data)

### Map Editor
- [ ] Edit collision "block info" (walkable/not-walkable grid, ~256x256 8-bit bools)
- [ ] Visual editor for block info maps

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


# TO SORT:
* On true conflicts, maybe split the tables into e.g. ItemInfo and ItemInfo_Client. This is so that we can ensure that a project can always load, even in a very broken state (in this case, probably disallow/error on build?)
	This way, people can supply scripts or use cli shell mode to use SQL queries for example to fix up conflicts.
