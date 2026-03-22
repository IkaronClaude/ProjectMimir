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

### ✅ P0: rebuild-game WIPES ALL DATABASES TO BACKUP STATE — FIXED

**Root cause:** When sqlserver container is recreated, SQL Server starts with an empty
`sys.databases` — the .mdf files are on the volume but not yet registered. The existence
check saw COUNT=0 and triggered `RESTORE ... WITH REPLACE`, overwriting live data.

**Two-layer fix:**
1. `rebuild-game.bat` — explicitly lists game services in `up -d`; sqlserver is never touched
2. `setup-sql.ps1` — when .mdf files exist but db is unregistered, ATTACH instead of restore.
   Only falls through to `RESTORE FROM DISK` when no data files exist at all (fresh volume).
   Removed `WITH REPLACE` from restore path entirely (restoring over existing data is never safe).

---

### ✅ P0: SHN File Inspection CLI (`mimir shn`) — DONE

Implemented. Commands:
- `mimir shn <file>` — schema (default)
- `mimir shn <file> --row-count`
- `mimir shn <file> --head <N>` / `--tail <N>` / `--skip <N> --take <M>`
- `mimir shn <file> --diff <file2>` — positional row diff with reorder heuristic
- `mimir shn <file> --decrypt-to <out>` — write decrypted bytes for hex analysis

### ✅ P0b: Server Full Roundtrip — DONE

Zone.exe starts cleanly on a fully Mimir-built server. All server-side blockers resolved:

- **ItemInfo.shn / ItemInfoServer.shn** — ✅ row order fixed (TableMerger single-pass over target.Data); ✅ client-only row env bug fixed (copy action pre-tags + targetEnvName fallback in Merge); ✅ duplicate join key bug fixed (sourceIndex now Queue-based FIFO — 17 Q_ items appearing twice in source no longer produce ghost server-only rows; built ItemInfo.shn now byte-identical to source)
- **ChargedEffect.shn** — ✅ data-identical to source; `Same Handle[1738]` is a pre-existing duplicate in the original server files that Zone tolerates as a warning
- **Field.txt** — ✅ fixed (EUC-KR encoding + INDEX vs STRING[N] round-trip)
- **ActionViewInfo.shn** — ✅ fixed in P0e; both `9Data/Shine/ActionViewInfo.shn` and `9Data/Shine/View/ActionViewInfo.shn` now import and build correctly after re-running `init-template` + `import`.

### ✅ P0f: Client "Illegally Manipulated" Hash Check Failure — DONE

Root cause was a DB/client version mismatch — the `.bak` files were from a different client version than `Z:/ClientSource/ressystem`. Using matching client source resolved the problem. Not a Mimir roundtrip fidelity issue.

### ✅ P0e: Same-Named SHN Files at Multiple Paths — FIXED (commit 3c2468c)

`ReadAllTables` now uses a two-pass approach: shallower copy keeps the original name (`ActionViewInfo`), deeper copy gets a prefix (`View.ActionViewInfo`). `TemplateGenerator` sets `outputName = "ActionViewInfo"` on the deeper copy's template action so the build produces the correct filename. Import uses the internal name for the JSON file path to prevent collisions. Both `Shine/ActionViewInfo.shn` and `Shine/View/ActionViewInfo.shn` now roundtrip correctly. Covered by `SyntheticMultiPathTests` (9 integration tests).

### ✅ P0d: ItemDataBox load order issue — DONE

### P0c → P3: ShineTable Output Directives (Non-blocking)

Server is running and clients can connect/play — ShineTable roundtrip works in practice. Three known issues remain for completeness but are no longer blocking:

**1. Lowercase directives** — Mimir writes `#record`/`#columntype` etc. in lowercase vs original uppercase `#Record`/`#ColumnType`. Game parser appears case-insensitive in practice.

**2. `#Exchange` / `#Delimiter` not supported** — Some files use these. Mimir ignores them on write. No crash observed so likely not affecting live data values.

**3. `#Ignore \o042` not re-emitted** — Double-quote ignore directive parsed but not written back. No crash observed.

### P1: Linux VPS Deploy — Zone "Script : Bad"

Zone startup on Linux/Wine gets through GamigoZR HTML Pass 0-4 but fails at "Script" check. Works on Windows. Likely a Mimir build issue (encoding, line endings, missing files) or a Wine file access difference. Needs investigation.

### P3: Linux Deploy — Misc TODOs

- **Firewall GamigoZR port** — binds on host network, must block external access
- **Patch Zone.exe Dbg.txt path** — Zone.exe writes `Dbg.txt` to `SysWOW64/` instead of current directory; binary-patch to use local path
- **DB bridge CPU usage** — 57-66% idle (96 IOCP threads, not CPU-count based). Consider `cpus: 0.5` in compose or binary analysis.

### P2: Environment Type Flags on `mimir env init`

Instead of remembering multiple orthogonal switches (`--passthrough server`, `--patchable client`, etc.), declare the env type once at registration time:

```bat
mimir env init server Z:/Server --type server
mimir env init client Z:/ClientSource/ressystem --type client
```

- `--type server` — auto-enables `--passthrough` on `init-template`, disables patchable
- `--type client` — auto-enables patchable/pack baseline seeding, disables passthrough

Type is persisted in `mimir.json` per env (e.g. `"type": "server"`) so all downstream commands (`init-template`, `build`, `pack`) infer correct behaviour without per-command flags.

* [x] Add `--type server|client` to `mimir env init`
* [x] `--type server`: deploy-path = provided path, import-path = provided path + `/9Data`
* [x] `--type client`: seed-pack-baseline = true (same as --patchable)
* [x] Persist type in `EnvironmentConfig` (`type` JSON field, set by `env init --type`, also settable via `mimir env <name> set type`)
* [x] `init-template` reads type → applies passthrough automatically if server (via `Passthrough` flag already set at init)
* [x] `build` reads type → applies patchable/pack baseline logic automatically if client (via `SeedPackBaseline` flag, seeds baseline at end of build)
* [x] `pack` reads type → errors clearly if env is not type client (`SeedPackBaseline || Type == "client"` check)

### ✅ P1: Log file cleanup on container restart — DONE

`start-process.ps1` Step 6 now deletes all log files from the previous run before waiting for new ones to appear, so each container restart shows only the current run's output. (Archiving to timestamped dirs deferred to backlog — deletion is sufficient for now.)

### ✅ P1: CI/CD — DONE

**Game data CI/CD**: GitHub Actions self-hosted runner. On push to `main`, workflow writes `.mimir-deploy.env` + `.mimir-deploy.secrets` from GitHub secrets/vars, removes local `mimir.bat` (falls back to PATH version), then runs `mimir deploy server/api/webapp`. `mimir init` generates `.github/workflows/deploy.yml` for new projects. `Mimir.Webhook` container removed (was untested; GitHub Actions is simpler and more reliable).

**Repo CI**: `.github/workflows/ci.yml` — runs `dotnet build` + `dotnet test` on every push/PR to main via GitHub Actions (windows-latest, .NET 10). Badge in README. Real-world integration tests skip gracefully when game file env vars are unset.

### ✅ P2: Game Management REST API

ASP.NET Core container on the same Docker Compose network as SQL Server. `src/Mimir.Api/` — minimal API, .NET 10, JWT auth, BCrypt web credentials, two-credential design.

* [x] `POST /api/auth/login` — BCrypt verify -> JWT
* [x] `POST /api/auth/set-ingame-password` — update MD5 game pw (JWT)
* [x] `POST /api/accounts` — create account (BCrypt + MD5, transactional)
* [x] `GET /api/accounts/me` — own account info (JWT)
* [x] `GET /api/accounts/me/characters` — own character list (no nUserNo in response)
* [x] `GET /api/accounts/me/cash` — own premium balance
* [x] `POST /api/accounts/me/cash` / `POST /api/accounts/{id}/cash` — add cash (admin)
* [x] `POST /api/accounts/me/inventory` / `POST /api/accounts/{id}/inventory` — give item (admin)
* [x] `GET /api/shop` / `GET /api/shop/{goodsNo}` — public shop listing
* [x] `GET /api/characters/{charNo}` — public character info (no nUserNo)
* [x] `tWebCredential` table auto-created at startup
* [x] Docker: `deploy/Dockerfile.api`, `deploy/api.bat`, api service in docker-compose.yml

**Schema notes:** `GiveItemAsync` targets `tAccountItem` (adjust to `tCashItem`/`tPremiumItem` as needed). `ShopService` targets `tMallGoods` (adjust if different). `usp_User_insert` params assumed `@userID, @userPW, @email`. Admin = `nAuthID = 9`.

### ✅ P2: CI/CD webhook container (`mimir deploy ci`) — REMOVED

`Mimir.Webhook` has been removed from the solution in favour of GitHub Actions self-hosted
runner. The webhook approach required baking Mimir.Cli + MinGit into a Windows container
and was never successfully tested in production. GitHub Actions is simpler: secrets/vars
are managed in the repo settings, and the workflow file is generated by `mimir init`.

---

### ✅ EXPLAINED — CI/CD appears to ignore committed data changes (root cause: rowEnvironments)

Change was committed and pushed. CI ran. All CI hashes match (build = deployed = container
= `e233c919`) but local manual deploy has `d65b2344` (the correct new version). CI is
consistently building the wrong version.

Hashes:
- CI build/deployed/container: `e233c919` (old — "Short Sword")
- Local manual server:          `d65b2344` (new — "Not So Short Sword")

Suspected causes:
1. **`git reset --hard origin/master` not fetching the new commit** — CI may be on a
   detached HEAD or fetching the wrong remote/branch. Check CI logs for the
   `Update repository` step — does it show the new commit hash?
2. **`mimir build` reading from wrong project dir** — `mimir.bat` walks UP from CWD to
   find `mimir.json`. If there's a stale `mimir.json` above the workspace, it would
   build from there. Check what `MIMIR_PROJ_DIR` resolves to in CI.
3. **ItemInfo.json gitignored in project repo** — if the data file is excluded by
   `.gitignore`, git would never commit it and CI would always build from a stale copy.
   Check: `git check-ignore -v data/shn/ItemInfo.json` (or wherever it lives).

Root cause was `rowEnvironments` noise stripping — now fixed. Diagnostic steps no longer needed.

### ✅ P0: BUG — CI/CD restarts SQL on every deploy → SA password race → game servers fail — FIXED

Every CI deploy triggered `mimir deploy server` → `server.bat` → `docker compose down`
(all containers including sqlserver) → same Error 18456 on restart.

**Fixes applied:**
1. `server.bat` now uses explicit service list (`stop`/`up` on game containers only —
   `login worldmanager zone00–04 account accountlog character gamelog patch-server`).
   sqlserver is never touched on a normal deploy.
2. SQL healthcheck changed from `-E` (Windows auth — passes before setup-sql.ps1 finishes)
   to `CMD-SHELL` with SA credentials (`-U sa -P %SA_PASSWORD%`) — healthcheck only passes
   once SA password is correctly set and DB recovery is complete. Game containers now start
   after the real ready state, not just after SQL Server process starts.

### ✅ P1: Patch v1 not auto-applied by fresh clients — RESOLVED (duplicate of P2 fix below)

Already fixed — see "✅ P2 bug: Patcher triggered master for v0 clients" below. Condition
`currentVersion < (minVer - 1)` correctly means v0 with minVer=1 gives `0 < 0` → false
→ incrementals. No open work.

### ✅ P1: BUG — `mimir edit` and `mimir shell` stripped `rowEnvironments` on save — FIXED

Root cause was in the **edit** and **shell** commands, not import. Both commands loaded
`TableFile.RowEnvironments` from disk but discarded it when reconstructing `TableFile`
for write-back — `SaveAllTables` and the edit save loop both created `new TableFile`
without the `RowEnvironments` field.

**Fix**: Added `tableRowEnvironments` dict alongside `tableHeaders` in both command
handlers. Captured `tableFile.RowEnvironments` on load; included it in each `TableFile`
written back. `SaveAllTables` signature extended with the dict parameter.

Import itself was already correct — it sets `RowEnvironments` on every write. The
"1433 uncommitted changes" were from a separate full-reimport that reset merge metadata
back to what the import source produces (correct behavior for a full reimport).

### ✅ P1: BUG — `certs` volume mount fails when directory doesn't exist — FIXED

`api.bat` and `webapp.bat` now set `CERT_DIR=%MIMIR_PROJ_DIR%\certs` if not already
defined, and create it with `mkdir` before `docker compose up`. Both scripts also
load `.mimir-deploy.secrets` for consistent direct-call behavior.

### ✅ P2: BUG — Intermittent SA_PASSWORD login failure on first container start — FIXED

Root cause was the healthcheck using `-E` (Windows auth) passing before `setup-sql.ps1`
finished setting the SA password. Fixed by changing the sqlserver healthcheck in
`docker-compose.yml` to use SA credentials (`CMD-SHELL` with `-U sa -P %SA_PASSWORD%`).
See ✅ P0 above — same fix resolves both issues.

### ✅ P2: Secrets system (`mimir deploy secret set/get/list`) — DONE

`deploy/secret.bat` implemented: `set KEY VALUE`, `get KEY`, `list`, `check` (interactive
prompt for missing secrets). Writes values to `.mimir-deploy.secrets` (gitignored), key
names to `.mimir-deploy.secret-keys` (committable). `mimir.bat` loads both files.
`api.bat` and `webapp.bat` also load secrets for direct-call usage.

### ✅ P2: HTTPS / Let's Encrypt + HTTP→HTTPS redirect — DONE

Both `Mimir.Api` and `Mimir.StaticServer` now support three tiers:
1. `LETSENCRYPT_DOMAIN` + `LETSENCRYPT_EMAIL` set → LettuceEncrypt auto-cert (ACME
   HTTP-01). Cert persisted to `LETSENCRYPT_CERT_DIR` (default `C:/certs`, volume-mounted).
   `ConfigureHttpsDefaults` wired to `UseLettuceEncrypt`.
2. `HTTPS_CERT_PATH` set → manual PFX loaded via `X509CertificateLoader`.
3. Neither → HTTP only (dev default).

`app.UseHttpsRedirection()` added conditionally (active when either HTTPS option is set).
`LETSENCRYPT_DOMAIN`, `LETSENCRYPT_EMAIL`, `LETSENCRYPT_CERT_DIR=C:/certs` added to api
and webapp services in docker-compose.yml. User must expose ports 80+443 in docker-compose
and set `ASPNETCORE_URLS=http://+:80;https://+:443` when using LE.

### P2: Project scaffolding (`init.bat` / project README / .gitignore)

When setting up a new Mimir project repo, several things need to exist:
- `init.bat` — finds/prompts for mimir location, sets up the project
- Auto-generated `README.md` describing the project
- Complete `.gitignore` covering: `build/`, `deployed/`, `.env.secrets`, `*.bak`, logs, etc.
- Optionally: mimir as a git submodule

* [x] `init.bat` — interactive bootstrapper in Mimir root: finds mimir CLI (PATH or dotnet run fallback), prompts for project dir, server/client paths, SA_PASSWORD, JWT_SECRET; runs env init + secret set
* [x] `mimir init` CLI command — now also scaffolds `README.md` (project layout, quick start, daily workflow, deploy steps)
* [x] Document full new-project setup in Mimir README — fixed stale `mimir init` syntax, `--passthrough` flag, `setx PATH` warning, environment config schema (per-file), project structure, CLI reference table, Common Problems section
* [x] Audit auto-generated `.gitignore` for missing entries — expanded to include
      `deployed/`, `.mimir-pack-manifest*.json`, `.mimir-deploy.secrets`, `certs/`,
      `deploy/server-files/`, `deploy/api-publish/`, `deploy/webapp-publish/`, OS files

### ✅ P2: `mimir tail` — tail container logs from project dir — DONE

`deploy/tail.bat` added. Calls `docker compose logs -f [service...]`. Added `tail` to
available scripts in `mimir.bat`. Usage: `mimir deploy tail` (all) or
`mimir deploy tail account` (single service).

### ✅ P2: Shell `.help` command — DONE

Added `.help` case to the shell dot-command switch in `src/Mimir.Cli/Program.cs`.
Prints the same four lines shown at startup.

### ✅ P2: ServerInfo.txt and SQL password handling — NOT AN ISSUE

`ServerInfo_ODBC_INFO` is imported into JSON and committed, but it only contains
the default placeholder password (`V63WsdafLJT9NDAn` — same default as docker-compose.yml).
The real `SA_PASSWORD` is set via `mimir deploy secret set` and stored in
`.mimir-deploy.secrets` (gitignored). `setup-sql.ps1` generates the live `ServerInfo.txt`
at container start from the actual secret — the Mimir-built version is always overwritten.
Nothing sensitive is in git.

### ✅ P2: `restart: on-failure` for game containers — DONE

Added `restart: on-failure` to the `x-gameserver` anchor in `docker-compose.yml`.
All game containers now auto-restart on crash. `KEEP_ALIVE=1` is orthogonal (it keeps
the container alive after the process exits via the entrypoint loop, so the container
exits 0 on graceful stop — `on-failure` only triggers on non-zero exit).

### ✅ P2: Move `deployPath` into project repo — DONE

**Decision: directory symlink** (`deploy\server-files → Z:\Server`). Binaries are not
committed to git (too large, binary, licensing), but are accessible to the Docker build
context at `COPY server-files/...` time.

* [x] Decide: external volume reference via directory symlink (direct copy and submodule both rejected)
* [x] `init.bat` now creates the symlink automatically after `mimir env server init` (with fallback instructions if symlink creation fails due to permissions)
* [x] Generated project `README.md` (`mimir init`) documents `mklink /D deploy\server-files Z:\Server` as step 1 of the Deploy section
* [x] `deploy/server-files/` is already in `.gitignore`

### Split `mimir` CLI into focused sub-projects

As the CLI grows, split `Mimir.Cli` into separate dotnet tool projects by domain so each is small, focused, and independently documentable/discoverable:

- `Mimir.Env` (`mimir env`) — environment management (init, set, list, reseed-baseline)
- `Mimir.Sql` (`mimir sql`) — query/edit/shell (query, edit, shell, validate)
- `Mimir.Data` (`mimir data`) — data pipeline (import, build, init-template, edit-template, analyze-types)
- `Mimir.Deploy` (`mimir deploy`) — Docker deployment lifecycle (start, stop, update, restart, rebuild-*, set)
- `Mimir.Patch` (`mimir patch`) — pack/patch index management (pack, shn inspection)

Each project is a standalone dotnet global tool. `mimir.bat` in project directories dispatches to the appropriate sub-tool. Composable via scripts; keeps the monolith from growing unbounded.

* [ ] Define sub-project split boundaries (which commands go where)
* [ ] Create solution structure: one csproj per sub-tool, shared `Mimir.Cli.Common` for DI/project resolution
* [ ] Migrate commands from `Mimir.Cli` into the appropriate sub-project
* [ ] Update `mimir.bat` dispatcher to route `mimir env ...` → `mimir-env.exe ...` etc.
* [ ] Publish all sub-tools as dotnet global tools (or a single `mimir` meta-tool that delegates)
* [ ] Update `mimir init` to scaffold the updated `mimir.bat` pointing to sub-tools

### ✅ P2: Custom Web App Deploy (`mimir deploy webapp`) — DONE

Allow anyone to ship a web frontend (admin panel, player portal, etc.) alongside the game
stack. The key value is the **deploy infrastructure** — the web app tech is pluggable.
`Mimir.StaticServer` is an optional built-in convenience; users who want Next.js, Vite SSR,
or anything else bring their own container.

#### Design goals

- **Infrastructure first**: `webapp.bat` + docker-compose service are the core. The container
  image is a user-controlled variable, not a Mimir concern.
- **`Mimir.StaticServer` is optional**: shipped as a zero-config default for pre-built static
  output (e.g. a Vite SPA build). Not required. No YARP proxy, no complexity — just static
  files + SPA fallback.
- **Next.js preferred** for a real management UI: file-based routing, SSR, built-in API
  proxying via `next.config.js` rewrites. No CORS needed because the Next.js server proxies
  API calls over the Docker network (server-to-server), and the browser only ever talks to
  the Next.js origin.
- **Vite** (e.g. Vite + React/Vue) is the right choice for a lightweight static panel. Build
  output goes in `webapp-dist/`, Mimir.StaticServer serves it, API calls need CORS (see below).

#### Two runtime modes

```
Mode A — Static (Vite SPA output → Mimir.StaticServer)
  Browser ──port 8080──> Mimir.StaticServer ──wwwroot──> HTML/JS/CSS
  Browser ──port 5000──> Mimir.Api          (CORS required)

Mode B — SSR/Server (Next.js or any custom server, user's own container)
  Browser ──port 8080──> Next.js server     ──SSR pages──> HTML
  Next.js  ──port 5000──> Mimir.Api          (Docker network, no CORS)
  browser fetch('/api/...')  →  Next.js rewrites  →  http://api:5000/api/...
```

Mode B is the better long-term approach. Mode A is the quick-start path.

#### `Mimir.StaticServer` (`src/Mimir.StaticServer/`)

Minimal static file server. No proxy, no YARP. ~5 lines of code:

```csharp
// Program.cs
var app = WebApplication.Create(args);
app.UseDefaultFiles();
app.UseStaticFiles();           // serves from wwwroot (volume-mounted)
app.MapFallbackToFile("index.html");  // SPA client-side routing
app.Run();
```

- Port: configurable via `ASPNETCORE_URLS`, defaults to `http://+:8080`
- wwwroot: `C:/app/wwwroot` (volume-mounted from `<project>/webapp-dist/`)
- No packages beyond the web SDK

#### CORS on `Mimir.Api`

For Mode A, the browser calls the API at port 5000 from a page served at port 8080 — a
cross-origin request. Add a `CORS_ORIGINS` env var to Mimir.Api:

```csharp
var origins = builder.Configuration["CORS_ORIGINS"]?.Split(',') ?? [];
if (origins.Length > 0)
    builder.Services.AddCors(o => o.AddDefaultPolicy(p => p.WithOrigins(origins).AllowAnyHeader().AllowAnyMethod()));
// ...
if (origins.Length > 0) app.UseCors();
```

In docker-compose webapp service: `CORS_ORIGINS=http://localhost:8080` (or `*` for open dev).
In the `api` service: `- CORS_ORIGINS=${CORS_ORIGINS:-}` (empty = CORS disabled by default).

#### Pluggable build context in docker-compose

The `webapp` service uses env vars for the build context and Dockerfile, so swapping in a
Next.js container requires only `mimir deploy set` calls — no compose file edits:

```yaml
  webapp:
    build:
      context: ${WEBAPP_CONTEXT:-.}
      dockerfile: ${WEBAPP_DOCKERFILE:-Dockerfile.webapp}
    profiles: [webapp]
    ports:
      - "8080:8080"
    environment:
      - API_URL=http://api:5000
    volumes:
      - ../${PROJECT_NAME:-test-project}/webapp-dist:C:/app/wwwroot:ro
    depends_on:
      api:
        condition: service_started
```

Defaults (`context: .`, `dockerfile: Dockerfile.webapp`) give Mimir.StaticServer behaviour.
For Next.js: `mimir deploy set WEBAPP_CONTEXT=Z:\my-nextjs-app` and
`mimir deploy set WEBAPP_DOCKERFILE=Dockerfile`. No volume mount needed (app is baked in).

#### `deploy/Dockerfile.webapp` (default — Mimir.StaticServer)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-windowsservercore-ltsc2022
WORKDIR /app
EXPOSE 8080
COPY webapp-publish/ .
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "Mimir.StaticServer.dll"]
# wwwroot is volume-mounted from <project>/webapp-dist/ at runtime.
# To use Next.js or another server: replace this file and set WEBAPP_CONTEXT + WEBAPP_DOCKERFILE.
```

#### `deploy/webapp.bat <project> [source-dir]`

```bat
@echo off
setlocal
if "%~1"=="" ( echo ERROR: Project name required. & exit /b 1 )
set "PROJECT=%~1"

:: Optional: sync an external built output dir into the project's webapp-dist/
if not "%~2"=="" (
    echo Syncing webapp files from %~2 ...
    if not exist "%~dp0..\%PROJECT%\webapp-dist" mkdir "%~dp0..\%PROJECT%\webapp-dist"
    robocopy "%~2" "%~dp0..\%PROJECT%\webapp-dist" /MIR /NP
    if %errorlevel% gtr 7 ( echo ERROR: robocopy failed. & exit /b 1 )
)

:: If WEBAPP_CONTEXT points to a user container, skip publishing StaticServer
if /i "%WEBAPP_CONTEXT%"=="" (
    dotnet publish "%~dp0..\src\Mimir.StaticServer" -c Release -o "%~dp0webapp-publish" --no-self-contained
    if errorlevel 1 ( echo ERROR: dotnet publish failed. & exit /b 1 )
)

set COMPOSE_PROJECT_NAME=%PROJECT%
set PROJECT_NAME=%PROJECT%
set DOCKER_BUILDKIT=0
docker compose -f "%~dp0docker-compose.yml" --profile webapp build webapp
docker compose -f "%~dp0docker-compose.yml" --profile webapp up -d webapp
```

#### Next.js workflow (Mode B)

```bat
:: One-time setup: tell mimir where the Next.js project lives
mimir deploy set WEBAPP_CONTEXT=Z:\my-nextjs-app
mimir deploy set WEBAPP_DOCKERFILE=Dockerfile

:: The Next.js project needs a Dockerfile that knows about ASPNETCORE_URLS / port 8080.
:: next.config.js rewrites: /api/:path* -> http://api:5000/api/:path*

:: Deploy (builds the Next.js image, starts container)
mimir deploy webapp
```

Requires: `Dockerfile` in the Next.js project that exposes port 8080 and calls `npm start`.
Windows container base: `node:20-windowsservercore-ltsc2022`.

#### Vite SPA workflow (Mode A)

```bat
:: Build the app
cd Z:\my-vite-app && npm run build      :: output to dist/

:: Deploy (syncs dist/ to webapp-dist/, builds StaticServer image, starts container)
mimir deploy webapp Z:\my-vite-app\dist

:: Access at http://localhost:8080
:: API calls from JS: fetch("http://localhost:5000/api/accounts/me", { headers: { Authorization: ... } })
:: Requires CORS_ORIGINS set: mimir deploy set CORS_ORIGINS=http://localhost:8080
```

#### Checklist

* [x] `src/Mimir.StaticServer/Mimir.StaticServer.csproj` (web SDK + LettuceEncrypt)
* [x] `src/Mimir.StaticServer/Program.cs` (config.json endpoint + static files + SPA fallback + optional HTTPS)
* [x] `src/Mimir.StaticServer/wwwroot/index.html` + `app.js` + `style.css` — SPA: leaderboard / register / login / change-password
* [x] `deploy/Dockerfile.webapp` (aspnet:10.0 base, copies webapp-publish/)
* [x] `deploy/webapp.bat` (conditional publish, docker compose up)
* [x] `deploy/docker-compose.yml` — webapp service + CORS/Turnstile/reCAPTCHA/HTTPS env vars on api
* [x] `src/Mimir.Api/Program.cs` — rate limiting, CORS, /api/config, CaptchaService, optional HTTPS
* [x] `src/Mimir.Api/Services/CaptchaService.cs` — Turnstile / reCAPTCHA / no-op provider
* [x] `src/Mimir.Api/Services/CharacterService.cs` — GetLeaderboardAsync (TOP 100 by exp)
* [x] `src/Mimir.Api/Endpoints/CharacterEndpoints.cs` — GET /api/leaderboard
* [x] `src/Mimir.Api/Endpoints/AccountEndpoints.cs` — captcha verify on POST /api/accounts
* [x] `src/Mimir.Api/Endpoints/AuthEndpoints.cs` — captcha-after-failure login, set-web-password
* [x] `src/Mimir.Api/Services/AccountService.cs` — SetWebPasswordAsync
* [x] `src/Mimir.Api/Models/AccountModels.cs` — CaptchaToken fields, LoginResponse with RequiresCaptcha
* [x] `Mimir.sln` — added Mimir.StaticServer (GUID `{A1B2C3D4-000B-000B-000B-00000000000B}`)
* [x] `mimir.bat` — added `webapp` to Available scripts list

### P3: Extract Patch System into Standalone Library (`Patcher`)

The patch pack/apply logic is currently embedded in `Mimir.Cli`. Extract it into a reusable
C# library (potentially its own repo/solution) so that anyone rolling their own patcher
— visual GUI, custom client launcher, etc. — can just import the library rather than
depending on Mimir.

#### Proposed package split

```
Patcher/                        ← own repo/solution
  Patcher.Core/                 — shared models: PatchIndex, PatchEntry, VersionFile, SHA-256 helpers
  Patcher.Server/               — server-side: pack zips, build/update patch-index.json, prune old patches
  Patcher.Client/               — client-side: fetch index, compare version, download + verify + extract
  Patcher.ClientCLI/            — thin CLI wrapper around Patcher.Client
                                   dotnet run Patcher.ClientCLI -- --server http://127.0.0.1:8080 --repair
```

`Mimir.Cli` pack command becomes a thin wrapper:
```csharp
// mimir pack --env client
var packer = new Patcher.Server.PatchPacker(buildOutputDir, patchesDir);
await packer.PackAsync(baseManifest, patchIndexPath);
```

`Patcher.ClientCLI` is a standalone dotnet tool players can run directly — no Mimir dependency.
Third parties import `Patcher.Client` to build a visual launcher (WPF, Avalonia, WinForms, etc.)
and call `PatchClient.CheckForUpdatesAsync()` / `PatchClient.ApplyAsync()` directly.

#### Checklist

* [ ] Define `Patcher.Core` models (PatchIndex, PatchEntry, VersionFile) extracted from current Mimir code
* [ ] `Patcher.Server` — PatchPacker: build index, zip changed files, prune incrementals vs master
* [ ] `Patcher.Client` — PatchClient: fetch index, version compare, download + SHA-256 verify + extract
* [ ] `Patcher.ClientCLI` — `--server <url>`, `--repair`, `--version`, progress output to stdout
* [ ] `Mimir.Cli` pack command delegates to `Patcher.Server` NuGet package (or project reference)
* [ ] Decide: same repo/solution as Mimir, or separate `ProjectMimir.Patcher` repo
* [ ] Publish `Patcher.Client` and `Patcher.Server` as NuGet packages
* [ ] Publish `Patcher.ClientCLI` as a dotnet global tool

### P3: Trial Linux Containers with Wine

> Current deploy stack uses Windows Server Core containers (large images, slow builds, no Docker BuildKit). Investigate whether running the Fiesta game server processes under Wine inside Linux containers is viable. Benefits: smaller images, faster builds, BuildKit support, wider CI runner availability, cross-platform hosting, cheaper VPS/cloud hosting costs (Linux VMs are significantly cheaper than Windows Server licensed instances).

**Infrastructure complete** — `docker-compose.linux.yml` + all Linux Dockerfiles created. Toggle via `MIMIR_OS=linux` in `.mimir-deploy.env` (or `mimir deploy set MIMIR_OS linux`). All deploy scripts (`server.bat`, `rebuild-game.bat`, `rebuild-sql.bat`, `start.bat`, `stop.bat`, `logs.bat`, `restart-game.bat`, `tail.bat`, `update.bat`, `wipe-sql.bat`, `api.bat`, `webapp.bat`) now select compose file based on `MIMIR_OS`.

Key questions:
- Do game server exes (Zone, Account, Login, etc.) run under Wine without crashing?
- Does ODBC/SQL Server connectivity work via Wine + FreeTDS or unixODBC?
- Are there Wine compatibility blockers for the COM/Windows APIs the exes use?
- Image size comparison: Wine Linux image vs Windows Server Core image

* [x] Create `docker-compose.linux.yml`, `Dockerfile.server.linux`, `Dockerfile.sql.linux`, `Dockerfile.patch.linux`
* [x] Create `scripts/start-process.sh` (bash+Wine equiv of start-process.ps1) and `scripts/setup-sql.linux.sh`
* [x] Create `config.linux/` with `sqlserver,1433` instead of `sqlserver\SQLEXPRESS`
* [x] Wire `MIMIR_OS` switch into all deploy `.bat` files
* [ ] Spike: run a single game process (e.g. Login) under Wine in a Linux container, verify it starts and accepts TCP connections
* [ ] Test SQL Server connectivity: Wine + FreeTDS ODBC driver → SQL Server container
* [ ] Benchmark image size and build time vs current Windows containers
* [ ] Document findings — go/no-go for replacing Windows containers with Linux+Wine stack

### P2: Project CLI bootstrap ("mimir setup" or generated setup.sh)

> After cloning a Mimir project, the generated `mimir.sh` calls `mimir "$@"` but
> `mimir` isn't on PATH yet. Every collaborator needs to manually create an alias
> or symlink. This is a friction point for onboarding.
>
> Possible solutions:
> - Generate a `setup.sh` in the project that prompts for the Mimir repo location
>   and creates the alias/symlink (e.g. `ln -s ~/mimir/mimir.sh /usr/local/bin/mimir`)
> - Have `mimir init --mimir <path>` bake an absolute path into the generated scripts
>   (already supported via `--mimir` flag, but not discoverable)
> - Auto-detect: project `mimir.sh` walks up / checks `~/.mimir` / reads a
>   `.mimir-cli-path` file to find the CLI
> - `dotnet tool install` global tool approach (publish Mimir.Cli as a .NET tool)
>
> Whichever approach, it should be a single command after clone that Just Works.

* [ ] Decide on approach
* [ ] Implement setup script or auto-detection
* [ ] Update `mimir init` to generate the bootstrap mechanism
* [ ] Document in generated README

### P3: KIND Kubernetes Setup

Local multi-node Kubernetes cluster via KIND (Kubernetes in Docker) for testing deployment on k8s before going to production. Builds on the Docker Compose stack — same images, translated to k8s manifests.

* [ ] KIND cluster config (1 control-plane + 2 workers)
* [ ] Helm chart or raw manifests for all 12 game containers
* [ ] SQL Server StatefulSet with PersistentVolume for databases
* [ ] ConfigMap for ServerInfo / per-process config
* [ ] NodePort or LoadBalancer for game ports (9010, 9013, 9016–9028)
* [ ] Liveness probes tied to Windows service health
* [ ] Namespace isolation per Mimir project (mirrors per-project Docker naming)

### ✅ P1: Per-project deploy config (`mimir deploy set`) — DONE

`mimir deploy set KEY=VALUE` writes a key=value pair to `<project>/.mimir-deploy.env`. `mimir.bat` automatically loads this file into the environment before calling any deploy script, so variables like `SA_PASSWORD` are available to Docker Compose without being hardcoded in the compose file.

```bat
:: Set a custom SQL Server password for this project
mimir deploy set SA_PASSWORD=MyStrongPassword1
:: All subsequent deploys use it automatically
mimir deploy start
```

`SA_PASSWORD` in `docker-compose.yml` is now `${SA_PASSWORD:-V63WsdafLJT9NDAn}` (default preserved for zero-config setup). The `.mimir-deploy.env` file should be gitignored (contains secrets).

### ✅ P2: `mimir.bat` deploy forwarding + per-project containers — DONE

`mimir deploy <script>` from inside any project dir walks up to find `mimir.json`, derives the project name from the directory name, and calls `deploy\<script>.bat <project-name>`.

Deploy scripts set `COMPOSE_PROJECT_NAME=%PROJECT%` and `PROJECT_NAME=%PROJECT%` so Docker Compose automatically namespaces all containers and volumes per project. Volume paths in `docker-compose.yml` use `${PROJECT_NAME:-test-project}`. All hardcoded `container_name:` entries removed.

```bat
:: from inside Z:/my-server/
mimir deploy update         → build + pack + snapshot + restart
mimir deploy restart-game   → snapshot + restart (skip build)
mimir deploy start/stop/logs/rebuild-game/rebuild-sql/reimport
```

### ✅ P2 bug: Patcher triggered master for v0 clients — FIXED

Master condition was `currentVersion < minIncrementalVersion`. With minVer=1 a fresh client (v0) incorrectly downloaded the full master instead of applying v1 normally. Fixed to `currentVersion < (minIncrementalVersion - 1)` — v0 with minVer=1 gives `0 < 0` → false → incrementals. repair.bat now writes -1 to `.mimir-version` so `-1 < 0` → master. Fixed in `deploy/player/patch.bat` and `Program.cs` template.

### ✅ P2: Incremental pruning + master patch fallback — DONE

Every `mimir pack` run produces both an incremental patch and a full master snapshot. Pruning removes oldest incrementals once their total size exceeds the master. Patcher falls back to master for clients too old for incrementals or with a corrupted/missing version file. patch-index.json shape: `latestVersion`, `masterPatch`, `minIncrementalVersion`, `patches[]`.

### ✅ P2: Deploy env vars take effect on restart — DONE (already was)

`restart-game.bat` already uses `--force-recreate`, which recreates containers with
fresh env vars without rebuilding the image. Env-only changes (KEEP_ALIVE, SA_PASSWORD,
etc.) — run `mimir deploy restart`. Image changes (Dockerfile, scripts, binaries) —
run `mimir deploy rebuild-game`.

### ✅ P2c: Port shift for simultaneous servers — DONE

`docker-compose.yml` uses `${LOGIN_PORT:-9010}`, `${WM_PORT:-9013}`, etc. for every service.
`mimir.bat` now auto-computes all port vars from a single `PORT_SHIFT` value (set via
`mimir deploy secret set PORT_SHIFT 100`). Individual vars take precedence if already set.

Shifts: LOGIN/WM/ZONE00–04 game ports, PATCH_PORT, API_PORT, WEBAPP_PORT, CI_PORT.
SQL_PORT is not shifted (internal-only, clients never connect directly).

**Usage:**
```bat
:: Second server instance — all ports shifted by 100
mimir deploy secret set PORT_SHIFT 100
:: Login: 9110, WM: 9113, Zone00: 9116 … Zone04: 9128, Patch: 8180, API: 5100
```

### P1: BUG — Skill errors ("Invalid DemandSk") on zone startup

> Zone server logs `Invalid DemandSk` errors on startup. Errors appear cosmetic (zone runs
> fine) but may indicate data corruption introduced by recent Mimir merge/split pipeline
> changes. Need to diff built skill-related SHN files against source originals to check for
> data loss or column reordering.
>
> Potentially affected tables: `SkillData`, `SkillInfo`, `ActiveSkill`, `PassiveSkill`,
> `SkillTree`, or similar skill-related SHN files.

* [ ] Identify which SHN tables contain skill data (`DemandSk` column or similar)
* [ ] Diff Mimir-built SHN vs source originals for those tables (`mimir shn --diff`)
* [ ] If data differs, trace through merge/split pipeline to find where corruption occurs
* [ ] Fix and verify zone starts without skill errors

### ✅ P1: BUG — SQL Server rebuild race condition (SA_PASSWORD cleared after rebuild) — FIXED

> `rebuild-sql.bat` and `wipe-sql.bat` previously cleared `SA_PASSWORD` from `.env` after
> volume wipe, causing `set-sql-password` to have no `OLD_PASSWORD` to authenticate
> `ALTER LOGIN`. Result: password mismatch → healthcheck fails for 130s → game servers
> fail to start (appears as a "hang").
>
> **Root cause**: Clearing SA_PASSWORD was unnecessary — `setup-sql.ps1` already handles
> password sync on startup (tries configured SA_PASSWORD, falls back to install default,
> ALTER LOGINs if needed).
>
> **Fix**: Removed SA_PASSWORD clearing from both `rebuild-sql.bat` and `wipe-sql.bat`.

### ✅ P1: BUG — DOCKER_BUILDKIT trailing space in batch scripts — FIXED

> `set DOCKER_BUILDKIT=0` inside inline `( )` blocks captured a trailing space before `)`,
> producing value `"0 "` which Docker rejected as a non-boolean. Affected 6 deploy scripts:
> `webapp.bat`, `api.bat`, `start.bat`, `wipe-sql.bat`, `rebuild-game.bat`, `server.bat`.
>
> **Fix**: Quoted all assignments: `set "DOCKER_BUILDKIT=0"`.

### ✅ P1: BUG — start-process.ps1 keep-alive unreachable + short log drain — FIXED

> Zone containers boot-looped because: (1) service crashed before reaching "Running" state →
> `$svcSeenRunning` never set → keep-alive block unreachable → container exits; (2) 1.5s
> passive log drain missed output still being flushed.
>
> **Fix**: Added `$svcNeverRanTimeout = 30` — if service hasn't been seen running after 30s
> and status is "Stopped", treat as crash (same exit path as after-running crash). Increased
> log drain to 5s with active polling (10 x 500ms with Receive-Job).

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

### P8: QuestData Field Mapping — IN PROGRESS

> Map more of the 678-byte fixed data region beyond QuestID. See `quest-data-format.md` for
> ongoing analysis. Confirmed: no checksum in file, in-place byte modifications work. Using
> Mimir JSON pipeline to create test quests (edit FixedData hex in QuestData.json, `mimir build`).
>
> **Confirmed fields**: QuestID(2-3), BaseDialogID(6-7), b14(QuestFlags), b15(ButtonMask/QuestType),
> StartLevel(25), EndLevel(26), StartNPCID(28-29), GateQuestID(32-33), PreviousQuestID(56-57),
> IsClassRestricted(60), RequiredClassID(61), Objectives(90+), EXP(662-663).
>
> **Next**: Create test quests with individual field changes to determine meanings of b14, b15,
> b16, b17, bytes 658-661 (SP/fame?), bytes 666-677 (reward table refs?).

* [x] Reverse-engineer file structure (version, count, recordLength, fixedData + 3 scripts)
* [x] Map bytes 0–35 (quest identity, dialog, levels, NPCs)
* [x] Map bytes 36–63 (chain predecessor, class restriction)
* [x] Map bytes 86–189 (objectives region, 8-byte slot layout)
* [x] Map bytes 658–677 (EXP reward, candidate SP/fame/reward refs)
* [x] Confirm no checksum (manual hex edit → server starts fine)
* [ ] In-game test: b14 variants (quest flags/icon)
* [ ] In-game test: b15 variants (dialog button types)
* [ ] In-game test: b16/b17 (remote accept/hand-in)
* [ ] In-game test: bytes 658-661 (SP/fame candidate)
* [ ] In-game test: bytes 666-677 (reward table refs — zero them)
* [ ] Map bytes 190–513 (extended objectives region)
* [ ] Map bytes 514–657 (suspected reward items region)
* [ ] Incrementally extract known fields into proper named columns in QuestDataProvider

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

### Server build path should target 9Data directly ✓ DONE

**Convention:** `build/server/` IS the 9Data directory. Set `importPath = Z:/Server/9Data` so
sourceRelDir is `Shine/` etc. (not `9Data/Shine/`), and build output lands at
`build/server/Shine/ItemInfo.shn` directly — no `9Data/` subdirectory inside `build/server/`.

- `deploy/docker-compose.yml` volume changed: `deployed/server/9Data:C:/server/9Data`
  → `deployed/server:C:/server/9Data` — the whole `deployed/server/` is mounted as 9Data.
- Deploy scripts robocopy `build\server` → `deployed\server` (unchanged); since `importPath`
  now points at `Z:/Server/9Data`, only game data files are ever imported — no server-root
  exes or scripts can end up in the build output.
- `deployPath` env property added (`mimir env server set deploy-path Z:/Server`) to record
  where server binaries (exes, DLLs, GamigoZR, etc.) live separately from `buildPath`.
  Deploy scripts don't yet act on it — wires up with standalone project / P4 feature.

Remaining: `buildPath` default on `mimir env init` is still `build/{envName}`. Changing it
to be type-aware (`build/{envName}` = 9Data for server, `build/{envName}` = ressystem for
client) is part of P2 Environment Type Flags.

### Auto-archive old log files on container restart

On container restart, existing game server log files (Assert logs, ExitLogs, Msg logs, etc.) should be moved into an `old/` or timestamped subfolder rather than being appended to or overwritten. Makes it much easier to isolate logs from the current run vs. previous runs.

### Docker containers should exit when their game process exits

Currently containers stay alive even after the game server process (Login.exe, Zone.exe, etc.) shuts down or crashes. Once the deployment setup is stable, containers should be configured to exit when their main process does — so `docker ps` reflects actual server health and `docker compose up` restarts crashed processes correctly. Likely just a matter of ensuring the entrypoint doesn't swallow exit codes and that `restart: on-failure` or similar policy is set in `docker-compose.yml`.

### Deploy scripts should be callable from inside the project folder

Currently `deploy.bat`, `update.bat`, etc. live in `deploy/` and must be run from there (they use relative paths). They should be callable from the project root via a `mimir-deploy.bat` shim (or similar) that forwards to the real scripts with correct working directory context.

Similarly, `mimir init` should scaffold a `mimir.bat` in the new project directory that forwards all commands to the mimir executable used to run `init` — so `mimir.bat build`, `mimir.bat import`, etc. work from the project root without needing to know the install path. The init command already writes a basic `mimir.bat`, but it should capture the actual invocation path (e.g. `dotnet run --project ...` or the installed exe path) rather than assuming `mimir` is in PATH.

### deploy.bat wipes SQL database unconditionally ✓ FIXED

`deploy.bat` never called `rebuild-sql.bat` (tracker entry was stale). `setup-sql.ps1` already
skipped restoring databases that exist in the volume. Fixed the remaining risk:
- Existence check improved: now uses `SET NOCOUNT ON; SELECT COUNT(*) FROM sys.databases WHERE name = N'$db'`
  and matches on the numeric result, not string content — resistant to error text false matches.
- Removed `WITH REPLACE` from both restore paths (`WITH REPLACE` would silently overwrite a live
  database if the existence check ever false-negatives; without it, SQL Server refuses to restore
  over an existing database, making any bug in the check a safe, visible error instead).

### SQL password management

No good way to set/change the SQL Server password used by the game server and Docker setup. Currently hardcoded in deploy scripts and `ServerInfo.txt`. Need a proper mechanism — e.g. `mimir env server set sql-password <pw>` or a dedicated secrets file — so the password can be configured per-project without editing raw config files.

### Reimport to dummy dir for more accurate baseline seeding

Currently `mimir build` seeds the pack baseline from the actual build output directory, which means the baseline reflects Mimir's rebuilt files (not the original source files). A more accurate baseline — useful for measuring true diff vs. the stock client — could be produced by running a temporary import+build into a throwaway directory using the original source files, then seeding from that. This would let patch v1 contain only files the user actually changed vs. vanilla, even accounting for roundtrip fidelity. Low priority for now; the current build-output approach is pragmatic and produces correct incremental patches.

### `mimir pack` should auto-seed baseline if manifest missing

Currently if no pack manifest exists, `mimir pack` treats all files as new and produces a patch containing everything. Instead, `mimir pack` should automatically seed the baseline from the current build output (same logic as `mimir build` post-build seeding), then diff against it — producing a 0-file patch on the first run. This makes the workflow forgiving: even if the manifest was deleted or never created, the user can just run `mimir pack` and it self-heals without needing to re-run `mimir build`.

### Always create patch-index.json even when no changes to pack

If no pack manifest exists or there are no changed files, `mimir pack` currently exits early without writing `patch-index.json`. The index file should always be created/updated on every pack run — even a no-change run — so that clients can always find a valid index to query. An empty/current index with no new patch entry is a valid and useful state.

### Overrides must never be included in the pack baseline

`mimir build` currently seeds the pack baseline from the full build output directory, which includes any files copied from the overrides folder. This means override files are "known" to the baseline and only show up in a patch if they subsequently change. Instead, the baseline should only cover files produced by the core build (tables + copyFile actions) — override files should always be excluded from baseline seeding so that the first patch after a build always delivers them to players. After that first delivery they diff normally like everything else.

### Quest "available" tab shows fewer quests than expected

Early-game quests are present and functional — NPCs display and serve them correctly when clicked. However, several quests that should appear in the client's "available" quests tab do not show up (e.g. Archer sees only a few quests through level 5 rather than the expected set).

This is a **client-side display issue**, not a server data problem. Likely causes:
- **Outdated client files**: The client's `QuestData.shn` (or related quest-display files) may be from a different game version than the server. If the client's local quest conditions/availability data doesn't match what the server expects, the "available" filter logic may silently exclude quests.
- **Client-side QuestData.shn built by Mimir**: If the client env includes QuestData and Mimir rebuilt it, roundtrip fidelity issues (fixed-data region, PineScript strings) could affect the availability conditions the client evaluates.

**To investigate**: Compare client's `QuestData.shn` (from `Z:/ClientSource`) against `Z:/Server/9Data/Shine/QuestData.shn` using `mimir shn --diff`. If they differ, that's the source mismatch. Also check whether client `QuestData.shn` is included in the client build output at all.

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

### SHN ViewData Checksum — possible in-header integrity value

Zone log shows informational messages (not errors) for ~14 `*View.shn` files:
```
[Message] SHN - ViewData Checksum - AbStateView.shn
[Message] SHN - ViewData Checksum - ActiveSkillView.shn
... (ItemShopView, ItemViewInfo, MapViewInfo, MobViewInfo, NPCViewInfo, etc.)
```
These files likely contain an embedded checksum of their own content (in the data or header). Mimir preserves the `cryptHeader` bytes verbatim but if a checksum covers the decrypted record data, Mimir's rebuilt files (with zeroed string padding) will have a wrong checksum. This could cause silent data corruption or rejection in the client. Needs investigation of SHN header format to determine if/where a checksum field lives.

### Client "illegally manipulated" hash check failure ✓ RESOLVED

Root cause was a DB/client version mismatch. Using matching client source resolved it. Not a Mimir roundtrip fidelity issue.

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

### ItemInfo/ItemInfoServer row order mismatch ✓ RESOLVED

See P0b above. Row order fixed (TableMerger single-pass), duplicate join key fixed (Queue-based FIFO), built ItemInfo.shn now byte-identical to source. Zone.exe starts cleanly.

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

# Bugs

## Next Up

* **Per-deployment patch state** — Pack manifest and patch-index.json are currently per-project, but different deployments (dev vs prod) will diverge in patch state. Need a way to scope pack manifests and patch indexes per deployment so that dev and prod can have independent patch versioning.
* **GitHub Actions deploy button** — Add a `workflow_dispatch` workflow to the private `my-server` repo (NOT ProjectMimir). Single manual button on production branch that SSHs to the VPS and runs full deploy. Needs GitHub secrets: VPS_HOST, VPS_USER, VPS_SSH_KEY. **Do NOT commit this to ProjectMimir** — generate the file and give it to the user to place in their private repo. (In progress)
* **Server message tool via OpTool** — Connect OpTool (or similar) to send in-game server messages like "Server restarting in 5 minutes" as part of the GitHub Actions deploy pipeline. User already has a working script for dev. Set up for prod on VPS. (P2, my-server repo)
* **Auto patch log from git commits** — Collect all git commit messages since last deploy and generate a patch changelog automatically. Integrate into the deploy pipeline. (P3, my-server repo)

# Bugs

* **Leaderboard class IDs are wrong** — `CLASS_NAMES` mapping in `app.js` is incorrect. What shows as "Archer" (ID 1) is actually Fighter, etc. Need to verify correct class ID → name mapping from the game data and fix the map in `src/Mimir.StaticServer/wwwroot/app.js`.
* **Leaderboard shows "Race" column** — Fiesta doesn't have races. Remove `RACE_NAMES` and the Race column from the leaderboard in `app.js`.
* ~~**Build always resets pack baseline**~~ (fixed — `SeedBaselineAsync` now skips if manifest already exists)
* **Deploy: do full build before restarting containers** — `server.sh` stops containers, builds, then restarts. Should do the full build+pack+rsync first, only then stop/restart containers to minimize downtime. (P2)
* **Deploy: network connect errors** — `docker network connect` fails with "endpoint already exists" when containers are already on the network. Add `|| true` to prevent deploy script from exiting with error. (P2, my-server repo)