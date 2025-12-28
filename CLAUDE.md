# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a hotfix fork (`oarc-mod-tmp-fix`) of Oarc Multiplayer Spawn - a Factorio mod for separate spawn locations in multiplayer. Supports Factorio 2.0 with Space Age expansion support in progress. Incompatible with `oarc-mod` (original) and `rso-mod`.

## Development Setup

Link the repo into your Factorio mods folder for live edits:

```bash
ln -s "$(pwd)" "$HOME/.factorio/mods/oarc-mod-tmp-fix_0.2.3"  # adjust version to match info.json
```

Copy the scenario template for curated defaults:

```bash
rsync -a TEMPLATE_SCENARIO/OARC "$HOME/.factorio/scenarios/OARC"
```

Package a release:

```bash
zip -r ../oarc-mod_2.1.x.zip . -x '.git*' 'devplan.txt'
```

## Testing

No automated test harness. Validate changes inside Factorio using multiplayer saves or the shipped scenario, watching `factorio-current.log` for script warnings.

Developer helpers in `lib/oarc_tests.lua` are callable via console:

```
/sc TestFonts(game.player)
/sc ClearTestFonts(game.player)
```

Test critical flows after each change: new player spawn, buddy joins, regrowth, sharing toggles.

## Architecture

### Entry Point and Event System

`control.lua` is the main entry point. It orchestrates Factorio runtime events and loads modules from `lib/`. The mod is entirely event-driven using Factorio's event system (`on_init`, `on_tick`, `on_player_created`, `on_chunk_generated`, etc.).

### Module Load Order (from control.lua)

```lua
require("lib/oarc_utils")           -- Utilities and constants
require("lib/config")               -- Configuration defaults (DO NOT EDIT)
require("lib/config_parser")        -- Parse/validate configuration
require("lib/regrowth_map")         -- Chunk cleanup system
require("lib/holding_pen")          -- Temporary spawn surface
require("lib/separate_spawns")      -- Core spawn logic
require("lib/separate_spawns_guis") -- Spawn choice GUI
require("lib/oarc_gui_tabs")        -- Tabbed interface system
require("lib/offline_protection")   -- Enemy group protection
require("lib/scaled_enemies")       -- Enemy difficulty scaling
require("lib/sharing")              -- Item/electricity sharing
```

### Core Subsystems

**Separate Spawns** (`lib/separate_spawns.lua`, 87KB): Core spawn system managing unique spawn areas per player, buddy spawns, shared spawns, and main force spawns. Key state: `storage.unique_spawns`.

**Regrowth** (`lib/regrowth_map.lua`, 31KB): Chunk lifecycle management to prevent save bloat. Marks inactive chunks for deletion using flags (`REGROWTH_FLAG_REMOVAL`, `REGROWTH_FLAG_ACTIVE`, `REGROWTH_FLAG_PERMANENT`).

**Holding Pen** (`lib/holding_pen.lua`): Temporary surface `oarc_holding_pen` where players spawn before being teleported to their actual base via the delayed spawn queue.

**Sharing** (`lib/sharing.lua`): Cross-player resource sharing via custom entities `oarc-linked-chest` and `oarc-linked-power`.

### State Management

Global state stored in `storage` table with namespacing:

- `storage.ocfg` - Configuration settings
- `storage.oarc_surfaces` - Surface spawn settings
- `storage.unique_spawns` - Registered spawn areas
- `storage.player_respawns` - Player respawn points
- `storage.delayed_spawns` - Queue for delayed spawning
- `storage.rg` - Regrowth system state

### GUI System

Tab-based GUI located in `lib/gui_tabs/`:

- `spawn_controls.lua` - Player spawn/respawn controls
- `surface_config.lua` - Per-surface configuration
- `settings_controls.lua` - Mod settings management
- `player_list.lua` - Connected player list
- `item_shop.lua` - Coin-based equipment shop
- `server_info.lua` - Server status display
- `mod_info_faq.lua` - Help and FAQ

### Planet Configs

Space Age planet spawn configs in `lib/planet_configs/`: nauvis.lua, vulcanus.lua, gleba.lua, fulgora.lua, aquilo.lua

### Custom Events

8 custom events for external mod integration (defined in `data.lua`):

- `oarc-mod-on-spawn-choices-gui-displayed`
- `oarc-mod-on-spawn-created`
- `oarc-mod-on-spawn-remove-request`
- `oarc-mod-on-player-reset`
- `oarc-mod-on-player-spawned`
- `oarc-mod-character-surface-changed`
- `oarc-mod-on-chunk-generated-near-spawn`
- `oarc-mod-on-config-changed`

### Remote Interface

Exposed via `remote.add_interface("oarc_mod", ...)`:

- `get_mod_settings()` - Returns `storage.ocfg`
- `get_unique_spawns()` - Returns `storage.unique_spawns`
- `get_player_home_spawn(player_name)`
- `get_player_primary_spawn(player_name)`
- `create_custom_gui_tab(player, tab_name)`
- `get_custom_gui_tab_content_element(player, tab_name)`
- `remove_or_reset_player(player_name, remove_player)`

## Code Style

- Four-space indentation, prefer `local` scope
- Files: snake_case, Functions: PascalCase (e.g., `ValidateAndLoadConfig`), Constants: ALL_CAPS
- Preserve LuaDoc annotations (`---@param`, `---@class`)
- Comments focus on gameplay intent or non-obvious engine quirks

## Key Constants

```lua
CHUNK_SIZE = 32
MAX_FORCES = 64
TICKS_PER_SECOND = 60
TICKS_PER_MINUTE = 3600
ABANDONED_FORCE_NAME = "_ABANDONED_"
HOLDING_PEN_SURFACE_NAME = "oarc_holding_pen"
```

## Commit Guidelines

- Concise, sentence-case subjects capturing the subsystem touched
- Update `changelog.txt` alongside gameplay tweaks
- PRs should describe motivation, list manual test steps, include screenshots for GUI/map-gen changes
