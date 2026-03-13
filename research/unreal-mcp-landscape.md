# Unreal Engine MCP Landscape Research

> Research date: March 2026
> Purpose: Map the current state of MCP integrations for Unreal Engine, identify capabilities, and find expansion opportunities.

---

## 1. Existing Unreal MCP Projects

The Unreal MCP ecosystem has grown rapidly since late 2025. There are at least 7 significant open-source implementations, each taking a different architectural approach.

### 1.1 chongdashu/unreal-mcp

- **Architecture:** C++ plugin (52.4% of codebase) + Python MCP server (46.7%) using FastMCP
- **Communication:** TCP socket on port 55557. Python server connects to C++ plugin's TCP server. Connection-per-command pattern (Unreal closes connection after each response).
- **Tools:** Actor management (spawn, delete, transform, properties), Blueprint creation and component configuration, Blueprint node graph manipulation (events, functions, variables, connections), Editor viewport control (focus, camera)
- **Notable:** This is our own project's upstream architecture. Clean separation between MCP protocol handling (Python) and engine operations (C++).

### 1.2 ChiR24/Unreal_mcp

- **Architecture:** TypeScript MCP server + C++ Automation Bridge plugin
- **Communication:** Action-based dispatch via the bridge on configurable host/port (default 127.0.0.1:8091). Optional LAN access.
- **Tools (36 total):** The most comprehensive tool set in the ecosystem:
  - **Core:** `manage_asset`, `control_actor`, `control_editor`, `manage_level`, `system_control`, `inspect`, `manage_pipeline`, `manage_tools`
  - **World Building:** `manage_lighting`, `manage_level_structure`, `manage_volumes`, `manage_navigation`, `build_environment`, `manage_splines`
  - **Animation/Physics:** `animation_physics`, `manage_skeleton`, `manage_geometry`
  - **Visuals/Effects:** `manage_effect`, `manage_material_authoring`, `manage_texture`, `manage_blueprint`, `manage_sequence`, `manage_performance`
  - **Audio/Input:** `manage_audio`, `manage_input`
  - **Gameplay Systems:** `manage_behavior_tree`, `manage_ai`, `manage_gas` (Gameplay Ability System), `manage_character`, `manage_combat`, `manage_inventory`, `manage_interaction`, `manage_widget_authoring`
  - **Networking:** `manage_networking`, `manage_game_framework`, `manage_sessions`
- **Notable features:** Dynamic type discovery via runtime introspection, command safety with pattern-based validation blocking dangerous console commands, 10-second asset caching TTL, per-IP rate limiting (60 req/min) on metrics endpoints, graceful degradation (server starts without Unreal connection).

### 1.3 flopperam/unreal-engine-mcp

- **Architecture:** Python MCP server + C++ plugin
- **Communication:** TCP sockets with automatic reconnection
- **Tools:** Blueprint visual scripting (23+ node types including Branch, Switch, VariableGet/Set, SpawnActor, CallFunction), world building (procedural town/house/maze generation with recursive backtracking), physics and materials, actor management
- **Notable:** Focuses on world-building use cases. Can generate entire architectural structures and solvable mazes.

### 1.4 VedantRGosavi/UE5-MCP

- **Architecture:** C++ plugin + Python server
- **Notable:** Has detailed architecture documentation and research docs in the repo. Focuses on bridging AI assistants with UE5.

### 1.5 gingerol/vhcilab-unreal-engine-mcp

- **Focus:** Natural language scene building. Create objects, lights, and structures using text commands through Claude Code.
- **Notable:** Academic/research-oriented (VHCI Lab).

### 1.6 ayeletstudioindia/unreal-analyzer-mcp

- **Architecture:** TypeScript MCP server using Tree-sitter C++ parser
- **Tools:** Class analysis, class hierarchy mapping, reference finding, code search, pattern detection, API documentation query, subsystem analysis
- **Notable:** This is fundamentally different -- it's a *code analysis* tool, not an engine control tool. Analyzes Unreal C++ source code rather than controlling a running editor. Educational focus with pattern recognition and best-practice recommendations. Works on any C++ codebase, not just Unreal.

### 1.7 AlexKissiJr/UnrealMCP

- **Listed on:** mcpservers.org, lobehub.com
- **Focus:** Plugin-based MCP integration for Unreal Engine

### 1.8 Epic Games' Position

On the Epic Developer Community Forums (thread from late 2025, updated Feb 2026), Shaun Comly from Epic stated: *"We are actively investigating MCP and are very interested in it but do not yet have any concrete plans around shipping a MCP server in the near future."* As of February 2026: *"Nothing official, no."*

This means the community is ahead of Epic on MCP, and there's no immediate risk of an official implementation making community work redundant.

---

## 2. Unreal Engine Python API (`unreal` Module)

The `unreal` Python module is Unreal's official scripting interface. It reflects everything exposed from C++ to Blueprints, dynamically -- enabling new plugins automatically exposes their Blueprint-accessible APIs to Python.

### 2.1 Core Capabilities

| Category | What You Can Do |
|---|---|
| **Asset Management** | `load_asset()`, `load_class()`, `load_object()`, `load_package()`, asset import via `AssetImportTask`, LOD generation, bulk operations |
| **Asset Tools** | Create, duplicate, rename, delete assets via `AssetToolsHelpers.get_asset_tools()` |
| **Level Editing** | Load/save levels, spawn/delete/transform actors, query world state, manipulate actor properties |
| **Material Editing** | Create materials via `MaterialFactoryNew`, create material expressions, connect expression nodes programmatically. API is functional but awkward -- requires understanding the expression node connection model. |
| **Sequencer** | `SequencerTools` class, add tracks and sections, manipulate keyframes. Transform tracks (`MovieScene3DTransformTrack`), camera cuts, etc. Python Sequencer cookbook exists in community docs. |
| **Blueprint Interaction** | Create Blueprint classes, add components, set properties, compile. Full node graph manipulation is possible but complex. |
| **Editor Subsystems** | Access via `unreal.get_editor_subsystem()` -- see Section 5 below |
| **Static Mesh Operations** | LOD generation, collision setup, mesh operations |
| **Texture/Import** | Bulk import, format conversion, texture settings |
| **UI (Slate/UMG)** | Limited -- can create Editor Utility Widgets but not full Slate UI from Python |
| **Logging** | `unreal.log()`, `unreal.log_error()`, `unreal.log_warning()` |
| **Decorators** | `@unreal.ufunction()`, `@unreal.uproperty()` for defining UFunction/UProperty from Python |

### 2.2 Key Boundaries and Limitations

1. **Editor-only:** Python is only available in the Unreal Editor. Not available during Play-In-Editor, Standalone Game, or packaged builds. This is the most fundamental limitation -- Python cannot be used as a gameplay scripting language.

2. **Experimental status:** Epic does not guarantee backward compatibility. APIs may change or be removed between engine versions.

3. **Sequencer gaps:** While you can create level sequences, add tracks, and manipulate sections, you cannot programmatically assign animation clips to skeletal mesh animation tracks. Keyframe editing is possible but the API is unintuitive.

4. **Material editing quirks:** The expression-based material API works but requires understanding internal node connection semantics. Creating materials involves "strange intermediate objects that have to be placated" (community assessment).

5. **No runtime execution:** Cannot run Python scripts during gameplay. For MCP purposes this is acceptable since we're controlling the editor, not gameplay.

6. **Thread safety:** Python execution must happen on the game thread. Long operations can freeze the editor.

7. **Blueprint node graph:** While you can create and connect nodes, the API for visual scripting graph manipulation is complex and not all node types are equally well-supported.

---

## 3. Unreal Console Commands

Console commands are accessible via `ExecuteConsoleCommand()` in C++ or through the Remote Control API.

### 3.1 Command Categories

| Prefix/Category | Controls | Examples |
|---|---|---|
| `r.` | Renderer settings | `r.ScreenPercentage`, `r.Shadow.MaxResolution`, `r.Tonemapper.Quality` |
| `stat` | Statistics display | `stat fps`, `stat unit`, `stat memory`, `stat scenerendering` |
| `show` | Viewport visualization | `show collision`, `show bounds`, `show navigation` |
| `fx.` | Particle/Niagara FX | `fx.Niagara.QualityLevel` |
| `t.` | Timer/tick settings | `t.MaxFPS`, `t.OverrideFPS` |
| `p.` | Physics settings | `p.Gravity`, `p.MaxPhysicsDeltaTime` |
| `sg.` | Scalability groups | `sg.ShadowQuality`, `sg.TextureQuality`, `sg.PostProcessQuality` |
| `au.` | Audio settings | `au.MaxChannels` |
| `ai.` | AI/navigation settings | `ai.LogNavigation` |
| `net.` | Networking | `net.MaxClientRate` |
| `gc.` | Garbage collection | `gc.TimeBetweenPurgingPendingKillObjects` |
| General | Engine commands | `dumpconsolecommands`, `obj list`, `mem`, `exit`, `quit`, `restartlevel` |

### 3.2 Dangerous vs. Safe Commands

**Genuinely dangerous (should be blocked or gated):**
- `exit` / `quit` -- Shuts down the editor entirely
- `gc.ForceGarbageCollection` -- Can cause hitches; may corrupt references if called at wrong time
- `obj gc` -- Forces garbage collection
- `killall` -- Destroys all actors of a class
- `SET` command -- Can modify any property on any object, including engine-critical ones. The most powerful and dangerous command.
- `slomo 0` -- Stops time entirely, can make editor unresponsive
- `disconnect` -- Disconnects from server (multiplayer context)
- Any command that modifies .ini files or project settings permanently

**Safe for configuration (ideal for MCP exposure):**
- `r.` renderer tweaks (visual quality, not destructive)
- `stat` commands (read-only diagnostics)
- `show` commands (viewport visualization toggles)
- `t.MaxFPS` and similar performance caps
- `sg.` scalability commands (quality presets)

### 3.3 MCP Implications

ChiR24/Unreal_mcp implements pattern-based command validation to block dangerous console commands. This is a good practice. A whitelist approach (allow known-safe prefixes) is safer than a blacklist approach (block known-dangerous commands).

---

## 4. Unreal Remote Control Plugin

Epic's official Web Remote Control plugin runs an HTTP server inside the Unreal Editor on port 30010. It supports both HTTP requests and WebSocket connections.

### 4.1 HTTP Endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/remote/info` | List all available HTTP routes |
| PUT | `/remote/object/property` | Get/set property values on any UObject in memory (actors, assets) |
| PUT | `/remote/object/call` | Call any UFunction on a UObject |
| GET | `/remote/preset` | List all Remote Control Presets |
| GET | `/remote/preset/{preset_name}` | Get preset details and metadata |
| PUT | `/remote/preset/{preset_name}/function/{function_name}` | Call a function exposed via preset |
| PUT | `/remote/preset/{preset_name}/property/{property_name}` | Get/set a preset-exposed property |
| PUT | `/remote/preset/{preset_name}/metadata/{key}` | Create/update metadata |
| DELETE | `/remote/preset/{preset_name}/metadata/{key}` | Delete metadata |
| PUT | `/remote/search/assets` | Search for assets |
| PUT | `/remote/object/describe` | Get type information about a UObject |
| PUT | `/remote/batch` | Execute multiple requests in a single call |
| POST | `/remote/object/transaction` | Begin/end transactions for undo support |

### 4.2 Key Capabilities

- **Object property access:** Read/write any property on any UObject in memory. This includes actors in the level, assets, subsystems -- anything.
- **Function calls:** Invoke any UFunction on any UObject. This is extremely powerful -- essentially arbitrary function execution on any reflected object.
- **Asset search:** Search the asset registry with filters.
- **Batch operations:** Execute multiple requests atomically.
- **Transaction support:** Wrap operations in undo transactions.
- **WebSocket support:** Real-time property monitoring and event streaming.
- **Preset system:** Curate which properties/functions are exposed for controlled access.

### 4.3 What It Cannot Do

- **Execute arbitrary Python:** The Remote Control API doesn't have a "run Python script" endpoint. You can only call exposed UFunctions and access UProperties.
- **Blueprint graph manipulation:** No endpoints for modifying Blueprint node graphs.
- **File system operations:** No direct file creation/manipulation.
- **Build/compile operations:** Cannot trigger builds or compilations.

### 4.4 MCP Relevance

The Remote Control API is a viable alternative transport to raw TCP sockets. It provides structured, documented endpoints with built-in safety (property/function-level access control via presets). However, it's less flexible than a custom C++ plugin that can execute arbitrary engine operations. The custom plugin approach (used by most MCP servers) provides broader capability at the cost of implementing safety controls yourself.

An interesting hybrid approach: use the Remote Control API for property access and function calls (well-tested, maintained by Epic) while using a custom plugin for operations the Remote Control API doesn't support (Blueprint graph manipulation, Python execution, build commands).

---

## 5. Unreal Editor Subsystems

Editor subsystems are automatically instanced classes with managed lifetimes. They provide clean extension points accessible from both C++ and Python via `unreal.get_editor_subsystem()`.

### 5.1 Key Subsystems

| Subsystem | Capabilities |
|---|---|
| **EditorActorSubsystem** | Actor selection, duplication, conversion. `select_all_actors()`, `duplicate_selected_actors()`, `get_selected_level_actors()`, `set_actor_selection_state()` |
| **AssetEditorSubsystem** | Open/close asset editors. `open_editor_for_assets()`, `close_all_editors_for_asset()` |
| **LevelEditorSubsystem** | Level editing operations. `editor_get_game_view()`, `editor_play_simulate()`, `editor_request_end_play()`, `eject_pilot_level_actor()`, viewport control |
| **UnrealEditorSubsystem** | General editor world access. `get_editor_world()`, `get_game_world()` |
| **LayersSubsystem** | Layer management for organizing level content |
| **ImportSubsystem** | Asset import operations and callbacks |
| **EditorValidatorSubsystem** | Asset validation and data integrity checks |
| **StaticMeshEditorSubsystem** | LOD management, collision setup, mesh operations |
| **EditorAssetSubsystem** | Asset operations: load, save, delete, duplicate, rename, find |
| **EditorUtilitySubsystem** | Run Editor Utility Widgets and Blueprints |
| **EditorLevelLibrary** | (Not a subsystem but commonly used) Level actor operations |

### 5.2 Additional Useful Editor APIs (via Python)

| API | Purpose |
|---|---|
| `AssetToolsHelpers.get_asset_tools()` | Asset creation, import, duplication |
| `EditorAssetLibrary` | Asset file operations (rename, delete, duplicate, checkout) |
| `EditorLevelLibrary` | Spawn actors, get/set actor transforms, PIE control |
| `MaterialEditingLibrary` | Create material expressions, connect nodes, set parameters |
| `SequencerTools` | Sequencer automation |
| `ScopedEditorTransaction` | Undo/redo support for batched operations |
| `ScopedSlowTask` | Progress bar for long operations |

### 5.3 MCP Implications

These subsystems represent clean, stable APIs that should be the primary interface for MCP tools. They handle thread safety, undo/redo integration, and editor state management. Wrapping subsystem calls is preferable to implementing operations from scratch in the C++ plugin.

---

## 6. Plugin Development Capabilities

A custom Unreal plugin has effectively unlimited access to the engine.

### 6.1 What a Plugin Can Do

| Capability | Details |
|---|---|
| **Full Editor API access** | Every editor subsystem, every module, every registered class |
| **Slate UI** | Custom editor panels, windows, tool bars, menus, details customizations |
| **Custom asset types** | New UObject-derived types with custom factories, editors, thumbnails |
| **Blueprint exposure** | Any C++ function/property can be exposed to Blueprints (and therefore Python) via UFUNCTION/UPROPERTY macros |
| **Module loading** | Load/unload other modules, access any loaded module's API |
| **Console command registration** | Register new console commands |
| **Editor modes** | Custom editor modes with their own tools and UI |
| **Tick functions** | Execute code every frame |
| **TCP/HTTP servers** | Host network servers within the editor process (exactly what MCP plugins do) |
| **File I/O** | Full filesystem access |
| **Reflection system access** | Query and manipulate UObjects, UClasses, UProperties at runtime |
| **Transaction system** | Participate in undo/redo |
| **Notification system** | Toast notifications, message log entries |
| **Automation testing** | Register and run automated tests |

### 6.2 Module Types

- **Runtime** -- Loaded in all targets (editor, game, server)
- **Editor** -- Loaded only in editor builds (appropriate for MCP)
- **Developer** -- Loaded in editor and development builds but not shipping
- **Program** -- Standalone programs

### 6.3 MCP Implications

The C++ plugin approach gives MCP servers maximum flexibility. The plugin can access anything the editor can. The question isn't "what *can* we do?" but "what *should* we expose, and how do we keep it safe?"

---

## 7. Community MCP Implementations -- Patterns and Approaches

### 7.1 Architectural Patterns

Three main patterns have emerged:

1. **C++ Plugin + Python MCP Server (TCP):** Most common. Plugin runs a TCP server inside Unreal, Python server connects and translates MCP protocol to engine commands. Used by chongdashu, flopperam, VedantRGosavi.
   - *Pros:* Python ecosystem (FastMCP), easy to extend tool definitions, C++ for performance-critical engine operations
   - *Cons:* Two processes to manage, TCP serialization overhead, connection management complexity

2. **C++ Plugin + TypeScript MCP Server:** Used by ChiR24/Unreal_mcp.
   - *Pros:* TypeScript type safety, npm ecosystem
   - *Cons:* Same two-process architecture, less common in Unreal tooling

3. **Source Code Analysis (no engine connection):** Used by unreal-analyzer-mcp.
   - *Pros:* Works without running editor, good for code review/understanding
   - *Cons:* Cannot control the editor, limited to static analysis

### 7.2 Common Safety Patterns

- **Console command blocking:** Pattern-based validation to reject dangerous commands (ChiR24)
- **Rate limiting:** Per-IP request throttling (ChiR24 -- 60 req/min)
- **Graceful degradation:** Server starts even without Unreal connection (ChiR24, chongdashu)
- **Connection-per-command:** Reconnect for each command to handle Unreal's connection lifecycle (chongdashu)

### 7.3 What's Working Well

- Actor CRUD operations (spawn, delete, transform, query)
- Basic Blueprint creation and component configuration
- Viewport control and screenshots
- Console command execution (with safety filters)

### 7.4 What's Still Rough

- Blueprint node graph manipulation (complex, brittle APIs)
- Material editing (awkward intermediate objects)
- Sequencer control (limited by Python API gaps)
- No implementations tackle source control integration
- Build pipeline automation is minimal
- Testing/validation workflows barely explored

---

## 8. Opportunities for Expansion

Based on the analysis above, here are capabilities an ambitious MCP server should expose that current implementations largely don't.

### 8.1 High-Value, Currently Missing

| Capability | Why It Matters | Feasibility |
|---|---|---|
| **Arbitrary Python execution** | The `unreal` module can do almost anything in the editor. Rather than wrapping every operation in a C++ command, let the AI write and execute Python scripts directly. This is the single highest-leverage capability. | High -- just need a "run Python" command in the C++ plugin that calls `IPythonScriptPlugin::ExecPythonCommand()` |
| **Asset search and registry queries** | Find assets by type, name, path, tags. Current implementations can spawn known assets but can't discover what's available. | High -- `AssetRegistryModule` is straightforward |
| **Asset dependency graph** | "What references this texture? What will break if I delete this mesh?" | High -- `AssetRegistry::GetDependencies/GetReferencers` |
| **Material graph manipulation** | Create/edit materials programmatically. Critical for look-dev workflows. | Medium -- `MaterialEditingLibrary` exists but API is finicky |
| **Sequencer automation** | Create cinematic sequences, add camera tracks, set keyframes. Huge for automated content creation. | Medium -- API exists with gaps (see Section 2.2) |
| **Build and cook operations** | Trigger lighting builds, cook content, package projects. | Medium -- available via editor automation and commandlets |
| **Source control integration** | Check out files, submit changelists, resolve conflicts. | Medium -- `ISourceControlModule` API exists |
| **Project settings access** | Read/write project settings (physics, rendering, input, etc.) | High -- exposed via CVars and config system |
| **Editor Utility Widget execution** | Run existing Editor Utility Blueprints/Widgets | High -- `EditorUtilitySubsystem` |
| **Content validation** | Run asset validation, check for errors, data integrity | High -- `EditorValidatorSubsystem` |

### 8.2 Advanced Capabilities

| Capability | Why It Matters | Feasibility |
|---|---|---|
| **Live viewport capture** | Capture what the viewport currently shows as an image. Enable visual reasoning by the AI. | High -- screenshot APIs exist; the real value is making this a feedback loop |
| **Play-In-Editor control** | Start PIE, interact with running game, capture output log, stop PIE. Enables test-driven development workflows. | Medium -- `LevelEditorSubsystem` has PIE controls, output log capture is doable |
| **Diff and comparison** | Compare two versions of a Blueprint, level, or asset. | Hard -- no built-in diff API, would need custom implementation |
| **Procedural generation framework** | Higher-level tools for procedural content (not just spawn-one-cube but generate-a-village). | Medium -- composable from existing primitives but needs thoughtful design |
| **Niagara system creation** | Particle system authoring. Currently not exposed by any MCP server. | Hard -- Niagara API is complex |
| **Animation montage/blendspace creation** | Animation asset authoring. | Hard -- API coverage is incomplete |
| **Data table/curve management** | Create and edit data tables, curve assets. Common for gameplay data. | Medium -- APIs exist |
| **Landscape editing** | Terrain sculpting, painting, foliage placement. | Hard -- specialized APIs, complex |
| **World Partition operations** | Manage streaming levels, world partition grids. | Medium -- API exists in UE5.x |

### 8.3 Architectural Improvements

| Improvement | Description |
|---|---|
| **Hybrid transport** | Use Remote Control API for property/function access (maintained by Epic, stable) + custom plugin for operations RCA doesn't support. Reduces maintenance burden. |
| **Async command execution** | Current implementations are synchronous. Long operations (builds, imports) block the MCP server. Need task-based execution with status polling. |
| **Undo transaction grouping** | Wrap related MCP operations in single undo transactions so the user can Ctrl+Z an entire AI operation, not just the last sub-step. |
| **Resource/prompt integration** | Expose Unreal documentation, project conventions, and asset catalogs as MCP resources. Provide prompts for common workflows (e.g., "set up a third-person character"). |
| **Viewport streaming** | Stream viewport as image data for visual AI feedback. More sophisticated than point-in-time screenshots. |
| **Event subscriptions** | Use MCP sampling or notifications to alert the AI when things change in the editor (asset saved, compilation error, PIE started, etc.). |
| **Safety tiers** | Graduated permission levels: read-only (query state) -> modify (change properties) -> create (new assets/actors) -> destructive (delete, build, package) -> system (console commands, settings). Let users choose their comfort level. |

### 8.4 The Killer Feature: Python Execution Bridge

The single most impactful capability would be **arbitrary Python script execution** within Unreal. Here's why:

1. The `unreal` Python module exposes nearly everything the editor can do.
2. Rather than implementing 200+ individual MCP tools in C++, you implement ONE tool: "execute Python script."
3. The AI (Claude, etc.) already knows Python and can write Unreal Python scripts based on documentation.
4. Any new Unreal feature that gets Blueprint exposure automatically becomes accessible -- zero MCP server changes needed.
5. Combined with viewport capture for visual feedback, this creates a general-purpose editor automation system.

The risk is safety -- arbitrary code execution is powerful and dangerous. Mitigation approaches:
- Sandboxed execution environment with restricted imports
- Transaction wrapping for undo support
- Execution timeout to prevent infinite loops
- Output capture for error handling
- Explicit user consent for destructive operations

### 8.5 Priority Ranking for Our MCP Server Evolution

**Phase 1 -- Foundation (highest impact, lowest risk):**
1. Python script execution bridge
2. Asset search and registry queries
3. Asset dependency/referencer queries
4. Project settings read/write (CVars)
5. Undo transaction support

**Phase 2 -- Content Creation:**
6. Material editing tools
7. Sequencer automation
8. Enhanced viewport capture (for visual AI feedback)
9. PIE control and output log capture
10. Content validation

**Phase 3 -- Production Pipeline:**
11. Build/cook automation
12. Source control integration
13. Data table management
14. Editor Utility Widget execution
15. Safety tier system

**Phase 4 -- Advanced:**
16. Event subscriptions / editor notifications
17. Procedural generation framework
18. Landscape editing
19. Niagara system creation
20. Animation asset authoring

---

## Sources

### Unreal MCP Projects
- [chongdashu/unreal-mcp](https://github.com/chongdashu/unreal-mcp) -- Python + C++ plugin, TCP socket architecture
- [ChiR24/Unreal_mcp](https://github.com/ChiR24/Unreal_mcp) -- TypeScript + C++, 36 tools, most comprehensive
- [flopperam/unreal-engine-mcp](https://github.com/flopperam/unreal-engine-mcp) -- Python + C++, world-building focus
- [VedantRGosavi/UE5-MCP](https://github.com/VedantRGosavi/UE5-MCP) -- Architecture documentation
- [gingerol/vhcilab-unreal-engine-mcp](https://github.com/gingerol/vhcilab-unreal-engine-mcp) -- Academic, scene building
- [ayeletstudioindia/unreal-analyzer-mcp](https://github.com/ayeletstudioindia/unreal-analyzer-mcp) -- Code analysis, not engine control
- [AlexKissiJr/UnrealMCP](https://mcpservers.org/servers/AlexKissiJr/UnrealMCP) -- Plugin-based
- [Epic Forums MCP Discussion](https://forums.unrealengine.com/t/is-there-a-plan-to-provide-a-mcp-model-context-protocol-server/2580648)

### Unreal Python API
- [Scripting the Editor Using Python (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/scripting-the-unreal-editor-using-python)
- [Python API Reference (UE 5.0)](https://docs.unrealengine.com/5.0/en-US/PythonAPI/module/unreal.html)
- [Short-Fuse-Games/UnrealHelpers](https://github.com/Short-Fuse-Games/UnrealHelpers) -- Python utility examples
- [Python Scripting in Sequencer](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-scripting-in-sequencer-in-unreal-engine)

### Remote Control API
- [Remote Control API HTTP Reference (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/remote-control-api-http-reference-for-unreal-engine)
- [Remote Control Preset API (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/remote-control-preset-api-http-reference-for-unreal-engine)
- [UnrealRemoteControlWrapper](https://github.com/cgtoolbox/UnrealRemoteControlWrapper) -- Python wrapper

### Editor Subsystems
- [Programming Subsystems (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/programming-subsystems-in-unreal-engine)
- [LevelEditorSubsystem API](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Editor/LevelEditor/ULevelEditorSubsystem)
- [Most Used Editor APIs (TAPython)](https://www.tacolor.xyz/tapython/most_used_editor_apis.html)

### Console Commands
- [Console Commands Reference (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-console-commands-reference)
- [Community Console Variables List](https://forums.unrealengine.com/t/unreal-engine-5-all-console-variables-and-commands/608054)

### Plugin Development
- [Plugins in Unreal Engine (UE 5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/plugins-in-unreal-engine)
- [Creating Custom Asset Types (Community Tutorial)](https://dev.epicgames.com/community/learning/tutorials/vyKB/unreal-engine-creating-a-custom-asset-type-with-its-own-editor-in-c)
