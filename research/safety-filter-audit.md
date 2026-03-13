# UnrealClaude Safety Filter Audit & Removal Strategy

> **Purpose**: Comprehensive audit of all artificial restrictions in UnrealClaude MCP server,
> with a categorized removal plan for evolving toward unrestricted co-mind operation.
>
> **Date**: 2025-03-12
> **Status**: Research Complete — Ready for Implementation

---

## Executive Summary

UnrealClaude has **12 categories of restrictions** across 6 key files. Most are defensive
measures designed for untrusted operators — not for a co-mind setup where Claude and the
user operate as a collaborative team with full intent and accountability.

The restrictions range from **genuinely protective** (NaN/Infinity rejection) to **pure security
theater** (blocking `r.` console commands while exposing arbitrary script execution). This
document categorizes each restriction and recommends a disposition.

---

## Restriction Categories

### TIER 1: REMOVE — Security Theater / Actively Harmful

These restrictions block legitimate workflows while being trivially bypassed via other tools.

#### 1.1 Console Command Blocklist
- **File**: `Source/UnrealClaude/Private/MCP/MCPParamValidator.cpp`, lines 9-52
- **What**: Blocks 22+ commands by exact/prefix match, including ALL `r.*` and `gc.*` CVars
- **Impact**: Cannot modify rendering settings (`r.Shadow.Virtual.Enable`), garbage collection, etc.
- **Why Remove**: `unreal_execute_script` with `script_type: 'python'` can execute `unreal.SystemLibrary.execute_console_command()` anyway. The filter only forces users through a slower, more complex path.
- **Blocked commands**:
  - `quit`, `exit`, `crash`, `forcecrash`, `debug crash` — engine termination
  - `forcegc`, `gc.*` — garbage collection
  - `mem`, `memreport` — memory diagnostics
  - `obj` — object introspection
  - `exec` — execute file
  - `savepackage`, `deletepackage` — package operations
  - `net`, `admin` — network/admin
  - `shutdown`, `restartlevel` — level management
  - `open`, `servertravel` — map loading
  - `toggledebugcamera`, `enablecheats` — debug tools
  - `stat slow` — slow stat command
  - **`r.*` (ALL rendering CVars)** — the main pain point
- **Recommendation**: **Remove entirely.** Replace with a simple warning log for destructive commands (`quit`, `exit`, `crash`) rather than blocking them.

#### 1.2 Blueprint Path Restrictions
- **File**: `MCPParamValidator.cpp`, lines 278-318
- **What**: Blocks access to `/Engine/` and `/Script/` prefixed paths
- **Impact**: Cannot inspect or reference engine Blueprints, cannot work with script-package assets
- **Why Remove**: Read-only inspection of engine assets is essential for understanding parent classes, available functions, etc. The restriction prevents legitimate workflows.
- **Recommendation**: **Remove entirely.** Engine assets are read-only by nature — you can't accidentally corrupt them via MCP.

#### 1.3 Script Permission Dialog
- **File**: `ScriptExecutionManager.cpp`, line 62
- **What**: Pops up a dialog in Unreal Editor requiring manual user approval before each script runs
- **Impact**: Breaks autonomous workflow — every script execution requires the user to alt-tab to Unreal and click "Allow"
- **Why Remove**: In co-mind mode, Claude's actions are authorized by the conversation. The MCP client (Claude Code) already has its own permission system.
- **Recommendation**: **Remove the dialog** or add a config flag to disable it. Add a "trust mode" setting.

---

### TIER 2: RELAX — Overly Conservative Defaults

These have reasonable intent but are set too aggressively for power users.

#### 2.1 Pagination Hard Caps
- **Files**: All query tool `.cpp` files in `Private/MCP/Tools/`
- **What**: Hard cap of 1000 results, defaults of 25-100
- **Current limits**:
  | Tool | Default | Hard Max |
  |------|---------|----------|
  | get_level_actors | 25 | 1,000 |
  | asset_search | 25 | 1,000 |
  | blueprint_query | 25 | 1,000 |
  | character tools | 100 | 1,000 |
  | get_output_log | 100 | 1,000 |
- **Recommendation**: **Raise hard max to 10,000.** Keep defaults at 25-100 (sensible for most queries). Let the caller decide when they need more.

#### 2.2 String Length Limits
- **File**: `UnrealClaudeConstants.h`
- **Current limits**:
  | Field | Current Max |
  |-------|-------------|
  | Actor names | 256 |
  | Property paths | 512 |
  | Class paths | 1,024 |
  | Console commands | 2,048 |
  | Blueprint paths | 512 |
  | Variable names | 128 |
  | Function names | 128 |
- **Recommendation**: **Double all limits.** 128 chars for function names is tight for generated code. Console command limit of 2KB is fine.

#### 2.3 HTTP Request Size
- **File**: `UnrealClaudeConstants.h`, line 168
- **Current**: 1 MB
- **Recommendation**: **Increase to 10 MB.** Script content and batch operations can easily exceed 1 MB.

#### 2.4 Game Thread Timeout
- **File**: `UnrealClaudeConstants.h`, line 159
- **Current**: 30 seconds
- **Recommendation**: **Increase to 120 seconds.** Complex Blueprint modifications, asset searches over large projects, and compilation can exceed 30s.

#### 2.5 Movement Value Clamping
- **File**: `MCPTool_Character.cpp`
- **Current clamps**:
  - MaxWalkSpeed: 0-10,000
  - MaxAcceleration: 0-100,000
  - JumpZVelocity: 0-10,000
  - etc.
- **Recommendation**: **Widen by 10x or remove.** Game design is creative — artificial caps prevent experimentation (e.g., superhero games need jump velocities > 10,000).

---

### TIER 3: KEEP BUT SOFTEN — Genuine Protection

These prevent actual crashes or data corruption but could be made less aggressive.

#### 3.1 Numeric Bounds
- **File**: `MCPParamValidator.cpp`, lines 213-236
- **What**: Rejects NaN, Infinity, and values > 1e10
- **Recommendation**: **Keep NaN/Infinity rejection** (these cause real crashes). **Raise coordinate bound to 1e15** (solar system scale games exist).

#### 3.2 Dangerous Characters in Names
- **File**: `UnrealClaudeConstants.h`, line 105
- **What**: Blocks `<>|&;\`$(){}[]!*?~` in actor names, property paths, etc.
- **Recommendation**: **Keep for actor names** (Unreal itself doesn't support these). **Relax for property values** where special characters may be legitimate content (e.g., text fields, descriptions).

#### 3.3 Command Chaining Prevention
- **File**: `MCPParamValidator.cpp`, lines 196-208
- **What**: Blocks `;`, `|`, `&&`, backticks, `$(`, `${` in console commands
- **Recommendation**: **Keep `;` blocking for console commands** (prevents accidental multi-command injection). **Remove for script content** (scripts legitimately contain all these characters).

#### 3.4 Path Traversal Prevention
- **What**: Blocks `..` in paths
- **Recommendation**: **Keep.** Path traversal is a genuine security concern even in trusted setups.

---

## Implementation Plan

### Phase 1: Immediate Unblocking (High Impact, Low Risk)
1. Remove `r.*` and `gc.*` from console command blocklist
2. Remove `/Engine/` and `/Script/` path blocks for read operations
3. Raise pagination caps to 10,000
4. Add "trust mode" config flag to skip script permission dialog

### Phase 2: Relaxation (Medium Impact)
5. Double string length limits
6. Increase HTTP request size to 10 MB
7. Increase game thread timeout to 120s
8. Widen movement value clamps by 10x

### Phase 3: Cleanup (Polish)
9. Replace remaining blocklist with warning-only logging
10. Add configurable restriction profiles (strict/relaxed/unrestricted)
11. Make all limits configurable via project settings or config file

---

## Files Requiring Modification

| File | Changes |
|------|---------|
| `Source/UnrealClaude/Private/MCP/MCPParamValidator.cpp` | Remove/modify blocklists, path restrictions |
| `Source/UnrealClaude/Private/MCP/MCPParamValidator.h` | Update method signatures if needed |
| `Source/UnrealClaude/Public/UnrealClaudeConstants.h` | Update all constant values |
| `Source/UnrealClaude/Private/ScriptExecutionManager.cpp` | Add trust mode bypass |
| `Source/UnrealClaude/Private/MCP/Tools/MCPTool_GetLevelActors.cpp` | Raise cap |
| `Source/UnrealClaude/Private/MCP/Tools/MCPTool_AssetSearch.cpp` | Raise cap |
| `Source/UnrealClaude/Private/MCP/Tools/MCPTool_Character.cpp` | Widen clamps |
| All other `MCPTool_*.cpp` files with `FMath::Clamp(..., 1, 1000)` | Raise caps |

---

## Beyond Restriction Removal: Capability Expansion

With restrictions removed, the next frontier is **adding capabilities** that UnrealClaude
doesn't yet have. See `capability-expansion-plan.md` for the full roadmap. Key targets:

- **Niagara particle system** creation/modification
- **Sequencer** control (cinematic authoring)
- **Level streaming** management
- **PIE (Play In Editor)** start/stop/pause control
- **Landscape/terrain** tools
- **Audio** system integration
- **Data Layer / World Partition** management
- **Live Coding** integration (compile C++ without restarting editor)
- **Source control** operations (Perforce/Git)
- **Editor preferences & project settings** modification
