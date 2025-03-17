# GENESIS-UNREAL TECHNICAL INTEGRATION
## Detailed Technical Specification

### 1. INTEGRATION ARCHITECTURE

#### 1.1 System Overview

```
┌─────────────────┐    ┌───────────────────┐    ┌──────────────────┐
│  Genesis Engine  │ ─> │  Data Translator  │ ─> │  Unreal Engine   │
└─────────────────┘    └───────────────────┘    └──────────────────┘
        │                       │                        │
        ▼                       ▼                        ▼
┌─────────────────┐    ┌───────────────────┐    ┌──────────────────┐
│ Physics Results  │ ─> │ Intermediate Data │ ─> │  Visualization   │
└─────────────────┘    └───────────────────┘    └──────────────────┘
```

#### 1.2 Component Roles

- **Genesis Engine**: Handles all physics calculations for wave simulation
- **Data Translator**: Converts physics data to Unreal-compatible formats
- **Unreal Engine**: Renders the simulation data visually

#### 1.3 Data Exchange Formats

- **Height Field Maps**: 32-bit float arrays for water surface displacement
- **Velocity Fields**: 3D vector arrays for water particle movement
- **Foam/Spray Density Maps**: 2D scalar fields for white water effects
- **Metadata**: JSON structures containing simulation parameters

### 2. GENESIS ENGINE CONFIGURATION

#### 2.1 Solver Selection for Wave Simulation

| Phenomenon               | Recommended Solver | Parameters                             |
|--------------------------|--------------------|-----------------------------------------|
| Deep Water Propagation   | Spectral           | `spectral_resolution=1024`             |
| Shallow Water Interaction| SPH                | `particle_count=1M-10M`, `h=0.1-0.5`   |
| Breaking Waves           | MPM + PBD Hybrid   | `grid_resolution=0.1-1m`               |
| Foam Generation          | PBD                | `diffusion_rate=0.05`, `lifetime=3s`   |

#### 2.2 Genesis Output Configuration

```python
# Example Genesis configuration for wave export
genesis.configure_export(
    height_field=True,
    velocity_field=True,
    surface_mesh=True,
    foam_particles=True,
    export_frequency=4,  # Export every 4 simulation steps
    compression="lz4",
    precision="float32"
)
```

#### 2.3 Performance Optimization

- Adaptive time-stepping based on wave activity
- Spatial partitioning for focused computation on active areas
- Multi-GPU distribution for large domain simulations
- Checkpoint system for fault tolerance

### 3. DATA TRANSLATION LAYER

#### 3.1 Core Translation Functions

```cpp
// C++ pseudo-code for height field conversion
void ConvertHeightFieldToUnreal(
    const float* genesisHeightField,
    int width, int height,
    UTexture2D* unrealHeightMap
) {
    // Lock the texture for updating
    uint8* mipData = static_cast<uint8*>(unrealHeightMap->PlatformData->Mips[0].BulkData.Lock(LOCK_READ_WRITE));
    
    // Copy and convert the data
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x++) {
            int index = y * width + x;
            float height = genesisHeightField[index];
            
            // Convert to Unreal's height format and scale
            float unrealHeight = height * HeightScale;
            
            // Set the texture data
            FFloat16Color* pixelData = reinterpret_cast<FFloat16Color*>(mipData);
            pixelData[index].R = FFloat16(unrealHeight);
        }
    }
    
    // Unlock the texture
    unrealHeightMap->PlatformData->Mips[0].BulkData.Unlock();
    unrealHeightMap->UpdateResource();
}
```

#### 3.2 Data Format Specifications

- **Height Field**: 32-bit float R16F textures, normalized to 0-1 range
- **Velocity Field**: RGB16F textures, each channel representing velocity component
- **Displacement Vectors**: RG16F textures for horizontal displacement
- **Foam Density**: 8-bit grayscale textures or particle position arrays

#### 3.3 Coordinate System Transformation

- Genesis: Y-up coordinate system, Z-forward
- Unreal: Z-up coordinate system, X-forward
- Translation matrix for consistent transformation

### 4. UNREAL ENGINE IMPLEMENTATION

#### 4.1 Custom Water System Overview

```
┌─────────────────────┐
│  Water Manager Actor │
├─────────────────────┤
│- Height Map Texture  │
│- Velocity Texture    │
│- Foam Map Texture    │
│- Wave Parameters     │
└─────────────────────┘
         │
         ▼
┌─────────────────────┐     ┌─────────────────────┐
│  Water Surface Mesh  │ ┬─> │   Water Material    │
└─────────────────────┘ │   └─────────────────────┘
                        │            │
                        │            ▼
                        │   ┌─────────────────────┐
                        └─> │  Particle Systems   │
                            └─────────────────────┘
```

#### 4.2 Material Setup

**Base Water Material Parameters:**
- Displacement Map (for height field)
- Normal Map (derived from height field)
- Flow Map (from velocity field)
- Foam Mask (from foam density)
- Subsurface Scattering Profile
- Specular Intensity
- Water Color Gradient (depth-based)

**Material Function: Wave Normal Calculation**
```hlsl
// HLSL code for calculating normals from height field
float3 CalculateNormalFromHeightMap(Texture2D HeightMap, float2 UV, float Strength)
{
    float2 texelSize = 1.0f / HeightMap.GetDimensions();
    
    // Sample neighboring height values
    float h0 = HeightMap.Sample(HeightMap_Sampler, UV).r;
    float h1 = HeightMap.Sample(HeightMap_Sampler, UV + float2(texelSize.x, 0)).r;
    float h2 = HeightMap.Sample(HeightMap_Sampler, UV + float2(-texelSize.x, 0)).r;
    float h3 = HeightMap.Sample(HeightMap_Sampler, UV + float2(0, texelSize.y)).r;
    float h4 = HeightMap.Sample(HeightMap_Sampler, UV + float2(0, -texelSize.y)).r;
    
    // Calculate the normal
    float3 normal;
    normal.x = (h2 - h1) * Strength;
    normal.y = (h4 - h3) * Strength;
    normal.z = 1.0f;
    
    return normalize(normal);
}
```

#### 4.3 Dynamic Mesh Generation

```cpp
// C++ pseudo-code for mesh generation from height field
void UWaveMeshComponent::UpdateMeshFromHeightField(UTexture2D* HeightField)
{
    // Get height field data
    FTexture2DMipMap& Mip = HeightField->PlatformData->Mips[0];
    FByteBulkData& BulkData = Mip.BulkData;
    FFloat16Color* Data = static_cast<FFloat16Color*>(BulkData.Lock(LOCK_READ_ONLY));
    
    // Update vertex positions
    TArray<FDynamicMeshVertex> Vertices;
    TArray<uint32> Indices;
    
    int32 Width = HeightField->GetSizeX();
    int32 Height = HeightField->GetSizeY();
    
    for (int32 Y = 0; Y < Height; Y++)
    {
        for (int32 X = 0; X < Width; X++)
        {
            int32 Index = Y * Width + X;
            float HeightValue = (float)Data[Index].R;
            
            FVector Position = FVector(X * GridSize, Y * GridSize, HeightValue * HeightScale);
            FDynamicMeshVertex Vertex;
            Vertex.Position = Position;
            Vertex.TextureCoordinate = FVector2D((float)X / Width, (float)Y / Height);
            
            Vertices.Add(Vertex);
        }
    }
    
    // Generate indices for triangles
    for (int32 Y = 0; Y < Height - 1; Y++)
    {
        for (int32 X = 0; X < Width - 1; X++)
        {
            int32 Index = Y * Width + X;
            
            Indices.Add(Index);
            Indices.Add(Index + 1);
            Indices.Add(Index + Width);
            
            Indices.Add(Index + 1);
            Indices.Add(Index + Width + 1);
            Indices.Add(Index + Width);
        }
    }
    
    // Unlock height field data
    BulkData.Unlock();
    
    // Update mesh section
    UpdateMeshSection(0, Vertices, Indices);
}
```

#### 4.4 Particle Systems for Foam and Spray

- CPU Particle system for coarse foam representation
- GPU Particle system for detailed spray and mist
- Spawn rate driven by velocity field gradients
- Particle lifetime based on simulation parameters

### 5. REAL-TIME DATA SYNCHRONIZATION

#### 5.1 Data Transfer Protocols

- **Local Rendering**: Direct memory mapping for real-time transfer
- **Network Rendering**: WebSocket or gRPC streaming with LZ4 compression
- **Cached Rendering**: HDF5 or similar scientific data format

#### 5.2 Update Sequence

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Genesis Step 1 │ -> │ Translate Data │ -> │ Unreal Update │
└───────────────┘    └───────────────┘    └───────────────┘
        │                                         │
        ▼                                         ▼
┌───────────────┐                        ┌───────────────┐
│ Genesis Step 2 │                        │ Render Frame 1 │
└───────────────┘                        └───────────────┘
        │                                         │
        ▼                                         ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ Genesis Step 3 │ -> │ Translate Data │ -> │ Unreal Update │
└───────────────┘    └───────────────┘    └───────────────┘
        │                                         │
        ▼                                         ▼
┌───────────────┐                        ┌───────────────┐
│ Genesis Step 4 │                        │ Render Frame 2 │
└───────────────┘                        └───────────────┘
```

#### 5.3 Performance Considerations

- Maintain 1:N ratio of physics updates to render frames
- Implement interpolation for smooth transitions between physics states
- Utilize asynchronous loading for large datasets
- Implement level-of-detail system for distant waves

### 6. IMPLEMENTATION GUIDES

#### 6.1 Genesis Export Setup

```python
# Python code for Genesis export configuration
def configure_wave_simulation_export(simulation, export_path):
    # Configure basic simulation parameters
    simulation.set_domain_size(10000, 10000, 1000)  # 10km x 10km domain with 1km depth
    simulation.set_grid_resolution(10.0)  # 10m base grid resolution
    
    # Configure more detailed resolution for breaking waves
    simulation.add_adaptive_region(
        center=(5000, 5000, 0),
        size=(2000, 2000, 200),
        resolution=1.0  # 1m resolution in breaking zone
    )
    
    # Setup exporters
    height_field_exporter = simulation.add_exporter(
        type="height_field",
        resolution=(1024, 1024),
        path=os.path.join(export_path, "height_fields"),
        format="exr",
        compression="zip",
        frequency=4  # Export every 4 steps
    )
    
    velocity_exporter = simulation.add_exporter(
        type="velocity_field",
        resolution=(1024, 1024, 32),  # 32 depth layers
        path=os.path.join(export_path, "velocity_fields"),
        format="npz",
        frequency=4
    )
    
    foam_exporter = simulation.add_exporter(
        type="particle",
        filter="foam",
        path=os.path.join(export_path, "foam_particles"),
        format="csv",
        attributes=["position", "lifetime", "velocity"],
        frequency=4
    )
    
    # Add metadata export
    metadata_exporter = simulation.add_exporter(
        type="metadata",
        path=os.path.join(export_path, "metadata"),
        format="json",
        frequency=4
    )
    
    return {
        "height_field": height_field_exporter,
        "velocity": velocity_exporter,
        "foam": foam_exporter,
        "metadata": metadata_exporter
    }
```

#### 6.2 Unreal Engine Import Plugin

```cpp
// C++ pseudo-code for Unreal Engine import plugin
class FGenesisImportPlugin : public IModuleInterface
{
public:
    virtual void StartupModule() override
    {
        // Register asset type actions
        HeightFieldActions = MakeShared<FGenesisHeightFieldActions>();
        FAssetToolsModule::GetModule().Get().RegisterAssetTypeActions(HeightFieldActions.ToSharedRef());
        
        // Register custom asset factories
        HeightFieldFactory = MakeShared<FGenesisHeightFieldFactory>();
        FAssetToolsModule::GetModule().Get().RegisterAssetTypeActions(HeightFieldFactory.ToSharedRef());
    }
    
    virtual void ShutdownModule() override
    {
        // Unregister asset type actions
        if (FAssetToolsModule::IsModuleLoaded())
        {
            FAssetToolsModule::GetModule().Get().UnregisterAssetTypeActions(HeightFieldActions.ToSharedRef());
            FAssetToolsModule::GetModule().Get().UnregisterAssetTypeActions(HeightFieldFactory.ToSharedRef());
        }
    }
    
private:
    TSharedPtr<FGenesisHeightFieldActions> HeightFieldActions;
    TSharedPtr<FGenesisHeightFieldFactory> HeightFieldFactory;
};
```

#### 6.3 Runtime Update System

```cpp
// C++ pseudo-code for runtime update system
void AGenesisWaveManager::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // Increment time counter
    CurrentTime += DeltaTime * TimeScale;
    
    // Determine which time frame to use
    float Frame = CurrentTime * UpdateFrequency;
    int32 FrameA = FMath::FloorToInt(Frame);
    int32 FrameB = FMath::CeilToInt(Frame);
    float Alpha = Frame - FrameA;
    
    // Load or retrieve the two frames
    UTexture2D* HeightFieldA = GetHeightFieldForFrame(FrameA);
    UTexture2D* HeightFieldB = GetHeightFieldForFrame(FrameB);
    
    // Interpolate between frames (simplified)
    if (HeightFieldA && HeightFieldB)
    {
        // Set material parameters for interpolation
        WaterMaterialInstance->SetTextureParameterValue("HeightFieldA", HeightFieldA);
        WaterMaterialInstance->SetTextureParameterValue("HeightFieldB", HeightFieldB);
        WaterMaterialInstance->SetScalarParameterValue("InterpolationAlpha", Alpha);
    }
    
    // Update mesh if using dynamic mesh approach
    if (bUseDynamicMesh && WaveMeshComponent)
    {
        WaveMeshComponent->UpdateMeshFromHeightFields(HeightFieldA, HeightFieldB, Alpha);
    }
    
    // Update foam particles
    UpdateFoamParticles(FrameA, FrameB, Alpha);
}
```

### 7. OPTIMIZATION TECHNIQUES

#### 7.1 Level of Detail System

| Distance Range | Height Field Resolution | Mesh Tessellation | Update Frequency |
|----------------|-------------------------|-------------------|------------------|
| 0-500m         | 2048×2048              | High              | Every frame      |
| 500m-2000m     | 1024×1024              | Medium            | Every 2 frames   |
| 2000m-5000m    | 512×512                | Low               | Every 4 frames   |
| >5000m         | 256×256                | Minimum           | Every 8 frames   |

#### 7.2 Memory Management

- Texture streaming for large datasets
- Deferred destruction of unused resources
- Texture atlas for multiple wave sections
- Compressed data formats for network transfer

#### 7.3 Rendering Performance

- View-dependent tessellation
- Distance-based foam particle density
- Pre-computed lighting for static elements
- Adaptive shader complexity

### 8. DEBUGGING AND VALIDATION

#### 8.1 Diagnostic Tools

- Height field visualization with color mapping
- Vector field visualization for water flow
- Data comparison tools for simulation vs. real-world
- Performance measurement and bottleneck identification

#### 8.2 Validation Methods

- Visual comparison with reference footage
- Statistical wave analysis (height distribution, period accuracy)
- Energy conservation verification
- Breaking wave classification accuracy

---

## APPENDICES

### Appendix A: Data Format Specifications

#### A.1 Height Field Format
```
File Extension: .exr
Bit Depth: 32-bit float
Channels: 1 (height only)
Size: Varies by resolution tier
Metadata: Contains simulation parameters
```

#### A.2 Velocity Field Format
```
File Extension: .npz
Content: Multi-dimensional NumPy array
Dimensions: Width × Height × Depth × 3 (xyz components)
Data Type: float32
```

#### A.3 Particle Data Format
```
File Extension: .csv or .bgeo
Columns: position_x, position_y, position_z, velocity_x, velocity_y, velocity_z, lifetime, type
```

#### A.4 Metadata Format
```
File Extension: .json
Contents:
{
  "simulation_time": 123.45,
  "domain_bounds": [0, 10000, 0, 10000, -1000, 0],
  "resolution": [1024, 1024],
  "physical_scale": 10.0,
  "wave_statistics": {
    "significant_height": 2.5,
    "peak_period": 8.0,
    "direction": 240.0
  }
}
```

### Appendix B: Reference Implementation

[Link to GitHub repository with sample code]

---

*Document Version: 1.0*
*Last Updated: [Current Date]* 