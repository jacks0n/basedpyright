# feat: glob patterns and diagnostic-only overrides in executionEnvironments

## Summary

Adds glob/wildcard pattern support in `executionEnvironments[].root`, per-environment `typeCheckingMode`, and diagnostic-only overrides that decouple file matching from import resolution. This addresses the most-requested configuration limitation across basedpyright and upstream pyright communities.

**Key changes:**

- `root` field now accepts glob patterns (`*`, `**`, `?`) for matching files across multiple directories
- Glob-matched environments use the project root for import resolution, not the matched directory — solving the long-standing #668 problem
- Per-environment `typeCheckingMode` allows setting a base diagnostic preset (e.g. `"strict"`, `"basic"`) with individual rule overrides layered on top (#1638)
- JSON schema and configuration docs updated with examples

## Motivation

Users have been asking for a way to apply different diagnostic rules to subsets of their codebase without manually listing every path or breaking import resolution:

- **#668** — diagnostic rules without separate execution environments (import root coupling problem)
- **#1042** — scaling to many execution environments (168+ dirs in a university course repo)
- **#1217** — per-file-pattern rule overrides (`*_test.py`)
- **#307** — scope rules to specific parts of codebase
- **#1638** — `typeCheckingMode` per execution environment

Currently, `executionEnvironments` changes the import root for matched files. Users who just want different diagnostic rules (e.g. `reportPrivateUsage = false` for tests) get broken imports because the test directory becomes a separate root. This PR solves that by decoupling file matching from import resolution when glob patterns are used.

## Usage

Relax diagnostics for all test directories without breaking imports:

```json
{
    "executionEnvironments": [
        {
            "root": "**/tests",
            "typeCheckingMode": "basic",
            "reportPrivateUsage": false
        },
        {
            "root": "src"
        }
    ]
}
```

Test files matched by `**/tests` get relaxed diagnostic rules while still importing from the project root. No more listing every test directory individually.

Glob patterns also work in shared/extended configs — glob roots resolve relative to the project root, so a shared base config with `"root": "**/tests"` works correctly regardless of where the config file lives.

## Implementation

### Config parsing (`configOptions.ts`)

- Added `rootFileSpec` (FileSpec) and `isGlobRoot` (boolean) fields to `ExecutionEnvironment`
- Config parser detects glob characters in `root`, validates the pattern, and creates a `FileSpec` using existing `getFileSpec()` infrastructure
- Glob roots are resolved relative to `projectRoot` (not `configDirUri`) so they work correctly in extended configs
- Plain roots resolve relative to `configDirUri` as before (backward compatible)
- Added `isValidGlobPattern()` validation — rejects empty patterns and ambiguous `***`
- Added `typeCheckingMode` parsing in `_initExecutionEnvironmentFromJson`, positioned before individual override loops so overrides layer correctly

### File matching (`configOptions.ts`)

- Extracted `_fileMatchesEnvironment()` private helper from `findExecEnvironment()`
- Glob branch: `file.matchesRegex(env.rootFileSpec.regExp)`
- Plain branch: `file.startsWith(envRoot)` (unchanged, fast path preserved)

### Import resolution (`importResolver.ts`)

- Added `getEffectiveImportRoot()` helper — returns `projectRoot` for glob-root environments, `execEnv.root` for plain-root environments
- Updated all 12 import-resolution call sites to use the effective root
- Cache keys use effective root consistently across insert and lookup operations
- One deliberate exclusion: `isDefaultWorkspace(execEnv.root)` at L2814 — checks workspace type, not import resolution

### Background analysis (`backgroundAnalysisBase.ts`, `backgroundAnalysisProgram.ts`)

- Changed cross-thread execution environment lookup from `root.toString()` to array index
- Multiple glob environments can share the same `wildcardRoot`, so string-based identity would be ambiguous

### Schema (`pyrightconfig.schema.json`)

- Updated `root` field title and description documenting glob support and wildcard syntax
- Added `examples` array for IDE autocomplete: `["src", "**/tests", "src/*/utils"]`
- Added `typeCheckingMode` via `$ref` to existing definition

### Documentation (`docs/configuration/config-files.md`)

- Added "Glob Patterns in Root" subsection with wildcard character reference and examples
- Added "Diagnostic-Only Overrides" subsection showing the primary use case
- Added "First-Match-Wins Ordering" subsection with concrete path matching examples
- Added "typeCheckingMode in Execution Environments" subsection with layering example
- Updated sample JSON and TOML configs
- All basedpyright-exclusive features marked with admonitions

## Test plan

**Config parsing (9 tests):**
- Plain root backward compatibility — `isGlobRoot=false`, `rootFileSpec=undefined`
- Glob detection for `**`, `*`, `?` patterns — FileSpec creation verified
- Invalid pattern (`***`) produces clear error
- Serialization round-trip preserves glob fields
- Mixed plain and glob roots in same config

**Glob matching (7 tests):**
- `**` matches files at any directory depth
- `*` matches single path segment only
- `?` matches single character
- First-match-wins with plain root before glob root
- First-match-wins with glob root before plain root
- Multiple glob roots — first match wins
- Default environment fallback

**Diagnostic-only overrides (6 tests):**
- `getEffectiveImportRoot` returns `projectRoot` for glob-root
- `getEffectiveImportRoot` returns `execEnv.root` for plain-root
- Glob-root with diagnostic overrides uses `projectRoot`
- Integration: glob-matched file resolves imports from project root
- Integration: plain-root resolves imports from own root (regression)
- Integration: `getModuleNameForImport` with glob-root

**typeCheckingMode in EEs (11 tests):**
- All 6 modes apply correct base rule sets
- Individual overrides layer on top of preset
- EE without mode inherits top-level rules
- Invalid mode values produce errors
- Glob-root + typeCheckingMode + overrides work together
- Multiple EEs with different modes are independent

**Schema validation (2 tests):**
- DiagnosticRule matches pyrightconfig.schema.json
- DiagnosticRule matches package.json

**Total: 105 tests (70 pre-existing + 35 new), all passing**
