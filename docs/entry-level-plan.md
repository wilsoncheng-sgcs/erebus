# Entry-Level Erebus Variant: Floor-Marker Victims

> Design/implementation plan for the `entry-level-floor-victims` branch and
> the companion [erebus-map-editor-RCJA](https://github.com/wilsoncheng-sgcs/erebus-map-editor-RCJA)
> repo. Sections 1-3 (proto, `Victim.py`, `MainSupervisor.py`) are implemented
> here; section 4 (editor fork + Entry Level export branch) is implemented in
> the map editor repo. The Verification section's steps 1-2 have been
> code-reviewed but not run live (no Webots/browser available in the
> implementing environment) - still worth doing by hand before relying on
> this for a real event.

## Context

The user is adapting Erebus (RoboCup Junior Rescue simulation) for a local region event with two tiers: a **regional** tier (already scoped in a prior conversation, using the existing ruleset roughly as-is) and a further-simplified **entry level** tier for beginner teams. Two entry-level simplifications are in scope here:

1. **No floating walls** — confirmed to be a map-authoring guideline only. Walls in Erebus are always tile-edge properties on `worldTile.proto` (`topWall`/`bottomWall`/`leftWall`/`rightWall` fields); a wall detached from a tile boundary is structurally impossible once a `.wbt` is exported. There is nothing to change or enforce in code — this becomes a short documentation note for people building maps in the online editor.
2. **Floor-mounted colour markers instead of wall-mounted signs** — small coloured squares centered on a tile's floor, sized ~10% of the tile, sensed by a simple downward-facing colour sensor (already present on the robot and already used for swamp/hole sensing). This replaces the current vertical wall-mounted victim signs, which require facing/orientation checks beginners find harder to satisfy. A few colour-coded types are kept (red/yellow/green ↔ harmed/unharmed/stable) rather than one generic marker. Cognitive Targets (hazmat) and Fake victims are dropped entirely for this tier — out of scope for entry level.

A post-processing-script approach (transform an already-exported `.wbt`) was considered and **rejected**: it's an extra manual step end users would routinely forget, since the map generator is meant to be used directly. Instead, the plan is to **extract and fork the actual map editor** into a standalone app, with a first-class tier selector (Original / Intermediate / Entry Level) baked into the export flow itself — no separate step to miss.

This work spans **two separate projects, both as proper forks**: the Erebus simulator changes and the map editor fork.

**The Erebus side should also be a fork, not a direct edit to this clone.** Checked directly: this worktree's `origin` remote points at `https://github.com/robocup-junior/erebus.git` — the upstream project itself, not a repo the user owns push access to. Since this entry-level ruleset is a local-competition customization (not something intended to merge upstream), the right structure is a proper GitHub fork (`wilsoncheng-sgcs/erebus`) that this work happens on, rather than accumulating unpushable local commits against upstream's `origin`.

The map editor is a **second, separate new repository** (not nested inside the Erebus fork, and not a literal GitHub fork of `rcj-rescue-cms` either — confirmed decision: a fresh repo containing only the extracted `sim_editor` files plus a `LICENSE` carrying the original MIT attribution, avoiding pulling in `rcj-rescue-cms`'s unrelated Node/Express/MongoDB backend and CMS features). It carries a different upstream license provenance and an entirely different tech stack (Angular/Bootstrap/vanilla JS vs. this repo's Python/Webots) than the Erebus fork, so keeping them as two repos avoids blurring either.

**Confirmed division of labor**: the fork and new-repo creation are performed directly via `gh` CLI (already authenticated as `wilsoncheng-sgcs` with `repo` scope) as the first implementation steps, not manually by the user — i.e. plan approval is followed immediately by:
1. `gh repo fork robocup-junior/erebus --clone --remote` (creates `wilsoncheng-sgcs/erebus`, clones/wires it up) — engine-side changes (`FloorVictim.proto`, `Victim.py`, `MainSupervisor.py`) happen here.
2. `gh repo create wilsoncheng-sgcs/erebus-map-editor` (fresh, unrelated repo; name adjustable) — the extracted/forked map editor lives here, with a `LICENSE` file carrying the original `rcj-rescue-cms` MIT attribution.
3. Implementation (sections 1-6 below) proceeds against these two repos.

The real online editor's source does exist externally: `robocup-junior/rcj-rescue-cms`, MIT-licensed (2016, Fredrik Lofgren), branch `develop/2026-sim`, client logic at `public/javascripts/sim_editor/sim_editor.2026.js` (~161KB, unminified) plus `views/sim_editor/sim_editor_2026.pug`. Deep research into that code (via GitHub API/raw-content fetches — the repo isn't checked out locally) found it's **cleanly extractable**:
- The map data model (`$scope.cells`, a flat `"x,y,z"`-keyed object) is pure JSON with **zero backend/MongoDB coupling** — load/save is just browser File API (download/upload JSON), no REST calls.
- `.wbt` generation happens entirely client-side in one large function, `createWorld()` (~1100 lines, string templating — quaternion rotation math for wall-mounted victims/curved walls is the bulk of the complexity, not any server dependency).
- The Pug template's CMS coupling is shallow (navbar/footer/i18n only) — the actual editor grid is a self-contained Angular 1.x controller with no CMS-specific services.
- Verdict from the research pass: **medium-effort, low-risk extraction** — copy `sim_editor.2026.js` + the `createWorld()`/helper functions largely as-is, strip the CMS layout wrapper, hardcode the ~30 translation strings, keep Angular/Bootstrap/OpenCV.js as plain `<script>` tags. Since it's MIT-licensed, this fork is legally straightforward provided the license notice is retained.

Because we own this fork, **intermediate tier needs no map-format changes at all** — it reuses the exact same map data/export as "Original" (the intermediate/regional simplifications discussed earlier in this conversation are purely rules/scoring changes enforced by the simulator engine, not the map file). Only "Entry Level" needs new export behavior.

The existing identify-message wire protocol (9-byte `(est_x*100, est_z*100, type_char)` packet) is unchanged — floor markers are still identified by the robot stopping nearby and sending the same message type as today. Only the *shape/placement* of the marker and the *orientation check* in detection differ.

**Verified, not assumed:** shipped worlds use `xScale=zScale=0.4` (`game/worlds/world1.wbt:60-62`), so the real tile footprint is `0.3 * 0.4 = 0.12m`, not the nominal `0.3m` the proto's own default (`1.0`) would imply. The existing wall-mounted `Victim` sign is a `0.016m` box (`game/protos/Victim.proto:52`) — already about 13% of a 0.12m tile — so "10% of tile" is consistent with existing proportions, not a big departure. The marker size must therefore be **computed per-world from the actual scale found in the file**, not hardcoded.

## Approach

### 1. New proto: `game/protos/FloorVictim.proto`

Modeled directly on `Victim.proto` but flat and floor-mounted instead of vertical:

```
PROTO FloorVictim [
    field SFVec3f    translation   0 0 0
    field SFBool     found         FALSE
    field SFString   name          "FloorVictim"
    field SFString   type          "harmed"     # "harmed" | "unharmed" | "stable"
    field SFInt32    scoreWorth    10
    field SFFloat    size          0.03         # side length, set per-instance by the conversion script
]
```

- Geometry: `Box { size <size> 0.001 <size> }`, translation.y raised by half the box height (avoid z-fighting with tile floor, same trick `worldTile.proto` uses for its own floor box).
- No `rotation` field — a floor marker has no facing, so there's nothing for it to hold.
- Colour via a plain `Material { diffuseColor ... }`, no texture assets needed (simpler to author than `Victim.proto`'s 6 PNGs): harmed → red `0.85 0.1 0.1`, unharmed → green `0.1 0.75 0.1`, stable → yellow `0.9 0.85 0.1`.
- Skip a "found" visual state for v1 (no scoring impact, and there's no texture-swap mechanism for a flat material) — pure simplification, can be added later via `emissiveColor` if wanted.

### 2. Detection code: `game/controllers/MainSupervisor/Victim.py`

Add `FloorVictim(VictimObject)` as a sibling of `Victim`/`CognitiveTarget`, reusing `Victim.HARMED/UNHARMED/STABLE` and the same `H/U/S` type-char mapping — this keeps every scoring code path (`Robot.increase_score`, room multipliers, misidentification penalty) completely untouched, since the type-char space is identical to today's human victims.

- Override `on_same_side()` to `return True` unconditionally. This is the entire fix for "skip the wall-facing check for floor types" — a clean OOP override, **zero changes needed in `MainSupervisor.py`'s `_detect_victim()` filter itself** ([MainSupervisor.py:557-562](game/controllers/MainSupervisor/MainSupervisor.py:557)), since it calls `h.on_same_side(self.robot_obj)` uniformly across whatever object types the iterator holds.
- Leave `check_position()`'s `0.09m` radius as-is. It governs position-estimate tolerance (robot's actual + estimated proximity to the true marker location), not marker footprint — shrinking it to match the marker's physical size would make detection *harder*, which cuts against the beginner-friendliness goal. Reuse the same default the existing wall victims use.
- Extend `VictimManager` ([Victim.py:250-339](game/controllers/MainSupervisor/Victim.py:250)) with `self.floor_victims: list[FloorVictim] = self._get_floor_victims()`, mirroring `_get_victims()`. **Important difference from the existing getters**: `HUMANGROUP`/`TARGETGROUP` are assumed always present in every world, but `FLOORVICTIMGROUP` will only exist in entry-level worlds produced by the new script — `_get_floor_victims()` must handle `getFromDef('FLOORVICTIMGROUP')` returning `None` and just return `[]`, so regional/regular worlds (which will never have this DEF) keep working unmodified.
- Add the analogous reset loop in `reset_victim_textures()` ([Victim.py:341-347](game/controllers/MainSupervisor/Victim.py:341)) so floor victims' `identified` flag clears between runs like everything else.

### 3. Wiring in `MainSupervisor.py`

- [MainSupervisor.py:545](game/controllers/MainSupervisor/MainSupervisor.py:545): change the default iterator from `self.victim_manager.victims` to `self.victim_manager.victims + self.victim_manager.floor_victims`. Since regional maps never populate `floor_victims` (empty list) and entry-level maps never populate `victims` (the conversion script converts all of them), this concatenation is a no-op for existing maps and "just works" for entry-level ones without a mode flag. (There's already a precedent for this exact `list_a + list_b` pattern at [MainSupervisor.py:941](game/controllers/MainSupervisor/MainSupervisor.py:941), which combines `victims + targets` for the auto-camera list — add `+ floor_victims` there too for camera-follow consistency.)
- No other changes to `_detect_victim()`, scoring math, or the wire protocol.

### 4. Standalone map editor fork: new sibling repo `erebus-map-editor/`

A **new, separate git repository**, sibling to this Erebus clone (not nested inside it), hosting a static/self-contained web app forked from the CMS's `sim_editor.2026.js` + `sim_editor_2026.pug`. No Node/Express/MongoDB — just HTML/CSS/JS served as static files (or opened directly), matching the source app's already-stateless (file-in/file-out) design.

**Setup:**
0. `git init` a new repository at the chosen sibling path (default proposal: `erebus-map-editor/`, alongside this Erebus clone's parent directory), with its own `LICENSE` file (MIT, retaining the original 2016 Fredrik Lofgren copyright notice — required since substantial upstream code is being copied in) and its own `README.md`.

**Extraction steps:**
1. Copy `sim_editor.2026.js` into `js/sim_editor.js` largely verbatim — the map model (`$scope.cells`), file load/save (browser File API), and `createWorld()` `.wbt` generator have no CMS dependencies and port directly.
2. Convert `sim_editor_2026.pug` into a plain `index.html`: drop `extends ../includes/layout` and the navbar/breadcrumb/footer blocks, keep the `block content` tile-grid markup, add explicit `<script>`/`<link>` tags for Angular 1.x + UI Bootstrap + Bootstrap CSS + OpenCV.js (needed for the existing Room-4 custom-shape feature) as plain CDN or vendored includes. Replace `bootstrap-fileinput` with a plain `<input type="file">` (it was only used for the JSON import widget) to drop one dependency.
3. Replace the `{{ "key" | translate }}` i18n filter calls with hardcoded English strings (only ~30 keys) — drop `angular-translate` entirely since there's no multi-locale requirement for this fork.
4. `README.md` documents: the MIT license attribution/fork provenance from `robocup-junior/rcj-rescue-cms`, how to run the app (open `index.html`, or serve via any static file server), and that it must be able to reference this Erebus repo's `game/protos/FloorVictim.proto` (e.g. via a documented relative path or copy step) when generating Entry Level `.wbt` files, since the EXTERNPROTO reference in generated worlds points at that proto.

**New feature — ruleset tier selector:**
- Add a `$scope.ruleTier` (`"original" | "intermediate" | "entryLevel"`) driven by a radio/dropdown control added to the top of the editor UI (near the existing width/height/time/name fields in the pug-derived markup).
- **Original / Intermediate**: no change to `createWorld()` — both export through the exact existing code path unchanged (per the earlier decision that intermediate is a rules/scoring-only tier).
- **Entry Level**: branch inside `createWorld()`:
  - Skip the existing wall-token victim placement logic (`visualHumanPart()` / `calculateWallTokenRot()` / half-wall-victim loop) and the cognitive-target/fake generation paths entirely.
  - For each tile that has a victim assigned, emit a new `FloorVictim { ... }` node instead, using the tile's **already-known center coordinates** from the existing tile-iteration loop in `createWorld()` — this is strictly simpler than the earlier post-processing-script design, since the editor already has ground-truth tile positions and never needs to reverse-engineer them from exported text. Size: `tile_side * 0.10`, where `tile_side` is already computed elsewhere in `createWorld()` for tile geometry.
  - Emit `EXTERNPROTO "../protos/FloorVictim.proto"` in the header only for entry-level exports (the existing `Victim.proto`/`CognitiveTarget.proto`/`Fake.proto` EXTERNPROTOs are omitted instead, since none of those node types get used).
  - Group the emitted nodes under a new `DEF FLOORVICTIMGROUP Group { ... }`, matching the `FLOORVICTIMGROUP` DEF name the engine-side `Victim.py` change (`_get_floor_victims()`) looks for.

**UI restrictions for Entry Level tier:**
- When `ruleTier === "entryLevel"`, hide/disable the per-wall victim-type controls, cognitive-target code entry, fake-victim toggle, and half-wall-victim controls in the tile editor grid — replace with a single per-tile "victim colour" control (red/green/yellow ↔ harmed/unharmed/stable), since floor markers have no wall-side/orientation concept.
- **No-floating-walls guidance**: rather than just a docs note (as originally planned), since we now own the editor, add a lightweight non-blocking export-time warning when Entry Level is selected: flag any wall segment (`isWall` cell) that isn't part of a valid tile-boundary run, so map authors get immediate feedback in the tool they're already using instead of relying on a separate written guideline. This does not block export — it's advisory only, since it's still not a state the engine itself can misinterpret (see Context).

### 5. Scoring

No changes. `FloorVictim` reuses `Victim`'s `H/U/S` type space, so the existing `+10` type-match bonus, `+score_worth` base, room multipliers, and `-5` misidentification penalty all apply unchanged. Rebalancing point values for entry level (if wanted) is a separate decision better made after playtesting — not bundled into this change.

### 6. Documentation

The new repo's `README.md` covers: MIT license attribution to the upstream `rcj-rescue-cms` project, how to run the standalone editor, the tier selector and what each tier changes, and the colour legend (red=harmed, green=unharmed, yellow=stable) for entry-level floor markers. The "no floating walls" rule is surfaced as an in-editor advisory warning (see section 4) rather than a separate written guideline, since we control the tool directly.

## Files to create/modify

**In `wilsoncheng-sgcs/erebus` (fork, created via `gh repo fork` as the first implementation step — this worktree's `origin` is upstream and not push-able), engine-side changes only:**
- **New**: `game/protos/FloorVictim.proto`
- **Modify**: `game/controllers/MainSupervisor/Victim.py` — add `FloorVictim` class, extend `VictimManager` with `floor_victims`, extend `reset_victim_textures()`
- **Modify**: `game/controllers/MainSupervisor/MainSupervisor.py` — one-line iterator change at line 545, one-line addition at line 941
- **Reference (no changes)**: `game/protos/Victim.proto`, `game/protos/worldTile.proto`, `game/controllers/MainSupervisor/Tile.py`

**In `wilsoncheng-sgcs/erebus-map-editor` (fresh repo, created via `gh repo create` as the first implementation step — not a literal GitHub fork of `rcj-rescue-cms`), the editor extraction:**
- **New**: `index.html`, `js/sim_editor.js`, `css/` (forked from `sim_editor_2026.pug` / `sim_editor.2026.js` / `maze_edit.css`)
- **New**: `README.md`, `LICENSE` (MIT, retaining original attribution)
- **External reference (fork source, not modified in place)**: `robocup-junior/rcj-rescue-cms` branch `develop/2026-sim`, `public/javascripts/sim_editor/sim_editor.2026.js`, `views/sim_editor/sim_editor_2026.pug`

## Decisions made (deferred items from initial design pass, now resolved)

- Map generation: a first-class tier selector in a forked/self-hosted editor, not a post-processing script — avoids an easily-missed extra step for end users.
- Marker size: computed per-world from the tile geometry already known inside `createWorld()`, `tile_side * 0.10` as a **side length** (not area-based).
- `check_position` radius (engine-side): unchanged at `0.09m`.
- Mixed wall/floor victims in one map: not supported — Entry Level exports only ever contain floor victims; Original/Intermediate only ever contain wall victims.
- CognitiveTarget/Fake for Entry Level: omitted at generation time (UI hidden, `createWorld()` branch skips them) rather than stripped after the fact.
- "Found" visual state: skipped for v1 (cosmetic only, no scoring impact).
- Node grouping: new sibling `DEF FLOORVICTIMGROUP`, not reusing `HUMANGROUP`.
- "No floating walls": surfaced as an in-editor advisory warning at export time, not a separate doc (see section 4) — still not something the engine itself needs to validate, since it's structurally impossible once a `.wbt` exists.
- Editor extraction scope: fork `sim_editor.2026.js` + `sim_editor_2026.pug` largely as-is (confirmed low backend coupling, pure client-side `.wbt` generation); keep Angular/Bootstrap/OpenCV.js, drop `angular-translate` and `bootstrap-fileinput` as the two prunable dependencies.
- Repo layout: the editor fork lives in its **own new sibling repository**, not nested inside the Erebus fork — separate license/tech stack, separate versioning.
- Erebus-side changes also land on a **fork** (`<user>/erebus`), not upstream `origin` — confirmed this worktree's `origin` is `robocup-junior/erebus` directly, which isn't push-able by the user.

## Verification

1. Serve/open the new repo's `index.html` locally, build a small test map (a handful of tiles, a couple of victims) with tier = **Original**, export, and diff the resulting `.wbt` against what the unmodified upstream editor would produce for the same map (structurally — same node types/counts) to confirm the fork didn't regress the baseline path.
2. Repeat with tier = **Entry Level**: confirm the exported `.wbt` contains `FloorVictim` nodes (not `Victim`/`CognitiveTarget`/`Fake`), an `EXTERNPROTO` line for `FloorVictim.proto`, and a `DEF FLOORVICTIMGROUP` group; open it in Webots and visually confirm coloured squares sit flush and centered on tile floors with the right colour per type, no z-fighting.
3. Trigger the in-editor "floating wall" advisory warning deliberately (place a disconnected wall segment) and confirm it surfaces without blocking export.
4. Run an example player controller (e.g. `player_controllers/ExamplePlayerController_updated.py`) against the Entry Level world: drive over a marker, stop, send the identify packet, confirm a score increase and that detection no longer depends on approach angle (test multiple approach directions to confirm the orientation check truly no longer applies).
5. Run the same controller against an unmodified Original/Intermediate world afterward to confirm the `victims + floor_victims` iterator change and `_get_floor_victims()` are no-ops there (identical score behavior to before the engine-side change).
6. Sanity-check room-multiplier and misidentification paths still work for floor victims (approach a room-2/3/4 marker if the test map has one; trigger a deliberate wrong-type identify to confirm the `-5` penalty still applies).
