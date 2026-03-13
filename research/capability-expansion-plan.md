# UnrealClaude Capability Expansion Plan

> **Purpose**: Strategic roadmap for evolving UnrealClaude from a safety-constrained MCP server
> into an unrestricted, maximally capable co-mind interface for Unreal Engine.
>
> **Date**: 2025-03-12
> **Companion doc**: `safety-filter-audit.md` (restriction removal details)
> **Landscape research**: `../unreal-mcp/research/unreal-mcp-landscape.md` (copied below where relevant)

---

## Strategic Vision

UnrealClaude should be the **most capable MCP server for Unreal Engine in existence**.
The current implementation is already architecturally superior to all competitors (HTTP server
via UE's built-in HttpServerModule, Node.js MCP bridge, async task queue, per-tool C++ handlers).
But it's held back by artificial safety restrictions designed for untrusted operators.

In co-mind mode, Claude and the operator share intent and accountability. The right safety model
is **visibility and undo**, not **prevention and blocking**.

### Design Principles

1. **No artificial limits** — If Unreal Engine allows it, UnrealClaude should allow it
2. **Warn, don't block** — Log warnings for potentially destructive operations; never silently refuse
3. **Undo support** — Wrap operations in UE transactions so mistakes are recoverable
4. **Visibility** — Comprehensive logging so the operator always knows what happened
5. **Escape hatch** — When a dedicated tool doesn't exist, script execution provides full access

---

## Current State: What UnrealClaude Already Has

### Architecture (Superior to Competitors)
- **C++ Plugin**: UE 5.7, EditorSubsystem-based, HTTP server on port 3000
- **MCP Bridge**: Node.js wrapper (vs competitors' Python/TypeScript)
- **Async Queue**: Task submission with status polling and 10-min timeout
- **Per-Tool C++ Handlers**: Clean separation in `Private/MCP/Tools/MCPTool_*.cpp`

### Existing Tools (~35)
| Category | Tools |
|----------|-------|
| **Actors** | spawn, delete, move, set_property, get_level_actors |
| **Assets** | search, dependencies, referencers, generic asset ops |
| **Blueprints** | query (list/inspect/get_graph), modify (create/variables/functions/nodes/wiring) |
| **Animation** | anim_blueprint_modify (state machines, transitions, conditions, animation assignment) |
| **Materials** | create_material_instance, set_parameters, assign to mesh/actor |
| **Characters** | list, inspect, movement params, components, character_data (DataAssets/DataTables) |
| **Input** | Enhanced Input (create actions, mapping contexts, triggers, modifiers) |
| **Editor** | open_level, capture_viewport, run_console_command, get_output_log |
| **Scripts** | execute_script (cpp/python/console/editor_utility), cleanup, history |
| **Tasks** | submit, status, result, list, cancel (async execution) |
| **Context** | get_ue_context (API documentation retrieval) |

---

## Phase 1: Unrestrict (Immediate — No New Features)

Remove artificial restrictions per `safety-filter-audit.md`.

### 1.1 Console Commands — Full Access
**File**: `MCPParamValidator.cpp:9-52`
- **Remove** the entire `GetBlockedConsoleCommands()` blocklist
- **Keep** a warning log for genuinely destructive commands (`quit`, `exit`, `crash`)
- **Result**: `r.Shadow.Virtual.Enable`, `stat gpu`, `gc.*` all work immediately

### 1.2 Blueprint Path Access — Engine & Script
**File**: `MCPParamValidator.cpp:278-318`
- **Remove** `/Engine/` and `/Script/` path blocks
- **Keep** path traversal (`..`) prevention
- **Result**: Can inspect engine Blueprint classes, understand parent hierarchies

### 1.3 Pagination — Raise Caps
**Files**: All `MCPTool_*.cpp` files with `FMath::Clamp(..., 1, 1000)`
- **Raise** hard max from 1,000 to 10,000
- **Keep** defaults at 25-100 (sensible for token efficiency)
- **Result**: Can query full actor lists, complete asset registries

### 1.4 Script Execution — Trust Mode
**File**: `ScriptExecutionManager.cpp:62`
- **Add** a "trust mode" setting that bypasses the permission dialog
- **Enable** via project setting or config flag
- **Result**: Autonomous script execution without manual intervention

### 1.5 Numeric Bounds — Widen
**File**: `MCPParamValidator.cpp:213-236`
- **Keep** NaN/Infinity rejection (causes real crashes)
- **Raise** coordinate bound from 1e10 to 1e15 (solar system scale)
- **Widen** movement clamps by 10x (game design is creative)

### 1.6 String Limits — Double
**File**: `UnrealClaudeConstants.h`
- **Double** all string length limits
- **Result**: Longer function names, complex property paths, bigger console commands

### 1.7 Timeouts & Sizes — Increase
- Game thread timeout: 30s → 120s
- HTTP request body: 1 MB → 10 MB

---

## Phase 2: Expand Core (New Capabilities)

### 2.1 PIE (Play In Editor) Control
**Priority**: Critical — enables test-driven workflows
- `start_pie` — Start Play-In-Editor session
- `stop_pie` — Stop current PIE session
- `pause_pie` / `resume_pie` — Pause/resume gameplay
- `get_pie_state` — Query if PIE is running, paused, etc.
- **Implementation**: `ULevelEditorSubsystem` has `EditorPlaySimulate()`, `EditorRequestEndPlay()`
- **Value**: Claude can test gameplay changes immediately, observe results, iterate

### 2.2 Undo/Redo System
**Priority**: Critical — makes all operations reversible
- `begin_transaction(description)` — Start undo group
- `end_transaction()` — Close undo group
- `undo()` / `redo()` — Undo/redo last transaction
- **Implementation**: `FScopedTransaction`, `GEditor->UndoTransaction()`
- **Value**: Wrap multi-step MCP operations in single undo actions

### 2.3 Project Settings & CVars
**Priority**: High — complements console commands
- `get_cvar(name)` — Read CVar value with metadata (type, range, description)
- `set_cvar(name, value)` — Set CVar value
- `list_cvars(filter)` — Search/list available CVars
- `get_project_setting(category, key)` — Read project settings
- `set_project_setting(category, key, value)` — Write project settings
- **Implementation**: `IConsoleManager`, `GConfig`

### 2.4 Editor Selection & Focus
**Priority**: High — enables interactive workflows
- `select_actors(names)` — Select actors in viewport
- `get_selected_actors()` — Query current selection
- `focus_viewport_on(actor_name)` — Focus camera on actor
- `set_viewport_camera(location, rotation)` — Direct camera control
- **Implementation**: `EditorActorSubsystem`, `FLevelEditorViewportClient`

### 2.5 Content Validation
**Priority**: Medium — quality assurance
- `validate_assets(paths)` — Run UE asset validation
- `check_map_errors()` — Run Map Check
- `get_compilation_errors()` — Blueprint compilation errors
- **Implementation**: `EditorValidatorSubsystem`, `FMessageLog`

---

## Phase 3: Content Creation Tools

### 3.1 Material Graph Editing
- `create_material(name, domain)` — Create full material (not just instances)
- `add_material_expression(material, type, position)` — Add expression nodes
- `connect_material_expressions(source, output, target, input)` — Wire nodes
- `set_material_expression_value(node, param, value)` — Set expression parameters
- **Implementation**: `MaterialEditingLibrary`, `UMaterialExpression` subclasses

### 3.2 Sequencer Automation
- `create_level_sequence(name)` — Create new sequence
- `add_sequence_track(sequence, actor, track_type)` — Add tracks
- `set_sequence_keyframe(track, time, value)` — Set keyframes
- `play_sequence(name)` — Preview sequence
- **Implementation**: `SequencerTools`, `UMovieScene*` classes

### 3.3 Data Tables
- `create_data_table(name, struct_type)` — Create DataTable
- `add_data_table_row(table, row_name, values)` — Add rows
- `query_data_table(table, filter)` — Query rows
- `update_data_table_row(table, row_name, values)` — Modify rows
- **Implementation**: `UDataTable` API

### 3.4 Asset Import
- `import_asset(source_path, destination, options)` — Import external files
- `reimport_asset(asset_path)` — Reimport with updated source
- `bulk_import(mappings)` — Batch import
- **Implementation**: `AssetImportTask`, `AssetToolsHelpers`

### 3.5 Niagara Particle Systems
- `create_niagara_system(name)` — Create particle system
- `add_niagara_emitter(system, emitter_template)` — Add emitters
- `set_niagara_parameter(system, module, param, value)` — Configure parameters
- **Implementation**: Niagara module APIs (complex but documented)

---

## Phase 4: Production Pipeline

### 4.1 Build & Cook
- `build_lighting(quality)` — Build lighting
- `cook_content(platform, maps)` — Cook content for platform
- `package_project(platform, config)` — Full packaging
- **Implementation**: `FEditorBuildUtils`, commandlets

### 4.2 Source Control
- `checkout_files(paths)` — Check out for editing
- `submit_files(paths, description)` — Submit changelist
- `revert_files(paths)` — Revert changes
- `get_file_status(paths)` — Query SC status
- **Implementation**: `ISourceControlModule`

### 4.3 Editor Notifications & Events
- `subscribe_to_events(event_types)` — Register for editor events
- `get_pending_notifications()` — Poll for events
- Event types: asset_saved, compilation_error, pie_started, pie_stopped, actor_deleted, etc.
- **Implementation**: UE delegate system, custom event aggregator

### 4.4 World Partition & Streaming
- `get_world_partition_info()` — Query partition grid
- `load_streaming_level(path)` — Load sub-level
- `unload_streaming_level(path)` — Unload sub-level
- **Implementation**: `UWorldPartition`, `ULevelStreaming`

---

## Phase 5: Differentiators (What Nobody Else Has)

### 5.1 Visual Feedback Loop
Extend `capture_viewport` into a continuous feedback system:
- Capture after every modification
- Include before/after comparison
- Enable Claude to "see" the results of its changes
- Annotate captures with actor labels/bounds

### 5.2 Intelligent Batch Operations
Higher-level compound tools:
- `populate_scene(description)` — AI-driven scene population
- `setup_character(config)` — Complete character setup (mesh, animation, input, movement)
- `create_gameplay_loop(spec)` — End-to-end gameplay system creation

### 5.3 Blueprint Debugging
- `set_blueprint_breakpoint(blueprint, node)` — Set breakpoints
- `get_blueprint_watch_values()` — Read watched variables
- `step_blueprint_execution()` — Step through Blueprint execution

### 5.4 Performance Profiling
- `start_profiling(duration)` — Capture performance data
- `get_profiling_results()` — Frame times, draw calls, memory
- `identify_bottlenecks()` — Automated analysis
- **Implementation**: `FStatsManager`, Insights API

---

## Competitive Landscape

### How We Compare After Full Implementation

| Capability | UnrealClaude (Current) | UnrealClaude (Planned) | ChiR24 (Best Competitor) |
|---|---|---|---|
| Actor CRUD | Yes | Yes | Yes |
| Blueprint Editing | Full (nodes/wiring) | Full | Basic |
| Animation Blueprints | Full | Full | Basic |
| Materials | Instances only | Full graph editing | Yes |
| Console Commands | Blocked (`r.*`, `gc.*`) | Unrestricted | Pattern-filtered |
| Script Execution | Yes (with dialog) | Yes (trust mode) | No |
| Enhanced Input | Yes | Yes | Yes |
| Characters | Yes + DataAssets | Yes + DataAssets | Yes |
| Async Tasks | Yes | Yes | No |
| Sequencer | No | Yes | Yes |
| PIE Control | No | Yes | No |
| Undo Support | No | Yes | No |
| Niagara | No | Yes | No |
| Build/Cook | No | Yes | No |
| Source Control | No | Yes | No |
| Profiling | No | Yes | Yes |
| Visual Feedback | Screenshot | Continuous loop | No |
| Landscape | No | Phase 5 | No |
| World Partition | No | Phase 4 | No |
| Asset Import | No | Phase 3 | No |

### Key Differentiators
1. **Script execution** — No other MCP server lets the AI write and run arbitrary C++/Python in UE
2. **Async task queue** — No other server supports background execution with polling
3. **Animation Blueprint** — Most comprehensive ABP manipulation in the ecosystem
4. **Trust mode** — Only server designed for co-mind operation (no safety theater)

---

## Implementation Notes

### Branching Strategy
- Create `feature/unrestrict` branch for Phase 1
- Merge to `main` after testing
- Phase 2+ can be parallel branches

### Testing Approach
- Test each unrestriction against a sample UE project
- Verify no actual crashes (NaN/Infinity rejection must stay)
- Verify all `r.*` commands work after unblocking
- Verify `/Engine/` Blueprint inspection works
- Verify large actor queries (>1000) work

### Risk Assessment
- **Low risk**: Removing console command blocks, raising pagination caps
- **Medium risk**: Trust mode for scripts (user must understand implications)
- **No risk**: Raising string limits, timeouts, buffer sizes
