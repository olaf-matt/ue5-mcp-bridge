# Unreal Engine 5.7 Levels, Actors & BSP Context

This context is loaded when working with level/actor MCP tools
(`get_level_actors`, `spawn_actor`, `move_actor`, `delete_actors`, `set_property`, `open_level`).

**Related contexts:** `actor.md` (spawning, components, transforms), `_gaps.md` (tool limitations).

> These are behaviors that repeatedly surprise agents driving the editor through MCP. Most come
> from the gap between the *editor world*, the *running PIE world*, and what the *Outliner* shows.

---

## 1. `get_level_actors` returns engine-internal actors the Outliner hides

The Outliner shows a curated list; `get_level_actors` returns the **actual** actors in the world,
including engine-internal ones the Outliner suppresses. Expect to see, in almost every level:

- `WorldSettings`
- A **builder brush** (typically `Brush_0`, class `Brush`) at the origin — see §2
- `AbstractNavData-Default`
- `GameplayDebuggerPlayerManager`
- Auto-spawned managers (e.g. a physics/buoyancy manager) and debug-draw actors

**Implication:** the count won't match what the user sees in the Outliner, and not every returned
actor is a "real" scene object. Filter by `class_filter` / `name_filter`, and don't act on an
actor just because it's in the list — confirm it's the one the user means.

---

## 2. The builder brush (`Brush_0`) is not a floor and does not render

Every level has a default **builder brush** — usually named `Brush_0`, class `Brush`, sitting at
the origin (Z=0). It is the editor's geometry-authoring tool, **not** scene geometry:

- It does **not render in game/PIE**.
- It usually does **not** appear in the Outliner.
- It has no gameplay purpose.

Do not mistake it for a floor, a backdrop, or a culprit for a visual issue. Hiding or deleting it
changes nothing visible at runtime.

---

## 3. `bHidden` on a Brush actor does NOT remove its BSP surface

BSP brush geometry is compiled into the level's **model** and rendered from there, independently of
the brush actor's visibility flag. So `set_property bHidden=true` on a BSP brush hides **nothing**
that's actually rendered.

To remove a BSP surface you must **delete the brush** (the editor rebuilds the BSP) or rebuild
geometry. This only matters for *additive/subtractive* BSP brushes that actually contribute
surfaces — the builder brush (§2) contributes none.

---

## 4. `set_property` essentials

- **Component sub-properties use dot notation:** `RootComponent.RelativeLocation`,
  `StaticMeshComponent.StaticMesh`, `LightComponent.Intensity`. Component names are the exact
  class-instance names (e.g. `HeightFogComponent0`, not the class name) — confirm via the actor.
- **Struct / FVector values:** the **UE text format** `(X=0,Y=0,Z=0)` is the reliable form for
  `FVector`/`FRotator`. A JSON object (`{"X":0,...}`) is documented as supported but has been seen
  to be rejected for some struct properties — if a struct set fails, switch to the text form.
- **`bHidden`** is *Actor Hidden In Game* (game/PIE render visibility) — not editor-only visibility,
  and (per §3) not effective on baked BSP surfaces.
- A `set_property` can **silently no-op** if the property is a Blueprint variable that isn't
  *Instance Editable* — enable that flag in the Blueprint before expecting an instance set to take.

---

## 5. Editor world vs PIE — the biggest gotcha

MCP edits (`set_property`, `spawn_actor`, `move_actor`, `delete_actors`, etc.) operate on the
**editor world**. A **running PIE session does not reflect them** — PIE copies the editor world at
play start and runs its own copy.

Consequences:
- After an editor-world edit, **stop and replay PIE** to see it. "I changed it and nothing happened"
  is almost always this.
- `capture_viewport` returns the **PIE** view while PIE is running, and the **editor** view
  otherwise — so a capture may not show your just-made editor edit until PIE restarts.
- To change something *in* a running PIE session, you must do it at runtime (e.g. Blueprint logic,
  `set_niagara_variable` on a live component), not via editor-world `set_property`.

---

## 6. `open_level` invalidates actor references

`open_level` tears down the current world. Any actor names/handles obtained before it are stale.
It is a **sequential-only** operation — never batch it with other actor ops; re-query the level
after opening.

---

## 7. Quick checklist

1. `get_level_actors` count ≠ Outliner count — filter and confirm the actor is the intended one.
2. A `Brush_0` at origin is the builder brush — ignore it for runtime visuals.
3. Need a BSP surface gone? Delete the brush (rebuilds BSP); `bHidden` won't do it.
4. FVector set failing? Use `(X=,Y=,Z=)`, not JSON.
5. Edit "didn't take" in PIE? You edited the editor world — stop and replay.
6. After `open_level`, re-query actors; old references are dead.
