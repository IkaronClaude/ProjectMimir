# Mimir Project Tracker

## Current Phase: Core Features Complete

SHN + raw table import, SQL query/edit/shell, constraint validation, and SHN build all working.
1234 tables import successfully. Building out definitions system and tooling.

---

## Architecture

```
\[Server Files] --import--> \[JSON Project] --edit/query--> \[JSON Project] --build--> \[Server Files]
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
        ItemInfo.json         # { header, columns, data: \[{...}, ...] }
        MobInfo.json
        Shine/View/
          ItemViewInfo.json
    shinetable/
      Shine/MobRegen/
        AdlVal01\_MobRegenGroup.json
      Shine/NPCItemList/
        AdlAertsina\_Tab00.json
      Shine/Script/
        AdlF\_Script.json
      Shine/World/
        ChrCommon\_Common.json
    configtable/
      ServerInfo/
        ServerInfo\_SERVER\_INFO.json
      Shine/
        DefaultCharacterData\_CHARACTER.json
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
  "constraints": \[
    {
      "description": "NPC shop items reference ItemInfo",
      "match": { "path": "data/shinetable/Shine/NPCItemList/\*\*", "column": "Column\*" },
      "foreignKey": { "table": "ItemInfo" },
      "emptyValues": \["-", ""]
    }
  ]
}
```

* **Table definitions**: `idColumn` (INTEGER PRIMARY KEY), `keyColumn` (UNIQUE index, nullable),
  `columnAnnotations` (displayName + description for each column)
* **Constraints**: Glob patterns on path/table/column, FK targets with shorthand
  (omit column → keyColumn, `@id` → idColumn), `{file}` template
* **FK enforcement**: Constraints become real SQLite FOREIGN KEY clauses at load time,
  with topological table ordering and empty value → NULL mapping

---

## Completed

### P0 - Foundation

* \[x] Solution structure, Directory.Build.props, all csproj files
* \[x] `Mimir.Core` - Models (ColumnType, ColumnDefinition, TableSchema, TableFile, TableEntry), IDataProvider, IProjectService, DI
* \[x] `Mimir.Shn` - ShnCrypto (XOR cipher), ShnDataProvider (read/write), DI
* \[x] `Mimir.Sql` - SqlEngine (SQLite in-memory, load/query/extract), DI
* \[x] `Mimir.Cli` - import/build/query/dump/analyze-types/validate/edit/shell commands
* \[x] Switched from JSONL to single .json per table (header+columns+data)
* \[x] Fixed type 10 (0x0A) = 4-byte padded string (map keys like Rou, Eld)
* \[x] Fixed type 29 (0x1D) = uint64 flags (targets, activities)
* \[x] Fixed type 26 reading: proper null-terminated string (not hacky rowLength calc)
* \[x] Updated IDataProvider to return `IReadOnlyList<TableEntry>` (multi-table files)
* \[x] **Full import: 219/219 SHN tables** (QuestData.shn handled by QuestDataProvider)

### P1 - Shine Tables (text formats)

* \[x] Identified 2 text formats across all of 9Data
* \[x] `#table/#columntype/#columnname/#record` parser (MobRegen, NPCItemList, Script, World, etc.)
* \[x] `#DEFINE/#ENDDEFINE` parser (ServerInfo, DefaultCharacterData configs)
* \[x] `ShineTableDataProvider` with auto-detection (format: "shinetable" / "configtable")
* \[x] Write support for both table and define formats
* \[x] DI registration
* \[x] **Full import: 1234 total tables** (218 SHN + ~1016 text tables) from all of 9Data
* \[x] Directory structure preserved in project layout

### P2 - Definitions \& Constraints

* \[x] `ProjectDefinitions` model (table keys, column annotations, constraint rules)
* \[x] `DefinitionResolver` (glob matching, `{file}` template, pattern resolution, key column resolution)
* \[x] `validate` CLI command (resolves constraints, checks via SQL, reports violations)
* \[x] Constraint-backed SQLite: FOREIGN KEY clauses in CREATE TABLE
* \[x] Topological sort for table load order (referenced tables first)
* \[x] Empty value → NULL mapping (so FK checks skip "-" and "" values)
* \[x] Deferred index creation after bulk load (graceful fallback on duplicate data)
* \[x] `idColumn` / `keyColumn` designation per table
* \[x] Column annotations with `displayName` and `description`
* \[x] FK shorthand: omit column → uses keyColumn, `@id` → uses idColumn

### P2.5 - Encoding \& Data Fidelity

* \[x] SHN string encoding: EUC-KR (code page 949) for Korean strings
* \[x] JsonElement handling in SHN write path (ConvertToByte/UInt32/etc.)
* \[x] ConvertFromSqlite: direct Int64/Double handling (no checked Convert overflow)
* \[x] ConvertFromSqlite: string-to-numeric TryParse fallback for raw table data

### P3 - Edit \& Shell

* \[x] `edit` command: single-shot SQL modification with automatic save back to JSON
* \[x] `shell` command: interactive SQL REPL (sqlcmd-style)

  * `.tables` - list all loaded tables
  * `.schema TABLE` - show column definitions with annotations
  * `.save` - save all tables back to JSON
  * `.quit` - quit with dirty-check prompt
  * SELECT/PRAGMA/EXPLAIN → query display; other SQL → execute with row count
  * Dirty tracking (prompts to save on quit if changes made)

### Build

* \[x] `build` command for SHN: JSON → SHN binary output
* \[x] `build` command for shine tables: JSON → .txt output (table + define formats)
* \[x] Build preserves original directory structure from manifest paths

---

## Priority

> \*\*Guiding principle:\*\* Fully functional basics before cool features. A working pipeline
> (import → edit → build → deploy) that handles all data correctly is worth more than
> half-finished advanced features.

### 🔥 P0: SHN File Inspection CLI (`mimir shn`)

> Needed immediately as a diagnostic tool — without it, investigating the row order blocker
> and other SHN fidelity issues requires manual hex work. Build this first.

See backlog item "SHN file inspection CLI commands" for full spec.

### 🔥 P0b: ItemInfo/ItemInfoServer Row Order Mismatch (Zone Blocker)

> Zone.exe is currently non-functional. Fix as soon as `mimir shn --diff` exists to
> confirm the root cause. See open issue "ItemInfo/ItemInfoServer row order mismatch".

### 🔥 P0d: ItemDataBox load order issue (after fieldInfo)

Confirmed sequence of blockers so far:
1. ItemInfo/ItemInfoServer row order → fix in P0b
2. fieldInfo (ShineTable) loading → **confirmed fixed** by copying `World/Field.txt` from `Z:/Server` directly (workaround). Underlying ShineTable output issues still tracked in P0c.
3. **Next**: ItemDataBox load order error — another load ordering issue in Zone startup after fieldInfo loads. Needs investigation once P0b is deployed.

### 🔥 P0c: ShineTable Output Issues (Next Zone Blocker After ItemInfo)

> Once ItemInfo row order is fixed, Zone.exe will likely fail on ShineTable (.txt) loading
> (fieldInfo and similar). Three known issues:

**1. Lowercase directives** — Mimir writes `#record`, `#columntype`, `#columnname`, `#table` etc. in lowercase. Original server files use uppercase (`#Record`, `#ColumnType`, `#ColumnName`, `#Table`). Unknown whether the game parser is case-sensitive, but safest to match original casing exactly.

**2. `#Exchange` / `#Delimiter` not supported** — Some files use `#Delimiter \x20` (space as field delimiter) combined with `#Exchange # \x20` (swap `#` and space so `#` can appear in data). Mimir currently ignores these directives entirely, which will corrupt any file that uses them on write.

**3. `#Ignore \o042` not re-emitted** — `\o042` is `"` (double-quote). The `#Ignore` directive tells the parser to treat that character as invisible/escaped. Mimir reads and applies it during import (via Preprocessor) but never re-emits it on write, so rebuilt files may fail to parse if any data values contain double-quotes.

**To investigate:** Grep `Z:/Server` for `#Exchange`, `#Delimiter`, `#Ignore` to see which files use them and whether any affected data values actually contain the characters in question. Fix lowercase directives unconditionally; fix Exchange/Ignore only if grep confirms real-world usage.

### P1: Text Table String Length Bug

> Configtable #DEFINE STRING columns hardcode length 256, silently truncating longer strings.
> Simple bug, real data fidelity issue — fix ASAP.

* \[x] Remove hardcoded 256 limit for configtable STRING columns (changed to Length=0, variable-length)

### P2: Conflicting Table Handling (Client Data)

> 45 tables conflict between server and client (same name, different data). Currently ignored
> during import. This is critical — the program must handle all data correctly before adding
> new features. Import both versions, build the correct one per environment.

* \[x] Multi-source import: `--client` option on import command
* \[x] Source origin tracking: server / client / shared per table (in metadata)
* \[x] Duplicate detection: identical tables marked "shared", mismatches reported as conflicts
* \[x] Build command: `--client-output` option writes client/shared tables to separate directory
* \[x] TableComparer for data-level duplicate detection
* \[x] Client data explored: 203 SHN files in `Z:/Client/Fiesta Online/ressystem/`
* \[x] Multi-source import verified: 1336 tables (1210 server, 84 client, 42 shared, 82 conflicts)
* \[x] Handle conflicting tables via conflict column splitting (`conflictStrategy: "split"` on merge actions)
* \[x] `edit-template` CLI command to set conflictStrategy on merge actions
* \[x] Full round-trip verified: init-template → edit-template → import → build → server SHN byte-identical, client data-identical (SHN header metadata from server base is a known limitation)

### P3: Local Docker Test Server

> Stand up a real Fiesta server locally in Docker Compose to test Mimir's output end-to-end.
> Two Windows containers: SQL Server Express (7 game databases) + game server (all 11 processes).
> Server binaries volume-mounted from `Z:/Server/`, game data from Mimir build output.
> Validates that built data files actually work — "server boots and players can log in."

* \[x] Docker Compose config (`deploy/docker-compose.yml`)
* \[x] SQL Server container with database restore from .bak files (`Dockerfile.sql`, `setup-sql.ps1`)
* \[x] Game server container with all 11 processes (`Dockerfile.server`, `start-server.ps1`)
* \[x] Volume mounts for server binaries + Mimir build output (9Data)
* \[x] ServerInfo.txt override for Docker (ODBC Driver 17, `sqlserver` hostname)
* \[ ] Smoke test: server boots, accepts connections with Mimir-built data
* \[ ] Integration: `mimir build` → `docker compose restart gameserver` → verify server health
* Note: BuildKit must be disabled for Windows containers (`set DOCKER\_BUILDKIT=0`)

### P4: Standalone Tool \& Self-Contained Projects

> \*\*Goal:\*\* Install mimir to PATH, then `mimir init "Project1"` creates a fully self-contained
> project directory with everything needed: mimir.json, deploy/ (Dockerfiles, compose, server
> executables copied in), data/, build/, overrides/. No symlinks — direct file copies for server
> binaries. Then `cd Project1 \&\& mimir build` builds both client and server, and Docker commands
> just work.
>
> This is an architectural shift: mimir becomes a global tool, projects are standalone directories.
> Current layout (mimir source repo contains test-project/) goes away.

* \[ ] `mimir init <project>` — prompts for server/client paths, creates project structure
* \[ ] Copy server executables (Account/, Login/, Zone\*/, etc.) into project deploy/ dir
* \[ ] Copy database .bak files into project
* \[ ] Auto-generate Dockerfiles, docker-compose.yml, ServerInfo.txt inside project
* \[ ] `mimir build` (no args) — detects project from cwd, builds all envs
* \[ ] Publish as dotnet global tool or standalone exe for PATH install
* \[ ] Remove setup.ps1 (replaced by `mimir init`)

### P4b: CI/CD — Push-to-Deploy Pipeline

> The end goal: push a JSON change to a GitHub repo → CI validates + builds → server auto-restarts
> with new data → client patch is packed and ready to download. Builds on the k8s setup from P3 —
> the local cluster is our test target for the full CI/CD loop before going to production.

* \[ ] Dockerfile for `mimir` CLI
* \[ ] GitHub Actions workflow: validate + build on push
* \[ ] Exit non-zero on validation failure
* \[ ] Server deploy integration (auto-restart on new build via k8s rollout)
* \[ ] Client patch packaging (build client output into distributable zip)

### P5: Client Patcher

> Simple standalone patcher app for players. Loads a patch index JSON from a web server,
> compares against the client's current version, downloads the zip for the current patch,
> and extracts to the client folder (overwriting existing files). This also gives us a real
> target to test client build packaging against — CI builds the client data, packs a patch
> zip, uploads it, and the patcher pulls it down.

* \[x] Patch index format: JSON manifest listing versions, zip URLs, checksums
* \[x] Patcher app: fetch index → compare version → download zip → extract over client dir
* \[x] Version tracking: local version file in client dir, compare against index
* \[x] `mimir pack` CLI command: build client env → diff against manifest → zip changed files → update patch index
* \[x] Patcher script: `deploy/patcher/patch.bat` + `patch.ps1` (PowerShell, SHA-256 verification)
* \[x] 9 integration tests covering full pack lifecycle (first pack, incremental, no-change, base-url)
* \[ ] Test loop: `mimir build --all` → `mimir pack` → patcher downloads + applies → client launches
* \[ ] Progress reporting (download %, extraction %)
* \[x] **Fix packer base state** — `mimir import` now seeds `.mimir-pack-manifest.json` (v0) by hashing all source files from each env's `importPath` at the end of import. Pack manifest root is auto-derived: if `importPath` parent is not a drive root (e.g. `Z:/ClientSource/ressystem` → root `Z:/ClientSource`), relative paths match build output (`ressystem/ItemInfo.shn`); if parent IS a drive root (e.g. `Z:/Server`) importPath itself is used. `mimir pack` then diffs against this baseline so patch v1 only contains actual changes. Use `--retain-pack-baseline` on import/reimport to skip the reseed. Also added `deploy/update.bat` (quick hot-swap: build → pack → robocopy → restart game servers, no SQL/Docker rebuild).

### P6: Edit in External Editor

> Open a single table in an external SHN editor (e.g. Spark Editor) for quick visual edits.
> Mimir builds the SHN to a temp file, opens the system file dialog / launches the editor,
> waits for it to close, then re-imports just that one file back into the project. Useful for
> visual spot-checking or leveraging existing editor UIs for quick tweaks without SQL.

* \[ ] `mimir open <project> <table>` CLI command
* \[ ] Build single table to temp SHN file
* \[ ] Launch with system default app (or configurable editor path)
* \[ ] Wait for process exit, then re-import the modified SHN back into the project
* \[ ] Diff detection: only update JSON if the file actually changed
* \[ ] Optional `--editor <path>` flag to specify editor binary

### P7: Drop Table Consolidation

> Text tables (spawn groups, NPC item lists, drop tables, etc.) are split into hundreds of separate
> tables by source file, making SQL queries across them nearly impossible. Need a merge rule
> (template action or import option) that consolidates all tables of the same schema into a single
> table, with an extra column for the source key (original filename / mob name).
> E.g. all `\*\_MobRegen` tables → one `MobRegen` table with a `SourceKey` column.
> This makes `SELECT \* FROM MobRegen WHERE MobIndex = 123` actually work.
> This is also a merge rule problem like P2 — tackling them together makes sense.

* \[ ] Design merge-by-schema template action (or auto-detect same-schema tables from same format)
* \[ ] Add source key column during consolidation
* \[ ] Ensure build can split back out to individual files
* \[ ] Test with MobRegen (spawns), NPCItemList, and drop table families

### P8: QuestData Field Mapping

> Map more of the 666-byte fixed data region beyond QuestID. Generate a binary dump for
> collaborative hand-analysis against Spark Editor / known quest data. Expand FixedData
> into proper named columns as offsets are identified.

* \[ ] Generate annotated hex dump of QuestData fixed region for hand-analysis
* \[ ] Cross-reference with Spark Editor field definitions
* \[ ] Incrementally extract known fields into proper columns

### Nice-to-Have (Do When Convenient)

#### Fixed Test Fixtures for Integration Tests

> Integration tests currently generate SHN/TXT files programmatically via providers. Instead,
> use fixed committed test files as fixtures. High prio for reliability but not blocking anything.

* \[ ] Create `tests/fixtures/` directory with committed SHN and TXT test files
* \[ ] Migrate SyntheticRoundtripTests to use fixed fixture files instead of WriteShn/WriteTxt
* \[ ] Verify byte-identical roundtrip against known-good fixture files

### Completed

* \[x] QuestData.shn support (QuestDataProvider — custom binary parser, no XOR, 666-byte fixed region + 3 PineScript strings)
* \[x] README update

---

### SHN file inspection CLI commands

Quick `mimir shn` subcommands for inspecting raw SHN files without importing them into a project — useful for debugging, diffing, and verifying build output:
- `mimir shn <file> --schema` — print column names, types, lengths
- `mimir shn <file> --row-count` — print number of rows
- `mimir shn <file> --head <N>` / `--tail <N>` — print first/last N rows as a table
- `mimir shn <file> --skip <N> --take <M>` — slice rows
- `mimir shn <file1> <file2> --diff` — compare two SHN files (schema + row-by-row)
- `mimir shn <file> --decrypt-to <outfile>` — write decrypted raw bytes for hex analysis

Especially useful for diagnosing row order mismatches, schema differences, and roundtrip fidelity issues without needing a full project import.

## Sub-Projects / Future Projects

### Game Management API (separate project)

A Docker container exposing an HTTP API over the game databases — account creation, character queries, GM tools, server status, etc. Would form the backend for a web panel or admin UI. Likely a separate repo/project rather than part of Mimir itself, but would depend on the same Docker Compose network and SQL Server setup. Worth building once the server deployment is stable and the database schema is well understood.

---

## Shelved (Revisit Later)

### SHN Type Refinements

* \[ ] Deep analysis: check signedness of types 20/21/22 (signed vs unsigned)
* \[ ] Deep analysis: string empty patterns (dash=key vs empty=text)

### File Exclusions

* \[ ] "gitignore"-style exclusion patterns in definitions file
* \[ ] Exclude files that look like tables but shouldn't be editable (e.g. ServerInfo.txt)

---

## Planned

### Definitions per Project Type

> Definitions file (`mimir.definitions.json`) is now searched upward from the project directory
> (like `.gitignore`). Currently lives at the repo root, outside the project folder.
> In the future, definitions should be per \*\*project type\*\* and shipped with the application.
> e.g. "Fiesta Online" project type includes all table key/column metadata out of the box.
> The definitions file can also track the SHN encryption key if different from default.

### Column-based Table Type Matching

> Our glob pattern matching for definitions could also match by column signature.
> e.g. if columns = \["RegenIndex", "MobIndex", "MobNum", ...] → type is MobSpawnTable.
> Then Siren\_MobRegen matches that type and gets MobSpawnTable constraints applied automatically.
> This would make it easier to define constraints for families of tables with the same structure.

### Table Relationships \& Grouping

> Make it more obvious when tables belong together. Example: Siren.txt has MobRegenGroup + MobRegen.
> MobRegenGroup defines groups with a string key "GroupIndex" (IsFamily decides if pulling one pulls all).
> MobRegen uses RegenIndex (FK to MobRegenGroup.GroupIndex) to assign mobs to each group.
> Note: MobRegen has no PK and no ID - row number is the implicit ID.
> No unique constraint on FK either - multiple MobRegen records per MobRegenGroup is valid.

### Constants / Enums in Shell

* \[ ] Support named constants in shell mode: `#Classes.Gladiator` → `0x80` (or whatever the value)
* \[ ] Load enum definitions from definitions file
* \[ ] Replace constant references in SQL before execution

### Shell UX Improvements

* \[ ] Tab-completion for table names and column names
* \[ ] Column-aligned output (pad columns to fixed width, trim flexible-length columns at 32 chars)
* \[ ] When only 2-3 columns in SELECT, print full values without trimming

### Metadata Build-out

* \[ ] Go through all SHN/txt files and build column annotations for every table
* \[ ] Document what each column means across all game tables
* \[ ] Extract set definitions from drop tables (staggered 8, staggered 10, same-level types)

  * Include level, rarity, name "colour", dungeon/instance/enemy association

---

## SHN Type Codes

|Type|Hex|Size|Count|Meaning|
|-|-|-|-|-|
|1|0x01|1|310|byte (bools, small enums)|
|2|0x02|2|525|uint16 (IDs, rates)|
|3|0x03|4|319|uint32 (IDs, indices)|
|5|0x05|4|3|float (scales)|
|9|0x09|var|364|padded string (null-term, zero-padded to length)|
|10|0x0A|4|1|padded string (always 4-byte map keys: Rou, Eld, Urg)|
|11|0x0B|4|166|uint32 flags (0 = none)|
|12|0x0C|1|8|byte (mode/type)|
|13|0x0D|2|7|uint16 (type IDs)|
|16|0x10|1|1|byte flags|
|20|0x14|1|17|sbyte (likely signed, TBD)|
|21|0x15|2|144|int16 (likely upgrade indices, 0=none)|
|22|0x16|4|65|int32 (likely time values)|
|24|0x18|32+|19|padded string (string keys)|
|26|0x1A|var|3|null-terminated variable-length string|
|29|0x1D|8|5|uint64 flags (targets, activities)|

SHN indices are 1-based. Types 1-4 are unsigned, types 20-22 are likely signed counterparts.
String columns: `-` when empty = key/index, `""` when empty = free text.

---

## Stretch Goals

### Hot-swap Build (Build While Server Running) ✓ DONE

Windows Docker bind-mounts lock the mounted directory even `:ro`, so `mimir build --all` can't write to `build/server/` while containers are up.

**Implemented:** `build/server/` → `test-project/deployed/server/` robocopy snapshot on every deploy/restart. Containers mount `deployed/`, leaving `build/` always free. `mimir build` can run at any time without stopping the server — just run `restart-game.bat` after to push the new data.



### Data Validation / Linting

Built-in rules that detect common mistakes and inconsistencies:

* **Orphaned items** - Items in ItemInfo not in any drop table AND not sold by any NPC
* **Incomplete sets** - Sets missing a piece for a class, or items breaking set naming convention
* **Broken references** - Drop tables referencing nonexistent item IDs, quests rewarding invalid items
* **Stat anomalies** - Items with stats wildly out of range for their level bracket
* **Missing localizations** - Items/quests/NPCs with empty or placeholder name strings
* **Duplicate entries** - Same item ID appearing multiple times, duplicate drop table entries
* **Unreachable content** - Maps with no warp points, mobs that don't spawn anywhere

### Scripted Workflows

Composable CLI commands for common multi-step operations:

* Clone map
* Clone mob type (including skills, spawn groups, etc.)
* Clone armor set (duplicate items, adjust names/stats, add to drop tables, copy referenced assets into `overrides/client/`)
* Calculate stat formulas from given data, apply with multiplier to new sets
* Quick drop group creation \& assignment (e.g. apply to all mobs with level > 120)
* Quick enhancement stat adjustment, enhancement groups
* Quick NPC shop editing (e.g. "add armor set to NPC X")
* Multi-layer sets (e.g. "SetGroup XYZ, Rarity Orange, Set Bonus, Level 115, Formula +10 offset")
* Quick mob cloning for dungeons/instances/KQs (like Nest Boogie for Leviathan's Nest)
* Time limit editing (e.g. remove time limit from skins)
* Visual monster spawn group editor

### Client Data \& Asset Pipeline

> \*\*Table data (SHN/txt)\*\* is now handled by multi-source import. The import command
> accepts `--client` to import both server and client data, auto-detecting shared tables.
> Build writes to separate server/client output directories based on source origin.
>
> \*\*Binary resources (~10GB textures, nifs, gcf)\*\* still need the hybrid approach:
> Track an external `clientRoot` path plus local `assetOverrides/` folder.
> On build, copy from clientRoot then overlay assetOverrides.

* \[x] Import client-side table data (multi-source import with `--client`)
* \[x] Track sources in project manifest
* \[x] Build to separate server/client output directories
* \[ ] Track `clientRoot` path for binary assets
* \[ ] Asset override folder for modified textures/nifs
* \[ ] On build, copy from clientRoot + overlay overrides
* \[ ] "Edit overrides" - if cloned set has a matching gcf/psd, use that instead
* \[ ] Possible automated recoloring (find layer, dye, export to png on build)

### Quest System

* ~~Reverse engineer quest file format~~ (done — QuestDataProvider)
* ~~Quest reader/writer integrated into Mimir~~ (done)
* Cross-reference validation (quest rewards vs item DB, quest mobs vs mob DB)
* Map more fixed-data field offsets beyond QuestID (expand FixedData into proper columns)

### Scenario Scripting

> \*\*Note:\*\* QuestData.shn has PineScripts inlined directly into it (StartScript, InProgressScript,
> FinishScript columns). When tackling script editing, quest scripts live here — not in separate
> script files. The QuestDataProvider already extracts them as string columns, so they're
> queryable/editable via SQL today, but a proper script editor would need to parse PineScript syntax.

* Document the custom scripting language used for instance scripts
* Build a parser / AST for the scenario language
* Compiler / syntax checker
* Testing tools (dry-run a scenario script against game data)

### Map Editor

* \[ ] Edit collision "block info" (walkable/not-walkable grid, ~256x256 8-bit bools)
* \[ ] Visual editor for block info maps

### IDE Tooling (Far Future)

* VSCode extension with language support for scenario scripts
* Intellisense / autocomplete (item IDs, mob IDs, map names from loaded data)
* Hover info showing resolved references (hover item ID → show item name/stats)
* Linting integration (red squiggles for broken references)

---

## Backlog

### Server build path should target 9Data directly

Currently the server env `buildPath` is set to `build/server/`, so build output lands at `build/server/9Data/Shine/ItemInfo.shn`. The `buildPath` should be the 9Data dir itself — `build/server/9Data` — so that files land flat in the right place without the extra `9Data` prefix in the path. Requires updating default in `mimir env server init` and adjusting any snapshot/robocopy commands that reference the old layout.

Related: a separate **deploy path** is needed for server-side non-data files (exes, DBs, scripts, GamigoZR, etc.) that live one directory above 9Data. The deploy path env config would let `mimir deploy` (or `update.bat`) copy binaries + config files from the deploy path alongside the built 9Data snapshot. This cleanly separates "data Mimir owns" from "binaries Mimir doesn't touch".

### Auto-archive old log files on container restart

On container restart, existing game server log files (Assert logs, ExitLogs, Msg logs, etc.) should be moved into an `old/` or timestamped subfolder rather than being appended to or overwritten. Makes it much easier to isolate logs from the current run vs. previous runs.

### Docker containers should exit when their game process exits

Currently containers stay alive even after the game server process (Login.exe, Zone.exe, etc.) shuts down or crashes. Once the deployment setup is stable, containers should be configured to exit when their main process does — so `docker ps` reflects actual server health and `docker compose up` restarts crashed processes correctly. Likely just a matter of ensuring the entrypoint doesn't swallow exit codes and that `restart: on-failure` or similar policy is set in `docker-compose.yml`.

### Deploy scripts should be callable from inside the project folder

Currently `deploy.bat`, `update.bat`, etc. live in `deploy/` and must be run from there (they use relative paths). They should be callable from the project root via a `mimir-deploy.bat` shim (or similar) that forwards to the real scripts with correct working directory context.

Similarly, `mimir init` should scaffold a `mimir.bat` in the new project directory that forwards all commands to the mimir executable used to run `init` — so `mimir.bat build`, `mimir.bat import`, etc. work from the project root without needing to know the install path. The init command already writes a basic `mimir.bat`, but it should capture the actual invocation path (e.g. `dotnet run --project ...` or the installed exe path) rather than assuming `mimir` is in PATH.

### deploy.bat wipes SQL database unconditionally

`deploy.bat` currently calls `rebuild-sql.bat` which wipes and restores all game databases from `.bak` files, destroying any runtime state (character data, account data, etc.). This is only appropriate for a first-time setup or an intentional reset — not for a routine full deploy. `deploy.bat` should check whether the SQL container/databases already exist and skip the SQL rebuild if so, or split into separate `deploy-first-time.bat` vs `deploy.bat` scripts with clearly different semantics.

### SQL password management

No good way to set/change the SQL Server password used by the game server and Docker setup. Currently hardcoded in deploy scripts and `ServerInfo.txt`. Need a proper mechanism — e.g. `mimir env server set sql-password <pw>` or a dedicated secrets file — so the password can be configured per-project without editing raw config files.

### Split `mimir` CLI into focused sub-tools

As the CLI grows, consider splitting into separate executables by domain:
- `mimir env` — environment management
- `mimir sql` — query/edit/shell
- `mimir build` / `mimir import` — data pipeline
- `mimir deploy` — Docker deployment lifecycle
- `mimir patch` — pack/patch index management

Each could be a standalone dotnet tool, composable via scripts. Keeps each tool small and focused, easier to document and discover. Low priority — only worth doing once the feature set is stable.

### Reimport to dummy dir for more accurate baseline seeding

Currently `mimir build` seeds the pack baseline from the actual build output directory, which means the baseline reflects Mimir's rebuilt files (not the original source files). A more accurate baseline — useful for measuring true diff vs. the stock client — could be produced by running a temporary import+build into a throwaway directory using the original source files, then seeding from that. This would let patch v1 contain only files the user actually changed vs. vanilla, even accounting for roundtrip fidelity. Low priority for now; the current build-output approach is pragmatic and produces correct incremental patches.

### `mimir pack` should auto-seed baseline if manifest missing

Currently if no pack manifest exists, `mimir pack` treats all files as new and produces a patch containing everything. Instead, `mimir pack` should automatically seed the baseline from the current build output (same logic as `mimir build` post-build seeding), then diff against it — producing a 0-file patch on the first run. This makes the workflow forgiving: even if the manifest was deleted or never created, the user can just run `mimir pack` and it self-heals without needing to re-run `mimir build`.

### Always create patch-index.json even when no changes to pack

If no pack manifest exists or there are no changed files, `mimir pack` currently exits early without writing `patch-index.json`. The index file should always be created/updated on every pack run — even a no-change run — so that clients can always find a valid index to query. An empty/current index with no new patch entry is a valid and useful state.

### Overrides must never be included in the pack baseline

`mimir build` currently seeds the pack baseline from the full build output directory, which includes any files copied from the overrides folder. This means override files are "known" to the baseline and only show up in a patch if they subsequently change. Instead, the baseline should only cover files produced by the core build (tables + copyFile actions) — override files should always be excluded from baseline seeding so that the first patch after a build always delivers them to players. After that first delivery they diff normally like everything else.

### patch-index.json version collision on baseline reset

If the pack baseline is reset (v0 reseeded) and then `mimir pack` is run again, the pack produces a new "version 1" zip — but `patch-index.json` may already have a version 1 entry from a previous pack run, resulting in duplicate version numbers in the index. Need to either:
- Clear `patch-index.json` when reseeding baseline (implicit: reset = start over)
- Or track the current version in the per-env manifest and advance from there regardless

### Move pack manifest into environments/ dir

`.mimir-pack-manifest-{envName}.json` in the project root is ugly and ad-hoc. Move to `environments/{envName}/pack-manifest.json` alongside the env config. This also makes `--reseed-baseline-only` a natural fit as `mimir env <name> reseed-baseline` (reseeds just that env's manifest), keeping all env operations under the `mimir env` command namespace.

---

## Open Issues

### Zone.exe needs write access to SubAbStateClass.txt ✓ RESOLVED

Zone.exe opens `9Data/SubAbStateClass.txt` with WriteAppend permissions at startup. Volume was mounted `:ro`, blocking the open.

**Fix applied:** Implemented `build/ → deployed/` snapshot copy as part of deploy workflow. Volume now mounts `deployed/server/9Data` read-write.
- `docker-compose.yml` — changed mount to `../test-project/deployed/server/9Data:C:/server/9Data` (no `:ro`)
- `deploy.bat` — robocopy `build/server → deployed/server` before `docker compose up`
- `restart-game.bat` — same robocopy before restart, so `mimir build` + `restart-game.bat` is the full update cycle
- `build/` remains free to rebuild while containers are running (hot-swap ready)

### Build output 9Data folder is polluted with non-data files

`build/server/9Data/` contains files that don't belong in the game data directory — e.g. `ServerInfo.txt`, `.exe` files, `.ps1` scripts, and potentially others. These are likely being included because Mimir's import scans the server directory and picks up everything it can parse, including config/tool files that happen to live alongside the real data.

**Impact:** The `deployed/server/9Data/` snapshot (and previously the read-only mount) exposes these files to the containers, which is at minimum confusing and could interfere with server startup if an exe or config file shadows something the game expects.

**To investigate:**
- List what non-SHN/non-txt files are in `build/server/9Data/`
- Trace where they come from in the import (which source directory, which provider picks them up)
- Decide on fix: exclusion patterns in the import scan, a gitignore-style filter in `mimir.json` or `mimir.template.json`, or a post-build cleanup step

### ShineTable output missing preprocessor directives

Mimir's `ShineTableFormatParser.Write` outputs bare `#table`/`#columntype`/`#columnname`/`#record` blocks with no preprocessor directives. Original files may have directives at the top like:

```
#ignore \o042
#exchange # \x20
```

These are parsed and applied during import (handled by `Preprocessor`) but never re-emitted on write. If the game exe's parser requires these directives to correctly handle quoted strings or special characters in the data, their absence could cause parse errors or data corruption.

**To investigate:**
- Check which original server files have `#ignore`/`#exchange` directives (grep `Z:/Server`)
- Determine if any data values in those files contain the affected characters (e.g. `\o042` = `"` double-quote)
- If character-mangling directives are present: either re-emit them on write, or ensure Mimir has already applied them to the stored values (so the raw data no longer needs them)

### ItemInfo/ItemInfoServer row order mismatch ⚠️ BLOCKING ZONE

Zone.exe currently fails to start with:
```
ItemDataBox::idb_Load : iteminfo iteminfoserver Order not match[3228]
```
`[3228]` is likely a **row number**, not a column index. The game engine requires ItemInfo.shn and ItemInfoServer.shn to have matching rows at the same row numbers — i.e. the row for item ID X must be at the same position in both files. Mimir correlates rows by key columns (correct relational approach), but the built files may reorder rows relative to the originals, breaking the engine's assumption.

**To verify**: Decrypt and compare built `build/server/9Data/Shine/ItemInfo.shn` and `ItemInfoServer.shn` row order against the originals in `Z:/Server/9Data/Shine/`. If row order differs, the fix is to preserve original row order during build (sort by original row index, not by key).

Fallback: user can supply a known-good server release for cross-reference if needed.

The game loads `ItemInfo.shn` and `ItemInfoServer.shn` and cross-validates their column order. `[180]` is the column index where they diverge. This suggests that after merge+split, the column ordering in one or both files differs from the originals.

**Likely cause:** `TableSplitter.Split` rebuilds columns from the merged schema, which may reorder or drop env-specific columns. Column 180 in ItemInfo/ItemInfoServer is where the split columns (server-only vs client-only) begin diverging from the merged schema.

**To investigate:**
- Compare column order of `Z:/Server/9Data/Shine/ItemInfo.shn` vs Mimir's `build/server/9Data/Shine/ItemInfo.shn` (column 180+)
- Same for `ItemInfoServer.shn`
- Check whether `TableSplitter` preserves original column order per env (from `EnvMergeMetadata.ColumnOrder`)
- Check whether the `conflictStrategy: "split"` columns are being output in the right order

**Note:** MobChat error in same log is from a prior session before the `#RecordIn` fix was deployed.

### Zone.exe crash — GamigoZR dependency ✓ RESOLVED

Zone.exe crashes at startup (`ShineObjectManager::som_Initialize` returns `0xFFFFFFFF`) without GamigoZR running. GamigoZR is a core Gamigo service that must be running before Zone.exe starts — once running, all Zone log files (Assert, ExitLog, Msg, etc.) begin populating normally.

**Fix applied:**
- `start-process.ps1` — registers and starts `GamigoZR` service before Zone.exe for all Zone processes
- `Dockerfile.server` — added `COPY server-files/GamigoZR/ C:/server/GamigoZR/`
- To deploy: `xcopy /E /I Z:\Server\GamigoZR deploy\server-files\GamigoZR` then `rebuild-game.bat`

Stack dump for reference: `20120409-Hero[Release]-1`, `ProtocolPacket::pp_SetPacketLen[4294967295]`, `ShineObjectManager::som_Initialize[4294967295]`

---

## Notes

* SHN format: 32-byte header + XOR cipher + binary column/row data (see FiestaLib source)
* Existing SHN Editor source at: `Z:/Odin Server Files/Fiesta Tool Project Source/SHN Editor/`
* Spark Editor reference: https://github.com/Wicious/Spark-Editor
* Server data at: `Z:/Odin Server Files/Server/9Data/` (~220 .shn + hundreds of .txt files)
* Server and client share some files (ItemInfo.shn shared, ItemViewInfo.shn client-only, MobRegen server-only)
* Quest files use a different format from SHN - TBD
* Scenario files use a custom language (not Lua) - TBD
* .NET 10 SDK available on build VM (10.0.102)
* GitHub: https://github.com/IkaronClaude/ProjectMimir



# TO SORT:

* On true conflicts, maybe split the tables into e.g. ItemInfo and ItemInfo\_Client. This is so that we can ensure that a project can always load, even in a very broken state (in this case, probably disallow/error on build?)
  This way, people can supply scripts or use cli shell mode to use SQL queries for example to fix up conflicts.
* Generate actual project rather than fixed "test-project"
* Add .gitignore that removes /build/ folder from a mimir project, ready for version control
* ~~Allow QuestData.shn dynamic record length per row~~ (done — fixedDataSize auto-detected per file via null-scanning from record end)
* Build a lovely readme with a step-by-step from git clone of mimir to built server files and patches
* Build a lovely readme with step-by-step from built files to running docker instances

~~Docker gets stuck waiting for sql to launch~~ (fixed — healthcheck was using `localhost:1433` but SQL Express named instance uses dynamic port; changed to `.\SQLEXPRESS` named pipe connection)