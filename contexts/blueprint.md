# Unreal Engine 5.7 Blueprint Graph Context

This context is automatically loaded when working with Blueprint manipulation tools.

## Core Classes

### UBlueprint
Base class for all Blueprint assets. Key properties:
- `GeneratedClass` - The UClass generated from this blueprint
- `ParentClass` - The class this blueprint inherits from
- `BlueprintType` - BPTYPE_Normal, BPTYPE_Const, BPTYPE_MacroLibrary, etc.

### UEdGraph
Container for graph nodes. Every Blueprint has multiple graphs:
- **EventGraph** - Main execution graph
- **ConstructionScript** - Called on actor construction
- **Function graphs** - User-defined functions
- **Macro graphs** - Reusable node sequences

### UEdGraphNode
Base class for all graph nodes. Key members:
- `NodeGuid` - Unique identifier
- `NodePosX`, `NodePosY` - Position in graph
- `Pins` - Array of UEdGraphPin

### UEdGraphPin
Connection point on a node:
- `PinName` - Identifier
- `PinType` - FEdGraphPinType with category (PC_Exec, PC_Boolean, PC_Int, PC_Real, etc.)
- `Direction` - EGPD_Input or EGPD_Output
- `LinkedTo` - Array of connected pins
- `DefaultValue` - Default value as string

## UK2Node Hierarchy

```
UK2Node (Blueprint node base)
├── UK2Node_CallFunction      - Function calls
├── UK2Node_VariableGet       - Get variable value
├── UK2Node_VariableSet       - Set variable value
├── UK2Node_Event             - Event nodes (BeginPlay, Tick)
├── UK2Node_IfThenElse        - Branch node
├── UK2Node_MacroInstance     - Macro usage
├── UK2Node_Composite         - Collapsed graph
└── UK2Node_FunctionEntry     - Function entry point
```

## FBlueprintEditorUtils

Utility class for Blueprint manipulation. Key functions:

```cpp
// Find blueprint from graph
UBlueprint* BP = FBlueprintEditorUtils::FindBlueprintForGraph(Graph);

// Add member variable
FBlueprintEditorUtils::AddMemberVariable(
    Blueprint,
    TEXT("MyVariable"),
    FEdGraphPinType(UEdGraphSchema_K2::PC_Float)
);

// Remove member variable
FBlueprintEditorUtils::RemoveMemberVariable(Blueprint, TEXT("MyVariable"));

// Mark blueprint as modified
FBlueprintEditorUtils::MarkBlueprintAsModified(Blueprint);

// Compile blueprint
FKismetEditorUtilities::CompileBlueprint(Blueprint);

// Add new function graph
UEdGraph* FuncGraph = FBlueprintEditorUtils::CreateNewGraph(
    Blueprint,
    TEXT("MyFunction"),
    UEdGraph::StaticClass(),
    UEdGraphSchema_K2::StaticClass()
);
```

## Pin Type Categories (PC_*)

| Category | Description | C++ Type |
|----------|-------------|----------|
| `PC_Exec` | Execution pin (white) | N/A |
| `PC_Boolean` | Bool pin (red) | bool |
| `PC_Byte` | Byte pin | uint8 |
| `PC_Int` | Integer pin (cyan) | int32 |
| `PC_Int64` | 64-bit integer | int64 |
| `PC_Real` | Float/Double (green) | float/double |
| `PC_Name` | FName pin | FName |
| `PC_String` | FString pin (magenta) | FString |
| `PC_Text` | FText pin (pink) | FText |
| `PC_Struct` | Struct pin | UScriptStruct* |
| `PC_Object` | Object reference (blue) | UObject* |
| `PC_Class` | Class reference (purple) | UClass* |
| `PC_SoftObject` | Soft object ref | TSoftObjectPtr |
| `PC_SoftClass` | Soft class ref | TSoftClassPtr |
| `PC_Enum` | Enum value | UEnum* |
| `PC_Wildcard` | Any type (grey) | varies |

## Creating Nodes Programmatically

```cpp
// Spawn a function call node
UK2Node_CallFunction* CallNode = NewObject<UK2Node_CallFunction>(Graph);
CallNode->FunctionReference.SetExternalMember(
    GET_FUNCTION_NAME_CHECKED(UKismetMathLibrary, Add_FloatFloat),
    UKismetMathLibrary::StaticClass()
);
CallNode->AllocateDefaultPins();
Graph->AddNode(CallNode, false, false);
CallNode->NodePosX = 200;
CallNode->NodePosY = 100;

// Spawn a variable get node
UK2Node_VariableGet* VarGet = NewObject<UK2Node_VariableGet>(Graph);
VarGet->VariableReference.SetSelfMember(FName("MyVariable"));
VarGet->AllocateDefaultPins();
Graph->AddNode(VarGet, false, false);
```

## Connecting Pins

```cpp
// Find pins by name
UEdGraphPin* OutputPin = SourceNode->FindPin(TEXT("ReturnValue"));
UEdGraphPin* InputPin = TargetNode->FindPin(TEXT("A"));

// Connect pins
if (OutputPin && InputPin)
{
    const UEdGraphSchema* Schema = Graph->GetSchema();
    Schema->TryCreateConnection(OutputPin, InputPin);
}
```

## MCP Blueprint Operations

All blueprint operations are accessible via `unreal_ue(domain="blueprint", operation="...", params={...})`.
The router automatically routes to the correct backend tool based on the operation.

### Modify operations (→ `blueprint_modify`)

Auto-compiles the Blueprint after changes.

| Operation | Description | Key Params |
|-----------|-------------|------------|
| `create` | Create a new Blueprint | `package_path`, `blueprint_name`, `parent_class`, `blueprint_type` |
| `add_variable` | Add member variable | `blueprint_path`, `variable_name`, `variable_type` |
| `remove_variable` | Remove member variable | `blueprint_path`, `variable_name` |
| `add_function` | Create new function graph | `blueprint_path`, `function_name` |
| `remove_function` | Remove function graph | `blueprint_path`, `function_name` |
| `add_node` | Add a single node to a graph | `blueprint_path`, `node_type`, `node_params`, `pos_x`, `pos_y`, `graph_name`, `is_function_graph` |
| `add_nodes` | Batch add nodes with connections | `blueprint_path`, `nodes[]`, `connections[]`, `graph_name`, `is_function_graph` |
| `delete_node` | Remove a node from a graph | `blueprint_path`, `node_id`, `graph_name`, `is_function_graph` |
| `connect_pins` | Wire two pins together | `blueprint_path`, `source_node_id`, `source_pin`, `target_node_id`, `target_pin`, `graph_name`, `is_function_graph` |
| `disconnect_pins` | Break pin connection | `blueprint_path`, `source_node_id`, `source_pin`, `target_node_id`, `target_pin`, `graph_name`, `is_function_graph` |
| `set_pin_value` | Set default value for input pin | `blueprint_path`, `node_id`, `pin_name`, `pin_value`, `graph_name`, `is_function_graph` |

#### Targeting function graphs

All node operations (`add_node`, `add_nodes`, `delete_node`, `connect_pins`, `disconnect_pins`, `set_pin_value`) support two optional params that route to any graph in the Blueprint:

- `graph_name` (string) — the exact name of the graph (e.g. `"SetWeatherState"`, `"UpdateFog"`, `"EventGraph"`)
- `is_function_graph` (bool) — `true` to search `FunctionGraphs`, `false` (default) to search `UbergraphPages` (EventGraph)

When omitted, defaults to the first EventGraph (same as before).

```json
// Add a node inside the SetWeatherState function graph
{
  "domain": "blueprint",
  "operation": "add_node",
  "params": {
    "blueprint_path": "/Game/BP_WeatherSystem",
    "node_type": "CallFunction",
    "graph_name": "SetWeatherState",
    "is_function_graph": true,
    "node_params": { "function": "UpdateFog" },
    "pos_x": 600,
    "pos_y": 0
  }
}
```

#### Calling Blueprint's own functions (self-call)

`CallFunction` nodes can call user-defined functions on the same Blueprint without specifying `target_class`. The tool searches (in order):
1. `KismetSystemLibrary` — Print String, Line Trace, etc.
2. `KismetMathLibrary` — math operations
3. `GameplayStatics` — Get All Actors Of Class, etc.
4. **Blueprint's own function graphs** (self-call via `FunctionReference.SetSelfMember`)
5. Blueprint's generated class (inherited compiled functions)

```json
// Call UpdateFog (defined on the same Blueprint) — no target_class needed
{ "node_type": "CallFunction", "node_params": { "function": "UpdateFog" } }

// Call GetAllActorsOfClass from GameplayStatics
{ "node_type": "CallFunction", "node_params": { "function": "GetAllActorsOfClass", "target_class": "GameplayStatics" } }
```

#### Node IDs accepted by modify operations

`connect_pins`, `delete_node`, and `set_pin_value` accept two forms of `node_id`:
- **MCP-generated ID** — returned by `add_node`/`add_nodes` (e.g. `"CallFunction_UpdateFog_42"`)
- **Node GUID** — returned by `blueprint_query` `get_graph` operation (e.g. `"A1B2C3D4-..."`)

Always prefer using the GUID from `blueprint_query` when wiring connections to pre-existing nodes.

### Query operations (→ `blueprint_query`)

Read-only. Use `list` first to discover Blueprints, then `inspect` or `get_graph` for details.

| Operation | Description | Key Params |
|-----------|-------------|------------|
| `list` | Find Blueprints with optional filters | `path_filter`, `type_filter`, `name_filter`, `limit` |
| `inspect` | Get detailed Blueprint info (variables, functions, parent class) | `blueprint_path`, `include_variables`, `include_functions`, `include_graphs` |
| `get_graph` | Get graph structure (node count, events, graph names) | `blueprint_path` |

## Compilation

```cpp
// Full recompile
FKismetEditorUtilities::CompileBlueprint(Blueprint);

// Check for compile errors
if (Blueprint->Status == BS_Error)
{
    // Blueprint has errors - check Blueprint->Message
}

// Mark dirty (needs save)
Blueprint->MarkPackageDirty();
```

## Best Practices

1. **Always call AllocateDefaultPins()** after creating nodes
2. **Use Schema->TryCreateConnection()** for type-safe connections
3. **MarkBlueprintAsModified()** after any changes
4. **Compile after modifications** to validate changes
5. **Check pin compatibility** before connecting (use Schema->ArePinsCompatible)
