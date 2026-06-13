# Unreal Engine 5.7 Niagara Context

This context is automatically loaded when working with Niagara MCP tools
(`niagara_modify`, `niagara_query`, `set_niagara_variable`, `sample_render_target_2d_array`).

**Related contexts:** `material.md` (render-target assets), `blueprint.md` (Blueprint ↔ Niagara
wiring), `actor.md` (NiagaraActor / NiagaraComponent on level actors). Tool limitations are
collected in `_gaps.md`.

> Niagara is an isolated graph world. Nothing from Blueprint or C++ semantics transfers in —
> its terms, namespaces, parameter binding, and HLSL all follow Niagara-specific rules.
> Most mistakes below come from assuming Blueprint/material intuition applies here.

---

## 1. The Niagara MCP tools

| Tool | Purpose |
|------|---------|
| `niagara_query` (`inspect`, `inspect_collection`) | **Read-only.** List a system's exposed `User.*` parameters with type + `is_data_interface`. Always do this first. |
| `niagara_modify` | Inspect and edit a system's emitters/stages/modules. Many operations (see below). |
| `set_niagara_variable` | Set a `User.*` override on a NiagaraComponent attached to a **level actor** (runtime/editor live-set). |
| `sample_render_target_2d_array` | Read one texel back from a `TextureRenderTarget2DArray` — the only way to verify GPU-sim output via MCP. |

Common `niagara_modify` operations: `list_emitters`, `list_stages`, `list_modules`,
`get_module_source`, `get_stage_properties`, `dump_stage_graph`, `add_module`, `remove_module`,
`set_module_input`, `set_module_hlsl`, `set_stage_execute_behavior`, `set_system_user_param`,
`create_scratchpad_module`, `add_scratchpad_module_param`, `bind_module_input`,
`configure_di_parameter`, `compile`.

Stage names accepted: `EmitterSpawn`, `EmitterUpdate`, `ParticleSpawn` (or `Spawn`),
`ParticleUpdate` (or `Update`), and named sim/event stages as listed by `list_stages`.

---

## 2. Mental model — structure and namespaces

```
NiagaraSystem
  └─ Emitter(s)
       └─ Stages (scripts):  EmitterSpawn, EmitterUpdate, ParticleSpawn,
                             ParticleUpdate, + custom GPU sim stages / EventHandlers
            └─ Modules (FunctionCall or inline CustomHlsl), executed top→bottom
                 └─ each module reads/writes the Parameter Map
```

Everything flows through the **Parameter Map**. Parameters live in namespaces:

| Namespace | Meaning | Lifetime |
|-----------|---------|----------|
| `User.*` | Exposed to the outside (Blueprint/C++/level actor). The only ones you can set externally. | Persistent; settable at runtime |
| `Emitter.*` | Per-emitter working values | Per frame |
| `Particles.*` | Per-particle attributes | Per particle |
| `Module.*` | A module's own inputs (inner-graph namespace) | Local to the module |
| `Output.Module.*` | A scratch-pad module's outputs (inner-graph namespace) | Local |
| `Engine.*` | Engine-provided read-only (e.g. `Engine.DeltaTime`) | Per frame |

The **same value has different names inside vs outside** a scratch-pad module — see §5.

---

## 3. Read before you write (feedback loop)

There is no generic "get property" for a running sim. Confirm state with:

- `niagara_query inspect` → exact `User.*` names + types + `is_data_interface`.
- `get_module_source` → a module's HLSL, its param reads/writes, and inputs.
- `dump_stage_graph` → full node/pin graph of a stage (who feeds whom).
- `sample_render_target_2d_array` → the actual GPU output values (requires the sim to have ticked).

**`dump_stage_graph` output is large** and is usually spilled to a file with very long single
lines. Read it by character-range/regex, not by line offset/limit.

---

## 4. Parameter gotchas (the high-frequency mistakes)

### 4a. A param's display "category" is NOT part of its name
The editor groups user params under display categories. A parameter shown as
`WindControl ▸ WindSpeed` has the actual name **`User.WindSpeed`** — "WindControl" is a UI
category only. Setting `"WindControl.WindSpeed"` targets a name that does not exist.
**Always take the real name from `niagara_query inspect`.**

### 4b. Setting an unknown parameter name silently succeeds and does nothing
`set_niagara_variable` (and Blueprint `SetNiagaraVariable*`) report success even when the name
matches no exposed parameter. There is no error. **Always verify the set took effect** — via
`sample_render_target_2d_array`, a visible sim change, or re-inspect. A "no-op" here has
repeatedly been misdiagnosed as a deeper tool/engine bug when it was just a wrong name.

### 4c. Value params vs Data Interface params need different setters
Check `is_data_interface` in the inspect output:

| `is_data_interface` | Setter |
|---------------------|--------|
| `false` (float/int/bool/vec2/vec3/color) | `set_niagara_variable`, or Blueprint `SetNiagaraVariableFloat/Vec3/Bool/...` |
| `true` (e.g. array float3, render-target DIs) | Blueprint `SetNiagaraArrayVector` etc. from `NiagaraDataInterfaceArrayFunctionLibrary` — the scalar setters silently fail on DIs |

`set_niagara_variable` currently supports float/int/bool/vec3/color only — **no Vector4** (see
`_gaps.md`). A "color" set writes an `FLinearColor` that a `Vector4f` param will not read.

### 4d. A User param is only "live" if a per-frame stage re-reads it
Whether a runtime change to `User.X` affects the sim depends on where it is consumed:
- Read by an **EmitterUpdate / per-frame** module (often a "set per frame" copy `User.X → Emitter.X`)
  → **live**; runtime changes take effect immediately.
- Read only by an **EmitterSpawn / ParticleSpawn** module → **snapshot at spawn**; runtime changes
  do nothing until the system reinitializes (`ReinitializeSystem`).

Use `dump_stage_graph` / `get_module_source` to see which stage reads the param before promising
"live tuning." Don't assume an exposed param is live.

### 4e. Values can be direct; Data Interfaces must be Module-input + stack-linked
Inside an inner graph: value types (float/vec3) work as direct `User.*` / `Particles.*` Map Get
reads. **Data Interfaces do NOT bind through a direct `User.<DI>` read** even if the user param is
fully configured — a DI must be a classic Module-level input (undotted `Module.X`) linked in the
stack via `bind_module_input` (or `configure_di_parameter`). Rule of thumb:
**dotted/direct for values, Module-input + stack link for DIs.**

---

## 5. Scratch-pad modules (custom HLSL with inputs/DIs)

### 5a. Two kinds of custom HLSL — only one can use Data Interfaces
| Kind | Created by | Editable in UI | DI access |
|------|-----------|----------------|-----------|
| **FunctionScript module** | `create_scratchpad_module` | Yes (double-click opens its graph) | **Yes**, via DI-typed signature inputs |
| **Stage-level `UNiagaraNodeCustomHlsl`** | `rebuild_scratchpad_inner_graph` | No graph; right-click only Cut/Copy/Delete | **No** (DI symbols only exist if another module in the stage registers the same DI) |

**If you need a DI in custom HLSL, use a FunctionScript module.** Never convert it to a
stage-level node when DI access matters — that's irreversible and drops DI access.

### 5b. DI function calls in custom HLSL use member-call form
Author `MyDI.SampleFoo(uv, slice, mip, OutValue)` — **not** the post-rewrite symbol
`SampleFoo_MyDI(...)`. The translator only rewrites tokens beginning with `<DIPinName>.`.
Writing the underscore form directly is an "undeclared identifier" error. DI functions are
`void` with `out` parameters.

### 5c. Inner-graph names differ from stage-level names
| Stage-level (stack) view | Inner-graph name |
|--------------------------|------------------|
| `Particles.Foo` (output) | `Output.Module.Foo` (write via Map Set) |
| `User.Bar` (value, bound at stack) | `Module.Bar` (read via Map Get) |
| `User.MyDI` (DI, bound at stack) | `Module.MyDI` (must be `Module.*`) |
| `Particles.Baz` (existing attr) | `Particles.Baz` (direct read, no `Module.`) |

### 5d. Add inputs the correct way (auto Map Get/Set)
Adding a pin directly on the CustomHlsl node's signature does **not** wire the parameter map.
Instead, in the module's inner graph, use the left **Parameters panel → Module Inputs `+`**
(value or Data Interface type) — the editor auto-creates a Map Get reading `Module.<name>` and
wires it. **Module Outputs `+`** auto-creates a Map Set writing `Output.Module.<name>`.

### 5e. Editing HLSL needs a graph ChangeId bump to recompile
Writing the HLSL string via a direct property write leaves the graph's ChangeId unchanged, so the
compiler reuses a stale digest and silently ignores the edit. The `set_module_hlsl` /
scratch-pad ops handle this (bump ChangeId + refresh) — but if an HLSL edit "doesn't take,"
suspect a missing recompile/ChangeId bump.

### 5f. Module name is reserved after removal
Removing a scratch-pad module leaves its FunctionScript name reserved in asset metadata. Re-adding
a module with the **same name** can fail silently. **Use a new name after a removal.**

---

## 6. Making a stage run every frame (live spectrum / continuous regen)

To make normally-spawn-time math live-tunable, change the stage's execute behavior to run every
frame: `set_stage_execute_behavior` → `Always` (read current with `get_stage_properties`).

**Caveat:** if that stage consumes a **per-execution random** input, running it per frame will
"boil" (the result re-rolls every frame). Replace the random draw with a **deterministic per-texel
hash** (e.g. PCG-style integer hash of texel+index → Box-Muller → gaussians) so values are
identical every frame but statistically equivalent. Then the stage can run continuously and all
its inputs become smoothly tunable with no reinit/pop.

---

## 7. Blueprint ↔ Niagara integration (particle readback)

A common pattern: a GPU sim exports per-particle data back to a Blueprint via
`ExportParticleDataToBlueprint`.

```
BP component
  ├─ on begin: find the NiagaraComponent, then
  │     SetVariableObject("User.<HandlerParam>", self)   ← registers the callback
  ├─ each tick: SetNiagaraArrayVector(comp, "User.<Points>", positions)  ← DI array, set every tick
  └─ implements INiagaraParticleCallbackHandler:
        "Event Receive Particle Data" fires with Data : TArray<BasicParticleData>
```

- The callback object param is just a `UObject` user param (verify its exact name with inspect —
  it varies by asset; do not assume `User.CallbackHandler`).
- `BasicParticleData` carries `Position`, `Velocity`, `Size` (one float) per particle — repurpose
  these channels for whatever you export.
- **1-frame latency:** the GPU sim reads `User.<Points>` next frame, so exported data lags by one
  tick. Expected and usually fine at 60 fps.
- Use `target_object: "self"` on the `SetVariableObject` node (in `add_nodes`) to auto-wire the
  current component as the Object argument.

---

## 8. Render-target readback / sampling

GPU sims commonly write attributes into `TextureRenderTarget2DArray` slices (one slice per
cascade/LOD). To verify output:

`sample_render_target_2d_array { rt_path, u, v, slice }` returns RGBA floats. Requires the sim to
have ticked (PIE running, or the system ticked in-editor). **If two samples taken a moment apart
are bit-identical, the sim isn't ticking** — check realtime/PIE before concluding a set was a no-op.

Channel layout is asset-specific; read the system's writing module (`get_module_source`) to learn
which channel holds what before interpreting samples.

---

## 9. Known limitations / gaps (see `_gaps.md`)

- **No Vector4 in `set_niagara_variable`** — float/int/bool/vec3/color only. Per-cascade Vector4
  params (amplitude, choppiness, etc.) can't be set through it.
- **Silent no-op on unknown param name** — no validation/error (§4b).
- **`set_system_user_param` cannot set object/DI references** in some builds — set those in the
  Niagara editor, or via `configure_di_parameter` for DI bindings.
- **`rebuild_scratchpad_inner_graph` `inputs[]`/`outputs[]` are ignored** — add typed pins via
  `add_scratchpad_module_param` (or in-editor Parameters panel), not through that op.
- **Inner-graph Map Get/Set parameter names and a CustomHlsl signature are not fully read back** by
  `get_module_source` — verify structural edits in-editor.
- **`dump_stage_graph` output overflows the response limit** — spilled to a file with very long
  lines (read by character range).
- **Inherited (italic) modules** can't be removed via `remove_module` — remove in-editor.

---

## 10. Quick checklist

1. `niagara_query inspect` → confirm exact `User.*` names, types, `is_data_interface`.
2. Scalar value? → `set_niagara_variable`. DI/array? → `SetNiagaraArrayVector` (Blueprint).
3. Need it live? → confirm a per-frame stage reads it (`dump_stage_graph`), else expect a reinit.
4. After any set → **read back** (RT sample / inspect / visible change). Success ≠ effect.
5. Custom HLSL with a DI → FunctionScript module, DI as `Module.*` input, member-call form, bind in stack.
6. After structural edits → `compile` and confirm zero errors in the output log.
