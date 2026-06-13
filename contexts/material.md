# Unreal Engine 5.7 Material System Context

This context is automatically loaded when working with Material MCP tools.

## Core Classes

### UMaterialInterface

Base class for all materials. Cannot be instantiated directly.

```cpp
// Common material interface methods
UMaterialInterface* Material = LoadObject<UMaterialInterface>(nullptr, TEXT("/Game/Materials/MyMaterial"));

// Get material parameters
TArray<FMaterialParameterInfo> ScalarParams;
TArray<FGuid> ScalarGuids;
Material->GetAllScalarParameterInfo(ScalarParams, ScalarGuids);

TArray<FMaterialParameterInfo> VectorParams;
TArray<FGuid> VectorGuids;
Material->GetAllVectorParameterInfo(VectorParams, VectorGuids);

TArray<FMaterialParameterInfo> TextureParams;
TArray<FGuid> TextureGuids;
Material->GetAllTextureParameterInfo(TextureParams, TextureGuids);
```

### UMaterial

The base material with shader graph. Created in the Material Editor.

### UMaterialInstance

Base class for material instances. Overrides parent material parameters.

### UMaterialInstanceConstant

Editor-time material instance. Parameters are set and saved with the asset.

```cpp
// Create a material instance constant
UMaterialInstanceConstantFactoryNew* Factory = NewObject<UMaterialInstanceConstantFactoryNew>();
Factory->InitialParent = ParentMaterial;

UMaterialInstanceConstant* MatInst = Cast<UMaterialInstanceConstant>(
    Factory->FactoryCreateNew(
        UMaterialInstanceConstant::StaticClass(),
        Package,
        FName(*AssetName),
        RF_Public | RF_Standalone,
        nullptr,
        GWarn
    )
);

// Set parameters (editor-only methods)
MatInst->SetScalarParameterValueEditorOnly(FName("Roughness"), 0.5f);
MatInst->SetVectorParameterValueEditorOnly(FName("BaseColor"), FLinearColor(1, 0, 0, 1));
MatInst->SetTextureParameterValueEditorOnly(FName("Albedo"), AlbedoTexture);

// Apply and save
MatInst->PostEditChange();
MatInst->MarkPackageDirty();
```

### UMaterialInstanceDynamic

Runtime material instance. Parameters can be changed at runtime.

```cpp
// Create a dynamic material instance (runtime)
UMaterialInstanceDynamic* DynMat = UMaterialInstanceDynamic::Create(ParentMaterial, Outer);

// Set parameters (runtime methods)
DynMat->SetScalarParameterValue(FName("Emissive"), 2.0f);
DynMat->SetVectorParameterValue(FName("Color"), FLinearColor::Green);
DynMat->SetTextureParameterValue(FName("Texture"), MyTexture);
```

## Material Parameter Types

### Scalar Parameters

```cpp
// Get scalar parameter value
float Value;
if (MatInst->GetScalarParameterValue(FName("Roughness"), Value))
{
    UE_LOG(LogTemp, Log, TEXT("Roughness: %f"), Value);
}

// Set scalar parameter (editor-only for MIC)
MatInst->SetScalarParameterValueEditorOnly(FName("Roughness"), 0.3f);
```

### Vector Parameters

Used for colors (FLinearColor) or 4-component vectors.

```cpp
// Get vector parameter value
FLinearColor Color;
if (MatInst->GetVectorParameterValue(FName("BaseColor"), Color))
{
    UE_LOG(LogTemp, Log, TEXT("Color: R=%f G=%f B=%f A=%f"), Color.R, Color.G, Color.B, Color.A);
}

// Set vector parameter (editor-only for MIC)
MatInst->SetVectorParameterValueEditorOnly(FName("BaseColor"), FLinearColor(0.8f, 0.2f, 0.1f, 1.0f));
```

### Texture Parameters

```cpp
// Get texture parameter value
UTexture* Texture = nullptr;
if (MatInst->GetTextureParameterValue(FName("Albedo"), Texture))
{
    UE_LOG(LogTemp, Log, TEXT("Texture: %s"), *Texture->GetName());
}

// Set texture parameter (editor-only for MIC)
UTexture2D* NewTexture = LoadObject<UTexture2D>(nullptr, TEXT("/Game/Textures/MyTexture"));
MatInst->SetTextureParameterValueEditorOnly(FName("Albedo"), NewTexture);
```

## MCP Tool: material

Material instance creation and parameter management.

### Operations

| Operation | Description | Required Params |
|-----------|-------------|-----------------|
| `create_material_instance` | Create new MaterialInstanceConstant | asset_name, parent_material |
| `set_material_parameters` | Set params on an instance (override) OR a base material (defaults, recompiles+saves) | material_path, parameters |
| `set_skeletal_mesh_material` | Assign material to skeletal mesh slot | skeletal_mesh_path, material_path |
| `set_actor_material` | Assign material to an actor's mesh component | actor_name, material_path |
| `get_material_info` | Get material details and parameters | asset_path |
| `set_expression_value` | Edit a constant node (Constant/Constant2/3/4Vector) on a base material by node_id | material_path, node_id, value |
| `repair_expression_ids` | Force colliding node ids (MaterialExpressionGuids) unique | material_path |

### Example Usage

```json
// Create a material instance with parameters
{
  "operation": "create_material_instance",
  "asset_name": "MI_Character_Red",
  "parent_material": "/Game/Materials/M_Character_Base",
  "package_path": "/Game/Materials/Characters/",
  "parameters": {
    "scalars": {
      "Roughness": 0.4,
      "Metallic": 0.0
    },
    "vectors": {
      "BaseColor": {"r": 1.0, "g": 0.2, "b": 0.1, "a": 1.0}
    },
    "textures": {
      "Normal": "/Game/Textures/T_Character_Normal"
    }
  }
}

// Update parameters on existing material instance (override)
{
  "operation": "set_material_parameters",
  "material_path": "/Game/Materials/Characters/MI_Character_Red",
  "parameters": {
    "scalars": {
      "Roughness": 0.6
    },
    "vectors": {
      "EmissiveColor": {"r": 0, "g": 1, "b": 0, "a": 1}
    }
  }
}

// Set parameter DEFAULTS on a BASE UMaterial (material_path = a base material, not an instance).
// Sets the default on the parameter expression nodes, recompiles, and SAVES. Affects ALL instances.
// Names must already exist as parameters (use get_material_info to list them). Also accepts
// "static_switches": {"Name": true}. Wrong type or unknown name → hard error (no silent no-op).
{
  "operation": "set_material_parameters",
  "material_path": "/Game/Materials/M_Character_Base",
  "parameters": {
    "scalars": { "FoamTextureSize": 999 },
    "vectors": { "CascadeSizes": {"r": 1, "g": 2, "b": 3, "a": 4} },
    "static_switches": { "UseDetailNormal": true }
  }
}

// Edit a CONSTANT expression node on a base material (Phase 2b). node_id is the 'id' GUID from
// get_material_info's connected_inputs tree. value = number for Constant, {r,g,b,a} for the vector
// constants. Recompiles + saves. For PARAMETER nodes use set_material_parameters (by name) instead.
{
  "operation": "set_expression_value",
  "material_path": "/Game/OceanWater/Materials/M_PreviewOceanWater",
  "node_id": "8D070E4C442EBD3CDE2584BDC4CFDC49",
  "value": {"r": 0.05, "g": 0.12, "b": 0.16}
}

// Node ids come from MaterialExpressionGuid, which can COLLIDE between distinct nodes (copy-paste
// artifact). get_material_info reports has_duplicate_node_ids:true and set_expression_value REFUSES
// an ambiguous id. Fix by making every node id unique, then re-read:
{
  "operation": "repair_expression_ids",
  "material_path": "/Game/OceanWater/Materials/M_PreviewOceanWater"
}

// Set material on skeletal mesh slot
{
  "operation": "set_skeletal_mesh_material",
  "skeletal_mesh_path": "/Game/Characters/SK_Character",
  "material_slot": 0,
  "material_path": "/Game/Materials/Characters/MI_Character_Red"
}

// Set material on an actor's mesh component (StaticMesh, SkeletalMesh, etc.)
{
  "operation": "set_actor_material",
  "actor_name": "Cube",
  "material_path": "/Game/Materials/Characters/MI_Character_Red",
  "material_slot": 0
}

// Get material information
{
  "operation": "get_material_info",
  "asset_path": "/Game/Materials/Characters/MI_Character_Red"
}
```

### Response Format

```json
// create_material_instance response
{
  "asset_path": "/Game/Materials/Characters/MI_Character_Red",
  "asset_name": "MI_Character_Red",
  "parent_material": "/Game/Materials/M_Character_Base",
  "saved": true
}

// set_actor_material response
{
  "actor": "Cube",
  "component": "StaticMeshComponent0",
  "component_type": "StaticMeshComponent",
  "slot": 0,
  "old_material": "WorldGridMaterial",
  "new_material": "MI_Character_Red"
}

// get_material_info response — full introspection for BASE materials AND instances
{
  "name": "MI_Character_Red",
  "path": "/Game/Materials/Characters/MI_Character_Red",
  "class": "MaterialInstanceConstant",
  "is_instance": true,
  "parent": "/Game/Materials/M_Character_Base",

  // Rendering properties (resolved through instances too)
  "blend_mode": "Opaque",                 // Opaque|Masked|Translucent|Additive|Modulate|AlphaComposite|...
  "shading_models": ["DefaultLit"],       // array — e.g. ["SingleLayerWater"], ["Unlit"]
  "two_sided": false,
  "material_domain": "Surface",           // Surface|DeferredDecal|LightFunction|Volume|PostProcess|UI|...
  "usage_flags": ["StaticMesh"],          // only the enabled MATUSAGE_* flags (prefix stripped)
  "use_material_attributes": false,

  // Parameters: DEFAULTS for a base UMaterial, OVERRIDES for an instance
  "scalar_parameters": { "Roughness": 0.4, "Metallic": 0.0 },
  "vector_parameters": { "BaseColor": {"r": 1.0, "g": 0.2, "b": 0.1, "a": 1.0} },
  "texture_parameters": { "Normal": "/Game/Textures/T_Character_Normal" },
  "static_switch_parameters": { "UseDetailNormal": true },

  // Graph introspection (BASE materials only — depth-bounded ~3). Answers "what drives this output."
  "connected_inputs": {
    "BaseColor": { "class": "MaterialExpressionConstant3Vector", "caption": "0.8,0.2,0.1",
                   "inputs": { /* nested expressions w/ class, caption, parameter name */ } }
    // ...Metallic, Specular, Roughness, Normal, EmissiveColor, Opacity, Refraction, WorldPositionOffset, ...
  },
  // Custom-output nodes (only present if any) — Single Layer Water scatter/absorption lives here
  "custom_outputs": [
    { "class": "MaterialExpressionSingleLayerWaterMaterialOutput",
      "name": "SingleLayerWaterMaterial",
      "inputs": { "ScatteringCoefficients": { /* expr tree */ }, "AbsorptionCoefficients": { /* expr tree */ } } }
  ]
}
```

**Reading water/scatter materials:** a node like `MF_Scattering` driving `EmissiveColor` with a dark
`DeepColor` constant is a common cause of "goes black with depth" on an *opaque DefaultLit* surface —
don't assume Single Layer Water just because it's water. Check `shading_models` + `connected_inputs`.

## Skeletal Mesh Materials

### Accessing Material Slots

```cpp
USkeletalMesh* SkeletalMesh = LoadObject<USkeletalMesh>(nullptr, TEXT("/Game/Characters/SK_Character"));

// Get material array
TArray<FSkeletalMaterial>& Materials = SkeletalMesh->GetMaterials();

// Access slot info
for (int32 i = 0; i < Materials.Num(); ++i)
{
    FSkeletalMaterial& Mat = Materials[i];
    UE_LOG(LogTemp, Log, TEXT("Slot %d: Name=%s, Material=%s"),
        i,
        *Mat.MaterialSlotName.ToString(),
        Mat.MaterialInterface ? *Mat.MaterialInterface->GetName() : TEXT("None"));
}

// Set material on slot
Materials[0].MaterialInterface = NewMaterial;
SkeletalMesh->PostEditChange();
SkeletalMesh->MarkPackageDirty();
```

### FSkeletalMaterial Structure

```cpp
struct FSkeletalMaterial
{
    UMaterialInterface* MaterialInterface;  // The material assigned to this slot
    FName MaterialSlotName;                  // Name of the slot (from import)
    FMeshUVChannelInfo UVChannelData;        // UV channel mapping info
};
```

## Common Patterns

### Creating Material Variants

```cpp
// Load parent material
UMaterialInterface* Parent = LoadObject<UMaterialInterface>(nullptr, TEXT("/Game/Materials/M_Base"));

// Create instance with unique name
FString PackagePath = TEXT("/Game/Materials/Variants/");
FString AssetName = TEXT("MI_Variant_01");

UPackage* Package = CreatePackage(*(PackagePath + AssetName));

UMaterialInstanceConstantFactoryNew* Factory = NewObject<UMaterialInstanceConstantFactoryNew>();
Factory->InitialParent = Parent;

UMaterialInstanceConstant* Variant = Cast<UMaterialInstanceConstant>(
    Factory->FactoryCreateNew(UMaterialInstanceConstant::StaticClass(), Package, FName(*AssetName),
        RF_Public | RF_Standalone, nullptr, GWarn)
);

// Customize
Variant->SetVectorParameterValueEditorOnly(FName("Tint"), FLinearColor::Blue);

// Save
FAssetRegistryModule::AssetCreated(Variant);
Package->MarkPackageDirty();
```

### Batch Material Assignment

```cpp
// Assign same material to all slots
for (FSkeletalMaterial& Mat : SkeletalMesh->GetMaterials())
{
    Mat.MaterialInterface = DefaultMaterial;
}
SkeletalMesh->PostEditChange();
```

## Parameter Naming Conventions

Common material parameter names:
- `BaseColor` - Main albedo color (vector)
- `Roughness` - Surface roughness (scalar, 0-1)
- `Metallic` - Metallic amount (scalar, 0-1)
- `Specular` - Specular intensity (scalar)
- `Normal` - Normal map (texture)
- `Albedo` / `Diffuse` - Base color texture (texture)
- `EmissiveColor` - Emissive color (vector)
- `EmissiveIntensity` - Emissive strength (scalar)
- `Opacity` - Transparency (scalar, 0-1)

## Default Asset Paths

- Material instances: `/Game/Materials/`
- Character materials: `/Game/Materials/Characters/`

Use `package_path` parameter to customize output location.
