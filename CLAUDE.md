# Mimir - Fiesta Online Server Data Toolkit

## Environment

- **Platform**: Windows — use forward slashes (`Z:/path`) or double backslashes (`Z:\\path`) in bash/CLI commands
- **.NET version**: 10.0 (SDK 10.0.102)
- **GitHub**: https://github.com/IkaronClaude/ProjectMimir

## Quick Reference

- **Build**: `dotnet build Mimir.sln`
- **Test**: `dotnet test Mimir.sln`
- **Run CLI**: `dotnet run --project src/Mimir.Cli -- <command>` (run from inside the project dir; CWD walks up to find mimir.json like git)
- **Import**: `dotnet run --project src/Mimir.Cli -- import`
- **Init template**: `dotnet run --project src/Mimir.Cli -- init-template`
- **Edit template**: `dotnet run --project src/Mimir.Cli -- edit-template --table ColorInfo --conflict-strategy split`
- **Build for env**: `dotnet run --project src/Mimir.Cli -- build --env server`
- **Build all envs**: `dotnet run --project src/Mimir.Cli -- build --all`
- **Pack client patches**: `dotnet run --project src/Mimir.Cli -- pack --env client` (incremental zip + patch-index.json)
- **Override project**: add `-p <path>` / `--project <path>` to any command

## Data Locations

- **Server**: `Z:/Server/` (~218 SHN + hundreds of .txt) — import path for `server` environment. **Do NOT use as ground truth** — may become contaminated by manual copies. `Z:/ServerSource/` is the clean unmodified server backup (ground truth).
- **Client**: `Z:/ClientSource/ressystem` — import path for `client` environment (unmodified original SHN files, ground truth). `Z:/Client/` is the already-patched game client — do NOT use as import source.
- **Overlap**: 118 shared SHN files, 102 server-only, 39 client-only
- Client-only files are mostly UI/text/visual (BasicInfo*, TextData*, FontSet, DamageEffect, etc.)
- `.shd` files in ressystem are height maps / walkability grids, not data tables
- **Build output preserves directory structure**: server builds to `build/server/9Data/Shine/ItemInfo.shn`, client builds to `build/client/ressystem/ItemInfo.shn`

## Project Structure

```
src/
  Mimir.Core/          Core models, interfaces, definitions system
  Mimir.Shn/           SHN binary format (encrypted, EUC-KR encoded)
  Mimir.ShineTable/    Text table formats (#table and #DEFINE)
  Mimir.Sql/           SQLite in-memory engine
  Mimir.Cli/           CLI entry point (import/build/query/edit/shell)
tests/
  Mimir.Core.Tests/         DefinitionResolver tests
  Mimir.Shn.Tests/          (empty)
  Mimir.ShineTable.Tests/   Table + Define format round-trip tests
  Mimir.Sql.Tests/          SqlEngine load/extract/edit tests
  Mimir.Integration.Tests/  Full pipeline integration tests (synthetic + real-world)
test-project/               Real imported data (gitignored, ~1234 tables)
```

## Key Conventions

- **Development method**: Test Driven Development (TDD) — write failing tests first, then implement to make them pass
- **Test framework**: xUnit + Shouldly + NSubstitute (Directory.Build.props in tests/)
- **Test files need** `using Xunit;` (not auto-imported via global usings)
- **Format IDs**: `"shn"`, `"shinetable"`, `"configtable"` - never use old names "rawtable"/"rawtable-define"
- **Parser classes**: `ShineTableFormatParser` (#table format), `ConfigTableFormatParser` (#DEFINE format)
- **Definitions file** (`mimir.definitions.json`): Lives at repo root, NOT inside test-project. Searched upward from project dir.
- **SHN encoding**: EUC-KR (code page 949) - never use UTF-8 for SHN string operations
- **SQLite integer handling**: Use `unchecked()` casts for unsigned types (UInt32, UInt64) since SQLite stores everything as Int64
- **JsonElement**: All values from JSON-deserialized project files are `System.Text.Json.JsonElement`, not native types. Convert helpers must handle this.
- **Empty tables must round-trip**: Server exes may require tables to exist even with 0 rows. Parsers must preserve empty tables.
- **Internal classes** in Mimir.ShineTable use `InternalsVisibleTo` for test access.

## Common Pitfalls

1. **Checked overflow**: `Convert.ToUInt32(long)` throws on large values. Use `unchecked((uint)longVal)` instead.
2. **Korean strings**: EUC-KR is 2 bytes/char, UTF-8 is 3 bytes/char for Korean. SHN padded strings overflow if you measure in UTF-8.
3. **JsonElement in write path**: `Convert.ToByte(jsonElement)` doesn't work because JsonElement doesn't implement IConvertible. Must check `is JsonElement je` first and use je.GetXxx().
4. **ConvertFromSqlite**: Must handle `string`, `long`, and `double` value types separately - SQLite returns these native types.
5. **Parser column building**: TableFormatParser builds columns from #columntype/#columnname even with 0 records (not lazily on first record).

## Meta Instructions

- When the user says **"learn XYZ"**, add XYZ to this CLAUDE.md file (rephrased/prompt-engineered as appropriate for future context).
- Always keep `Tracker.md` (in the project root) open/consulted — it tracks ongoing work, decisions, and open questions.
- **Standard testing procedure**: Before every commit, run `dotnet test Mimir.sln` and confirm all tests pass — including the integration tests in `tests/Mimir.Integration.Tests/`. The synthetic integration tests (SyntheticRoundtripTests) exercise the full init-template → import → build pipeline with self-created SHN data and verify SHN bit-equality on roundtrip. They are the final gate before committing. Real-world integration tests (RealWorldRoundtripTests) require `MIMIR_SERVER_PATH` and `MIMIR_CLIENT_PATH` env vars and skip gracefully when not set.

## Data Flow

```
Server Files (SHN/txt) → import → JSON project (mimir.json + data/**/*.json)
JSON project → load into SQLite → query/edit → extract back to JSON → build → Server Files
```

## Test Strategy

- **Unit tests**: Per-project tests for parsers, models, SQL engine, etc. Use self-created test data only.
- **Synthetic integration tests** (`Mimir.Integration.Tests/SyntheticRoundtripTests`): Full pipeline tests using `ShnDataProvider.WriteAsync` to create temp SHN files, then running init-template → import → build in-process. Verifies metadata, env filtering, directory layout, data roundtrip, and SHN bit-equality. No external dependencies — runs in CI.
- **Real-world integration tests** (`Mimir.Integration.Tests/RealWorldRoundtripTests`): Uses `MIMIR_SERVER_PATH` and `MIMIR_CLIENT_PATH` env vars. Skips gracefully when not set. Verifies >1000 table imports, merged table metadata, directory structure, and SHN bit-identical roundtrip for single-env tables.
- **Pre-commit gate**: Always run `dotnet test Mimir.sln` before committing. All synthetic integration tests must pass. This is non-negotiable — integration tests are the final validation step.
- **test-project/**: Contains real imported data (gitignored). Can be regenerated via `import`. Data dirs: `data/shn/`, `data/shinetable/`, `data/configtable/`.
