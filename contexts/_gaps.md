# UnrealClaude MCP — Known Limitations & Read-Back Gaps

A consolidated inventory of tool limitations, silent failures, and read-back gaps observed when
driving the Unreal Editor through these MCP tools. Use it to set expectations *before* an
operation — especially the **read-back gaps**, where a tool can change state but can't confirm
the result.

> **Behavior varies by plugin/bridge version.** Treat each entry as "verify against your build,"
> not a permanent guarantee. Status tags: **VERIFIED** (observed first-hand) /
> **WATCH** (build-dependent or partially addressed in some versions).

**Related contexts:** `niagara.md`, `blueprint.md`, `material.md`, `level-and-bsp.md`.

---

## Silent failures (succeed but do nothing) — the dangerous class

| Area | What happens | Avoid / detect by |
|------|--------------|-------------------|
| **Niagara variable set** | Setting a `User.*` name that doesn't exist reports success and changes nothing. | Take exact names from `niagara_query inspect`; **read back** (RT sample / visible change). VERIFIED |
| **`set_property` on a non–Instance-Editable Blueprint var** | No effect; no error. | Enable *Instance Editable* on the variable first. VERIFIED |
| **`set_property` on baked BSP visibility** | `bHidden` on a Brush doesn't remove the rendered surface. | Delete the brush / rebuild geometry. See `level-and-bsp.md`. VERIFIED |
| **Display-category vs real name** | A param shown as `Category ▸ Name` is set with `User.Name`, not `Category.Name`. | Inspect for the real name. See `niagara.md`. VERIFIED |
| **Interface function inputs** | Inputs added to an interface function can be dropped silently in some builds. | Query the function back after adding inputs to confirm they stuck. WATCH |

A silent no-op is the most common false diagnosis — it gets mistaken for a deeper tool/engine bug.
**After any state-changing call where success ≠ confirmation, read the state back.**

---

## Read-back gaps (can write, can't verify via MCP)

| Tool | Gap | Workaround |
|------|-----|------------|
| Niagara inner graph | A scratch-pad module's Map Get/Set parameter names and a CustomHlsl signature aren't fully read back. | Verify structural inner-graph edits in-editor. WATCH |
| Blueprint graph | `get_nodes` returns node IDs but not full pin wiring. | Use `get_node_pins` on the specific node before wiring. VERIFIED |

**Rule:** before performing structural work (Niagara inner graphs, Blueprint wiring, material
graphs), confirm a read-back path exists. If it doesn't, say so up front — don't write blind.

---

## Type / capability gaps

| Tool / domain | Limitation | Workaround |
|---------------|-----------|------------|
| `set_niagara_variable` | No **Vector4** type (float/int/bool/vec3/color only). A "color" set writes an `FLinearColor` a `Vector4f` won't read. | Set Vector4 params in the Niagara editor. WATCH |
| `asset` domain | Cannot create **UserDefinedEnum / UserDefinedStruct / DataTable**, and cannot create **Blueprint classes**. | Use the `blueprint` domain for BP classes; create enums/structs/data tables in-editor. A `byte` var + comment can stand in for an enum. VERIFIED |
| `blueprint` regular functions | Parameters can't be added to a non-interface Blueprint **function graph** via MCP (input-add is interface-only in some builds). | Design functions to read member variables, or add parameters in-editor. WATCH |
| `blueprint add_variable` `category` | Accepted but ignored — variables land in "Default". | Move to the right category in-editor. WATCH |
| `blueprint` `BlueprintPure` instance methods | Pure functions on `AActor` (e.g. `GetActorLocation/Rotation/Transform`) may not resolve via `target_class: "Actor"` — only `BlueprintCallable` resolves for non-library classes in some builds. Library functions (Kismet*) work. | Use a `BlueprintCallable` alternative, or add the node in-editor. WATCH |
| `blueprint` SCS component get | Components added to the construction script aren't in `NewVariables`; `add_node VariableGet "<Component>"` may fail. | Add a VariableGet for the component separately and wire the `self` pin, or add in-editor. WATCH |
| `blueprint add_variable` object refs | Object-reference variable types *are* supported — pass the C++ class name without the `U` prefix, with or without `*` (e.g. `"SkeletalMeshComponent"`, `"MaterialInstanceDynamic*"`, `"Actor"`). | — (this one works; noted to prevent needless avoidance.) VERIFIED |
| Niagara scratch-pad rebuild | `inputs[]` / `outputs[]` on the inner-graph rebuild op are ignored. | Add typed pins via the dedicated add-param op or the editor Parameters panel. See `niagara.md`. WATCH |
| Niagara inherited modules | Modules inherited from a parent emitter (shown italic) can't be removed via MCP. | Remove in-editor. VERIFIED |
| `set_system_user_param` (Niagara) | Can't set object/Data-Interface references in some builds. | Set object/DI refs in-editor, or use the DI-configure op for DI bindings. WATCH |

---

## Large-output / tooling friction

| Tool | Issue | Handling |
|------|-------|----------|
| Niagara `dump_stage_graph` | Output can exceed the response size limit and is spilled to a file with very long single lines. | Read by character range / regex, not line offset/limit. VERIFIED |
| Output log | Full logs are huge. | Always filter (Error / Warning / a log category) or tail a small number of lines. VERIFIED |

---

## Availability varies by build

Some tools are not exposed in every build configuration:

- **Console-command** and **arbitrary-script** execution tools may be disabled (often for safety).
  If you need to toggle a cvar or run an editor script and the tool isn't present, fall back to
  domain tools or ask the user to run it in-editor. WATCH

---

## A note on the tool docs themselves

Tool descriptions and example strings live in the plugin's C++ `GetInfo()`. They can drift from
reality — an example parameter value in a docstring is **not** a guarantee the value is valid for
your asset. When a documented example fails, prefer values read live from the asset
(`*_query inspect`, `get_node_pins`, etc.) over the docstring example, and consider the docstring a
candidate for a fix.
