# Template / Stamp Mod — Handoff Document

> **Purpose:** Standalone briefing for the next Claude Code session that will
> implement an in-game template/stamp feature for Farthest Frontier. Built on
> top of the existing **BuildingCostsDump** MelonLoader mod and the
> **Farthest Frontier Planner** web app.

---

## Quick context

This is a follow-on project to the [Farthest Frontier Planner](https://sagedragoon79.github.io/FarthestFrontierPlanner/) — a browser-based city planner I built (with previous Claude sessions) for the city builder *Farthest Frontier* by Crate Entertainment. The planner lets you design a layout on a grid, sees real desirability/coverage feedback, and exports the layout as JSON.

The **PROJECT_OVERVIEW.txt** in this repo summarizes the planner. Read that first if you want full context. This handoff focuses on **what we're building next**.

## The pitch

Players design a layout in the planner (or in any future template editor). The mod adds an in-game **"Templates"** feature that lets the player **stamp** a saved layout onto the actual game world at their cursor position — queuing real construction tasks for villagers in one click.

Think: a starting-town blueprint you reuse every new game.

---

## Design decisions already locked in

These were debated and decided in the prior session — don't re-litigate unless you find a strong reason:

| Decision | Choice | Why |
|---|---|---|
| Construction type | **Real** (queue actual build sites for villagers) | Ghost-only would feel useless — players want the layout to *happen* |
| Unlock gating | **Yes** — only buildings the player has researched/unlocked can be stamped | Matches the game's normal placement gating; prevents skipping the tech tree |
| Collision | **All-or-nothing** — if any tile in the template fails (terrain, overlap, out of bounds), the entire stamp is rejected | Prevents partial gibberish layouts; clearer failure mode |
| Resource check | **None** — you can stamp even if you don't have the planks/stone for it | Matches the game's vanilla behavior (you can queue build sites without materials; villagers wait for resources) |
| CPU spike | **Rate-limit the placement loop** | Stamping 50+ buildings in one frame may stutter; spread across frames |
| Origin point | **Cursor = template anchor** | The cursor's grid tile becomes the template's (0,0) reference; or we offer center-based vs anchor-based as a setting |

## Architecture sketch

```
┌────────────────────────┐         ┌────────────────────────┐
│  Planner web app       │         │  Game (MelonLoader)    │
│  (already built)       │         │                        │
│                        │  JSON   │  ┌──────────────────┐  │
│  Designer mode         │ ─────►  │  │  Templates       │  │
│  Save layout to JSON   │ file    │  │  panel UI        │  │
│                        │         │  └──────────────────┘  │
│                        │         │                        │
│                        │         │  Stamp mode            │
│                        │         │  (cursor preview)      │
│                        │         │                        │
│                        │         │  Placement engine      │
│                        │         │  → BuildManager API    │
└────────────────────────┘         └────────────────────────┘
```

## Template JSON format (proposed)

```json
{
  "name": "Starter Plaza",
  "version": 1,
  "anchor": "center",
  "buildings": [
    {"id": "town_center", "dx": 0,  "dy": 0,  "dir": 0},
    {"id": "shelter",     "dx": -4, "dy": 0,  "dir": 0},
    {"id": "shelter",     "dx": -4, "dy": 4,  "dir": 0},
    {"id": "well",        "dx": 4,  "dy": 0,  "dir": 0},
    {"id": "dirt_road",   "dx": -2, "dy": 0,  "dir": 0, "len": 8}
  ]
}
```

- `id` — building identifier (planner's IDs map to game's `BuildingData.identifier`; the mod will need a small lookup table to bridge them, OR we standardize on the game's identifiers and the planner exports those)
- `dx, dy` — offset in grid tiles from the template anchor
- `dir` — rotation/direction (0 = default, 1 = vertical, 2/3 = diagonals for linear structures)
- `len` — length for linear structures (roads, walls, fences)

**Open question for the new session:** should the planner export use game identifiers (`MarketBuilding`) or planner identifiers (`market`)? Lean toward game identifiers — fewer mappings to maintain, more portable.

---

## Existing foundation

### BuildingCostsDump mod (v1.7) — lives at:

- Source: `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\BuildingCostsDumpMod\BuildingCostsDump.cs`
- Repo: https://github.com/sagedragoon79/BuildingCostsDump
- Built DLL: `bin/Release/BuildingCostsDump.dll`

This mod already demonstrates:
- MelonLoader bootstrap (`MelonMod.OnUpdate` polling for `GlobalAssets.buildingSetupData`)
- Reflection access to private serialized fields on `Building` subclasses
- Sprite extraction via `RenderTexture` blit + `Texture2D.ReadPixels`
- Walking the prefab graph (`BuildingData.prefabEntries[i].PREFAB()`)
- File I/O to `UserData/`

The new mod can either **extend** this one or be **separate**. Recommendation: **separate mod** — different concerns. Keep BuildingCostsDump as the data-extraction tool; new mod is the in-game feature. They share patterns but don't depend on each other.

### Decompiled game source

Lives at: `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\ff_decompile\`

Files the new session will need to read first:

- `BuildManager.cs` — has `cellSize = 5f`, owns the placement state machine
- `Building.cs` — base class for all buildings (huge file, ~7000 lines)
- `BuildingData.cs` — ScriptableObject with prefab references, costs, icon, prerequisites
- `Placeable.cs` — runtime placement controller; handles cursor follow, validation
- `PlaceableBuilding.cs` — building-specific placement
- Any `*HUDUI.cs` files — for adding UI buttons to the in-game HUD

### Critical discovery from prior session

**`BuildManager._cellSize = 5f`** — one grid tile in Farthest Frontier is **5 world units** wide. All radii from the game's data are in world units. Divide by 5 to get tile-space. This bit me hard last time; don't forget it.

### Planner export format

The planner saves to JSON via the **Save** button. Format is roughly:

```json
{
  "version": 1,
  "placed": [
    {"buildingId": "town_center", "x": 50, "y": 50, "w": 6, "h": 6, "dir": 0},
    ...
  ],
  "terrain": {"50,50": "water", ...}
}
```

To convert to a template, we'd:
1. Compute the centroid (or designated anchor)
2. Subtract anchor from each `x,y` → produces `dx,dy`
3. Strip terrain (templates are buildings only, at least for v1)

This conversion can happen either **in the planner** (export-as-template button) or **in the mod** (import any planner save and let the user pick the anchor).

---

## Recommended first session of work

In rough order — this is a 2-3 hour first session if it goes smoothly:

### Phase 1: Decompile + understand placement

**Goal:** Find the exact method the game's UI calls when a player clicks to place a building. Once we have that signature and call site, the rest is iteration.

```
Search in ff_decompile/ for:
- "Build" + "Queue" / "Queued" / "Place"
- "BuildSite" creation
- "ConstructionSite"
- references to BuildSiteData
- BuildManager methods involving Vector3 + BuildingData
```

Likely candidates:
- `BuildManager.PlaceBuilding(...)`
- `BuildManager.CreateBuildSite(...)`
- `Placeable.CommitPlacement()`
- Something on `PlaceableBuilding`

We need a method that takes:
- A `BuildingData` (or identifier we can look up)
- A `Vector3` world position (or grid coords we can convert)
- A rotation/direction
- And produces a real, queueable build site that villagers will work on

### Phase 2: Skeleton mod

```
TemplateStamperMod/
├─ TemplateStamperMod.cs       (MelonMod entry point)
├─ TemplateStamperMod.csproj   (copy from BuildingCostsDump)
├─ Stamper.cs                  (placement engine)
├─ Template.cs                 (JSON model + load/save)
├─ StamperUI.cs                (HUD button + panel)
└─ README.md
```

Reuse the `.csproj` setup from BuildingCostsDump — same references work.

### Phase 3: First placement test

Hardcode a tiny 3-building template (town_center, two shelters), bind it to a hotkey (e.g. F12), and just try to place it at the player's cursor. Goal: see actual build sites appear on the map.

### Phase 4: UI

Add a HUD button. Open a panel listing templates from `<game>/UserData/Templates/*.json`. Click a template → enter stamp mode → ghost preview follows cursor → click to commit, right-click to cancel.

### Phase 5: Validation + collision

Before committing the stamp, iterate every building in the template and validate:
- All tiles in bounds
- No collision with existing buildings
- No collision with impassable terrain
- Bridges are on water tiles
- Player has the building unlocked (check `prerequisiteIdentifiers` against tech tree state)

If ANY check fails, abort the entire stamp and show a tooltip explaining which tile/building was the blocker.

### Phase 6: Rate-limiting

Yield between placements with `MelonCoroutines.Start` or a coroutine that places one building per frame (or N per frame). Test with a 50-building template to see how it feels.

---

## Files the new session should bring in

Paste these into the new session's first prompt (or read them with the Read tool):

1. `C:\Users\saged\Desktop\FarthestFrontierPlanner\PROJECT_OVERVIEW.txt`
2. `C:\Users\saged\Desktop\FarthestFrontierPlanner\TEMPLATE_MOD_HANDOFF.md` (this file)
3. `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\BuildingCostsDumpMod\BuildingCostsDump.cs` (reference scaffold)
4. `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\BuildingCostsDumpMod\BuildingCostsDump.csproj` (reuse for the new mod)
5. `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\ff_decompile\BuildManager.cs`
6. `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\ff_decompile\Building.cs`
7. `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\ff_decompile\BuildingData.cs`
8. `C:\Users\saged\Documents\Farthest Frontier Planner Project Files\ff_decompile\Placeable.cs`
9. `C:\Users\saged\Desktop\FarthestFrontierPlanner\data\BuildingCostsDump.csv` (reference for game identifiers)

---

## Environment & paths

- **Game install:** `G:\SteamLibrary\steamapps\common\Farthest Frontier\Farthest Frontier (Mono)\`
- **Mods folder:** `G:\SteamLibrary\steamapps\common\Farthest Frontier\Farthest Frontier (Mono)\Mods\`
- **MelonLoader log (use Latest.log):** `<game>/MelonLoader/Latest.log`
- **MelonLoader path used by mod:** `<game>/MelonLoader/net35/MelonLoader.dll`
- **Game managed assemblies:** `<game>/Farthest Frontier_Data/Managed/Assembly-CSharp.dll`

## Build command (from BuildingCostsDump pattern)

```
cd <mod source folder>
dotnet build -c Release
```

Then copy `bin/Release/<modname>.dll` into the game's `Mods/` folder.

## Logging gotcha

MelonLoader **throttles/drops rapid log output during `OnInitializeMelon`**. If we need to log a lot during init, defer it to `OnMapLoaded` or later, otherwise messages get lost.

## Active mod compatibility

The user runs several Farthest Frontier mods (Tended Wilds, Forageable Transplantation, Manifest Delivery, Stalk and Smoke, FF UI Overhaul). The template mod should NOT conflict with these — most don't touch placement. The UI Overhaul mod (in active development by the user) may add HUD buttons too; we should pick a unique slot.

---

## Stretch goals (post-MVP)

- **In-game template editor** — right-click an existing layout in the world to "save as template"
- **Template browser UI** — preview thumbnails, search, tags
- **Sharing** — Steam Workshop integration or upload to a community gallery
- **Planner integration** — "Send to game" button in the planner that uses a deeplink or file drop
- **Rotation** — rotate the entire stamp before placing (4 cardinal directions)
- **Mirror / flip** — placement variations
- **Partial undo** — undo a stamp within N seconds
- **Stamp validation preview** — before committing, show tiles in green/red so the user sees exactly what will fail before clicking

---

## Notes & gotchas to remember

- **Sprite atlas Y-coords:** when extracting more icons via reflection, use `sprite.textureRect` not `sprite.rect`, and DO NOT y-flip. (Learned the hard way in BuildingCostsDump v1.5 → v1.6.)
- **`techTreeManager` is null on raw prefabs:** any property getter that touches it will throw. Read private serialized fields via reflection instead. (See `_strategicPlanningBonus` pattern in BuildingCostsDump v1.3.)
- **`BuildingData.placeablePrefab`** is the **ghost preview** prefab, NOT the actual building. Use `prefabEntries[i].PREFAB()` for the real one.
- **Disable conflicting mods** when extracting reference data (Dramatic_Desirability, VC_BuildingRadiusAdjust, etc).

## When in doubt

Look at how the game's own UI does it. The HUD's BUILD menu is the existing player-facing placement flow — it ultimately calls into `BuildManager` / `Placeable`. Any code path it uses, our mod can use the same one. Decompile the click-to-place chain and copy its idioms.

---

*Generated at end of the planner build session, 2026-04-28. Good luck. ⚒*
