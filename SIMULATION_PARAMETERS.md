# GENESIS SIMULATION PARAMETERS
## Technical Configuration Guide

### 1. PHYSICS ENGINE CONFIGURATION

#### 1.1 Core Physics Parameters

| Parameter Group | Parameter Name | Value Range | Recommended Value | Description |
|-----------------|----------------|-------------|-------------------|-------------|
| Wave Generation | `wave_spectrum_type` | Pierson-Moskowitz, JONSWAP, Custom | JONSWAP | Wave energy spectrum model |
| | `significant_wave_height` | 0.1-20.0 m | 1.0-4.0 m | Overall height of wave system |
| | `peak_period` | 1.0-25.0 s | 8.0-12.0 s | Primary wave period |
| | `directional_spreading` | 0.0-1.0 | 0.3 | Angular spread of wave directions |
| Fluid Dynamics | `grid_resolution` | 0.1-10.0 m | 1.0 m | Base resolution of simulation grid |
| | `simulation_depth` | 1.0-1000.0 m | 50.0 m | Maximum depth of simulation domain |
| | `adaptive_timestep` | true/false | true | Dynamically adjust time step for stability |
| | `min_timestep` | 0.001-0.1 s | 0.01 s | Minimum physics time step |
| | `max_timestep` | 0.01-0.5 s | 0.05 s | Maximum physics time step |
| | `cfl_number` | 0.1-1.0 | 0.5 | Courant-Friedrichs-Lewy condition limit |
| Breaking Waves | `breaking_threshold` | 0.5-1.0 | 0.78 | Wave steepness threshold for breaking |
| | `foam_generation` | 0.0-10.0 | 1.0 | Intensity of foam creation during breaking |
| | `foam_dissipation` | 0.0-10.0 | 0.5 | Rate of foam dissipation |
| | `vorticity_enhancement` | 0.0-5.0 | 1.0 | Additional turbulence in breaking waves |
| Environmental | `wind_speed` | 0.0-50.0 m/s | 5.0 m/s | Wind speed at 10m above surface |
| | `wind_direction` | 0-360° | 270° | Wind direction in degrees |
| | `air_density` | 1.0-1.5 kg/m³ | 1.225 kg/m³ | Density of air |
| | `water_density` | 1000-1035 kg/m³ | 1025 kg/m³ | Density of water |
| | `gravity` | 9.7-9.9 m/s² | 9.81 m/s² | Gravitational acceleration |
| | `surface_tension` | 0.0-0.1 N/m | 0.074 N/m | Water surface tension coefficient |

#### 1.2 Advanced Parameters

| Parameter Group | Parameter Name | Value Range | Recommended Value | Description |
|-----------------|----------------|-------------|-------------------|-------------|
| Numerical Solver | `solver_type` | FFT, JONSWAP, PBD, MPM, SPH | Hybrid | Core physics solver algorithm |
| | `fft_resolution` | 128-4096 | 1024 | Resolution for spectral methods |
| | `iteration_count` | 1-100 | 10 | Solver iterations per time step |
| | `relaxation_factor` | 0.1-1.0 | 0.5 | Solver relaxation parameter |
| | `convergence_threshold` | 1e-6-1e-2 | 1e-4 | Solver convergence criterion |
| Computational | `gpu_acceleration` | true/false | true | Enable GPU acceleration |
| | `double_precision` | true/false | false | Use double precision floats |
| | `thread_count` | 1-128 | 16 | Number of CPU threads |
| | `particle_batch_size` | 1024-65536 | 16384 | Particles per GPU batch |
| | `tile_size` | 8-64 | 16 | Spatial tiling for computation |
| Boundary | `boundary_condition` | periodic, absorbing, reflecting | absorbing | Domain boundary behavior |
| | `sponge_layer_width` | 0.0-100.0 m | 50.0 m | Width of absorbing boundary layer |
| | `sponge_strength` | 0.0-1.0 | 0.8 | Intensity of wave absorption at boundary |
| | `enforce_mass_conservation` | true/false | true | Strictly enforce mass conservation |
| | `bottom_friction` | 0.0-1.0 | 0.02 | Seabed friction coefficient |

### 2. DOMAIN CONFIGURATION

#### 2.1 Basic Domain Setup

```json
{
  "domain": {
    "size_x": 5000.0,            // Domain width in meters
    "size_y": 5000.0,            // Domain length in meters
    "size_z": 100.0,             // Domain height in meters
    "origin": [0.0, 0.0, 0.0],   // Domain origin coordinates
    "grid_resolution": 5.0,      // Base grid resolution in meters
    "adaptive_grid": true,       // Enable adaptive grid refinement
    "maximum_refinement": 3,     // Maximum levels of grid refinement
    "refinement_threshold": 0.5  // Height gradient threshold for refinement
  },
  
  "bathymetry": {
    "type": "imported",          // Type: flat, procedural, imported
    "file_path": "bathymetry/malibu_reef.exr",
    "scaling_factor": 1.0,
    "depth_offset": -50.0,       // Offset from water surface (negative = below)
    "smoothing_passes": 2        // Smoothing iterations for imported data
  }
}
```

#### 2.2 Adaptive Regions Configuration

```json
{
  "adaptive_regions": [
    {
      "name": "Breaking Zone",
      "type": "box",
      "center": [2500.0, 2500.0, 0.0],
      "size": [1000.0, 1000.0, 50.0],
      "resolution": 1.0,         // 1 meter resolution in breaking zone
      "priority": 1
    },
    {
      "name": "Surf Zone",
      "type": "box",
      "center": [2500.0, 2300.0, 0.0],
      "size": [1000.0, 200.0, 20.0],
      "resolution": 0.5,         // 0.5 meter resolution in surf zone
      "priority": 2
    },
    {
      "name": "Shoreline",
      "type": "polygon",
      "points": [
        [2000.0, 2000.0, 0.0],
        [3000.0, 2000.0, 0.0],
        [3000.0, 2100.0, 0.0],
        [2000.0, 2100.0, 0.0]
      ],
      "height": 10.0,
      "resolution": 0.25,        // 0.25 meter resolution at shoreline
      "priority": 3
    }
  ]
}
```

#### 2.3 Dynamic Adaptation Criteria

```json
{
  "dynamic_adaptation": {
    "enabled": true,
    "update_frequency": 5,       // Check for readaptation every 5 time steps
    "criteria": [
      {
        "type": "height_gradient",
        "threshold": 0.5,        // Adapt where wave steepness exceeds 0.5
        "target_resolution": 1.0
      },
      {
        "type": "velocity_magnitude",
        "threshold": 2.0,        // Adapt where velocity exceeds 2 m/s
        "target_resolution": 0.5
      },
      {
        "type": "foam_density",
        "threshold": 0.3,        // Adapt where foam density exceeds 0.3
        "target_resolution": 0.5
      }
    ],
    "maximum_regions": 10,       // Maximum number of dynamic regions
    "region_merge_distance": 5.0 // Merge regions closer than 5m
  }
}
```

### 3. WAVE CONFIGURATION

#### 3.1 Wave Spectrum Definition

```json
{
  "wave_spectra": [
    {
      "type": "JONSWAP",
      "significant_wave_height": 2.0,
      "peak_period": 10.0,
      "peak_enhancement": 3.3,
      "direction": 270.0,
      "directional_spreading": 0.3,
      "seed": 12345,
      "component_count": 256
    },
    {
      "type": "Swell",
      "significant_wave_height": 0.8,
      "peak_period": 14.0,
      "peak_enhancement": 6.0,
      "direction": 260.0,
      "directional_spreading": 0.1,
      "seed": 54321,
      "component_count": 128
    }
  ],
  
  "custom_wave_components": [
    {
      "amplitude": 0.5,
      "period": 8.0,
      "direction": 265.0,
      "phase": 0.0
    }
  ]
}
```

#### 3.2 Current Field Integration

```json
{
  "current_field": {
    "type": "imported",          // Type: uniform, imported, procedural
    "file_path": "currents/tidal_current_1.nc",
    "format": "netcdf",
    "scaling_factor": 1.0,
    "rotation_angle": 0.0,
    "interpolation": "bilinear",
    
    // For uniform current:
    "uniform_current": {
      "speed": 1.0,              // m/s
      "direction": 180.0         // degrees
    },
    
    // For procedural current:
    "procedural_current": {
      "type": "rip_current",
      "position": [2500.0, 2200.0],
      "width": 100.0,
      "strength": 2.0,
      "offshore_length": 300.0
    }
  }
}
```

#### 3.3 Wind-Wave Interaction

```json
{
  "wind": {
    "type": "uniform",           // Type: uniform, variable
    "speed": 5.0,                // m/s
    "direction": 270.0,          // degrees
    "gustiness": 0.2,            // Random variation
    "height_reference": 10.0,    // Reference height (m)
    
    // For variable wind:
    "variation": {
      "temporal": {
        "period": 600.0,         // Wind variation period (s)
        "amplitude": 2.0,        // Speed variation (m/s)
        "direction_range": 20.0  // Direction variation (±degrees)
      },
      "spatial": {
        "resolution": [10, 10],  // Grid for spatial variation
        "scale": 1000.0,         // Size of largest features (m)
        "intensity": 0.3         // Intensity of spatial variation
      }
    }
  }
}
```

### 4. SIMULATION EXECUTION

#### 4.1 Time Control

```json
{
  "time_control": {
    "start_time": 0.0,
    "end_time": 300.0,           // Simulate 5 minutes (300 seconds)
    "time_step_method": "adaptive",
    "fixed_time_step": 0.03,     // Used if method is "fixed"
    "min_time_step": 0.01,
    "max_time_step": 0.05,
    "cfl_target": 0.5
  }
}
```

#### 4.2 Simulation Phases

```json
{
  "simulation_phases": [
    {
      "name": "Initialization",
      "duration": 30.0,
      "description": "Wave field initialization",
      "parameters": {
        "damping_factor": 0.3,    // Gradual wave growth
        "wind_ramp_up": true,
        "solver_iterations": 20   // Higher accuracy during init
      }
    },
    {
      "name": "Main Simulation",
      "duration": 240.0,
      "description": "Primary simulation phase",
      "parameters": {
        "damping_factor": 0.0,    // No artificial damping
        "solver_iterations": 10
      }
    },
    {
      "name": "Analysis Window",
      "duration": 30.0,
      "description": "Detailed output for analysis",
      "parameters": {
        "output_frequency": 1,    // Output every frame
        "detail_level": "high"
      }
    }
  ]
}
```

#### 4.3 Performance Configuration

```json
{
  "performance": {
    "compute_device": "GPU",     // GPU or CPU
    "precision": "float32",      // float32 or float64
    "gpu_device_id": 0,          // For multi-GPU systems
    "thread_count": 16,          // CPU threads if using CPU
    "memory_limit": 8192,        // Memory limit in MB
    "tile_size": 16,             // Computational tile size
    "optimization_level": "balanced" // memory, balanced, or speed
  }
}
```

### 5. OUTPUT CONFIGURATION

#### 5.1 Output Formats and Parameters

```json
{
  "output_configuration": {
    "base_output_dir": "results/sim_20230315_001/",
    "output_frequency": 4,       // Save every 4 time steps
    "compression": "lz4",        // none, lz4, zstd
    "compression_level": 3,
    
    "outputs": [
      {
        "type": "height_field",
        "enabled": true,
        "format": "exr",
        "resolution": [1024, 1024],
        "precision": "float32",
        "file_pattern": "height_field_{frame:04d}.exr"
      },
      {
        "type": "velocity_field",
        "enabled": true,
        "format": "npz",
        "dimensions": [1024, 1024, 32],
        "precision": "float32",
        "file_pattern": "velocity_{frame:04d}.npz"
      },
      {
        "type": "normal_map",
        "enabled": true,
        "format": "exr",
        "resolution": [1024, 1024],
        "precision": "float32",
        "file_pattern": "normals_{frame:04d}.exr"
      },
      {
        "type": "foam_particles",
        "enabled": true,
        "format": "csv",
        "max_particles": 1000000,
        "attributes": ["position", "age", "velocity"],
        "file_pattern": "foam_{frame:04d}.csv"
      },
      {
        "type": "mesh",
        "enabled": false,
        "format": "obj",
        "resolution": [512, 512],
        "file_pattern": "mesh_{frame:04d}.obj"
      },
      {
        "type": "wave_metrics",
        "enabled": true,
        "format": "json",
        "metrics": ["significant_height", "peak_period", "breaking_events"],
        "file_pattern": "metrics_{frame:04d}.json"
      }
    ]
  }
}
```

#### 5.2 Analysis and Metrics

```json
{
  "analysis": {
    "enabled": true,
    "realtime": false,
    "regions": [
      {
        "name": "PrimaryBreak",
        "type": "box",
        "center": [2500.0, 2300.0, 0.0],
        "size": [200.0, 200.0, 10.0],
        "metrics": ["wave_height", "breaking_intensity", "peel_angle"]
      },
      {
        "name": "Shoreline",
        "type": "line",
        "start": [2000.0, 2050.0, 0.0],
        "end": [3000.0, 2050.0, 0.0],
        "point_count": 100,
        "metrics": ["runup_height", "wave_period"]
      }
    ],
    "global_metrics": [
      "significant_wave_height",
      "peak_period",
      "directional_spectrum",
      "breaking_wave_count",
      "energy_flux"
    ],
    "output_frequency": 10
  }
}
```

### 6. PRESET CONFIGURATIONS

#### 6.1 Quality Presets

| Preset Name | Resolution | Physics Detail | Particle Count | Purpose |
|-------------|------------|----------------|----------------|---------|
| Preview | 256×256 | Low | 100,000 | Fast iteration during setup |
| Standard | 512×512 | Medium | 500,000 | Regular simulation runs |
| High | 1024×1024 | High | 2,000,000 | Detailed analysis |
| Ultra | 2048×2048 | Ultra | 10,000,000 | Final visualization |

```json
{
  "quality_preset": "High",
  "preset_overrides": {
    "particle_count": 3000000
  }
}
```

#### 6.2 Location Presets

```json
{
  "location_presets": [
    {
      "name": "Malibu First Point",
      "location": {
        "latitude": 34.035,
        "longitude": -118.678
      },
      "domain_size": [5000.0, 5000.0, 100.0],
      "bathymetry": "bathymetry/malibu.exr",
      "default_wave": {
        "height": 1.5,
        "period": 12.0,
        "direction": 270.0
      },
      "default_tide": 0.8,
      "custom_parameter_adjustments": {
        "breaking_threshold": 0.75
      }
    },
    {
      "name": "Pipeline",
      "location": {
        "latitude": 21.665,
        "longitude": -158.053
      },
      "domain_size": [3000.0, 3000.0, 150.0],
      "bathymetry": "bathymetry/pipeline.exr",
      "default_wave": {
        "height": 3.0,
        "period": 14.0,
        "direction": 315.0
      },
      "default_tide": 0.2,
      "custom_parameter_adjustments": {
        "breaking_threshold": 0.8
      }
    }
  ]
}
```

### 7. OPTIMIZATION GUIDELINES

#### 7.1 Resolution vs. Performance

| Domain Size | Grid Resolution | Memory Usage | Performance Level | Suitable Hardware |
|-------------|----------------|--------------|-------------------|-------------------|
| 1km × 1km   | 10m            | ~500 MB      | Very Fast         | Gaming Laptop     |
| 1km × 1km   | 5m             | ~2 GB        | Fast              | Gaming Desktop    |
| 1km × 1km   | 2m             | ~10 GB       | Moderate          | Workstation       |
| 1km × 1km   | 1m             | ~40 GB       | Slow              | GPU Server        |
| 5km × 5km   | 10m            | ~10 GB       | Moderate          | Workstation       |
| 5km × 5km   | 5m             | ~40 GB       | Slow              | GPU Server        |
| 5km × 5km   | 2m             | ~250 GB      | Very Slow         | Multi-GPU Server  |

#### 7.2 Adaptive Resolution Guidelines

When using adaptive resolution:

1. **Base Grid Resolution:** Set this 5-10× coarser than the finest required detail level
2. **Maximum Refinement Levels:** Each level increases memory usage by ~25%
3. **Refinement Criteria:** Focus adaptation on areas of interest with specific criteria
4. **Refinement Thresholds:** Calibrate thresholds to refine 10-20% of the domain maximum
5. **Adaptive Region Size:** Keep individual adaptive regions smaller than 25% of the domain

#### 7.3 Computation Optimization Strategies

```
┌───────────────────────────────────────┐
│         OPTIMIZATION APPROACH         │
└───────────────────────────────────────┘
            │
┌───────────┴──────────┐
▼                      ▼
┌───────────────────┐  ┌───────────────────┐
│ Memory-Optimized  │  │ Speed-Optimized   │
└───────────────────┘  └───────────────────┘
        │                       │
        ▼                       ▼
┌───────────────────┐  ┌───────────────────┐
│ • Smaller domain  │  │ • Coarser base    │
│ • Stream results  │  │   resolution      │
│ • Sparse grids    │  │ • Simplified      │
│ • Checkpoint-     │  │   physics         │
│   restart         │  │ • GPU acceleration│
└───────────────────┘  └───────────────────┘
```

### 8. VALIDATION AND CALIBRATION

#### 8.1 Validation Scenarios

```json
{
  "validation_scenarios": [
    {
      "name": "StandingWave",
      "type": "analytical",
      "description": "Standing wave comparison with analytical solution",
      "parameters": {
        "wave_height": 1.0,
        "wave_period": 10.0,
        "water_depth": 50.0
      },
      "metrics": [
        {
          "name": "height_error",
          "type": "max_relative_error",
          "target_value": 0.01,  // 1% error tolerance
          "field": "height_field"
        },
        {
          "name": "phase_error",
          "type": "phase_difference",
          "target_value": 0.05,  // 5% phase error tolerance
          "field": "height_field"
        }
      ]
    },
    {
      "name": "SolitaryWave",
      "type": "analytical",
      "description": "Solitary wave propagation test",
      "parameters": {
        "wave_height": 0.5,
        "water_depth": 10.0
      },
      "metrics": [
        {
          "name": "shape_conservation",
          "type": "shape_similarity",
          "target_value": 0.95,  // 95% shape preservation
          "field": "height_field"
        },
        {
          "name": "speed_error",
          "type": "propagation_speed_error",
          "target_value": 0.02,  // 2% speed error tolerance
          "field": "height_field"
        }
      ]
    },
    {
      "name": "WaveBreaking",
      "type": "reference",
      "description": "Breaking wave comparison with tank measurement",
      "reference_data": "validation/tank_breaking_wave.csv",
      "metrics": [
        {
          "name": "breaking_point",
          "type": "position_error",
          "target_value": 2.0,   // 2 meter accuracy
          "field": "breaking_events"
        },
        {
          "name": "breaking_height",
          "type": "relative_error",
          "target_value": 0.1,   // 10% height accuracy
          "field": "breaking_metrics"
        }
      ]
    }
  ]
}
```

#### 8.2 Calibration Workflow

```
┌───────────────────────────────────────┐
│ 1. Run baseline simulation with       │
│    default parameters                 │
└───────────────────────┬───────────────┘
                        │
                        ▼
┌───────────────────────────────────────┐
│ 2. Compare with validation metrics    │
└───────────────────────┬───────────────┘
                        │
                        ▼
┌───────────────────────────────────────┐
│ 3. Identify parameters with highest   │
│    sensitivity                        │
└───────────────────────┬───────────────┘
                        │
                        ▼
┌───────────────────────────────────────┐
│ 4. Adjust parameters systematically   │
│    in isolation                       │
└───────────────────────┬───────────────┘
                        │
                        ▼
┌───────────────────────────────────────┐
│ 5. Evaluate improvement against       │
│    metrics                            │
└───────────────────────┬───────────────┘
                        │
                        ▼
┌───────────────────────────────────────┐
│ 6. Create calibrated parameter set    │
└───────────────────────────────────────┘
```

---

## APPENDICES

### Appendix A: Complete Configuration Example

```json
{
  "simulation_name": "Malibu_Swell_March2023",
  "description": "Malibu First Point, 6ft west swell, light offshore wind",
  "version": "Genesis 2.5",
  
  "domain": {
    "size_x": 5000.0,
    "size_y": 5000.0,
    "size_z": 100.0,
    "origin": [0.0, 0.0, -50.0],
    "grid_resolution": 5.0,
    "adaptive_grid": true,
    "maximum_refinement": 3,
    "refinement_threshold": 0.5
  },
  
  "bathymetry": {
    "type": "imported",
    "file_path": "bathymetry/malibu_reef.exr",
    "scaling_factor": 1.0,
    "depth_offset": -50.0,
    "smoothing_passes": 2
  },
  
  "adaptive_regions": [
    {
      "name": "Breaking Zone",
      "type": "box",
      "center": [2500.0, 2500.0, 0.0],
      "size": [1000.0, 1000.0, 50.0],
      "resolution": 1.0,
      "priority": 1
    },
    {
      "name": "Surf Zone",
      "type": "box",
      "center": [2500.0, 2300.0, 0.0],
      "size": [1000.0, 200.0, 20.0],
      "resolution": 0.5,
      "priority": 2
    }
  ],
  
  "wave_spectra": [
    {
      "type": "JONSWAP",
      "significant_wave_height": 2.0,
      "peak_period": 10.0,
      "peak_enhancement": 3.3,
      "direction": 270.0,
      "directional_spreading": 0.3,
      "seed": 12345,
      "component_count": 256
    }
  ],
  
  "current_field": {
    "type": "uniform",
    "uniform_current": {
      "speed": 0.5,
      "direction": 180.0
    }
  },
  
  "wind": {
    "type": "uniform",
    "speed": 3.0,
    "direction": 70.0,
    "gustiness": 0.1,
    "height_reference": 10.0
  },
  
  "time_control": {
    "start_time": 0.0,
    "end_time": 300.0,
    "time_step_method": "adaptive",
    "min_time_step": 0.01,
    "max_time_step": 0.05,
    "cfl_target": 0.5
  },
  
  "physics": {
    "solver_type": "Hybrid",
    "breaking_threshold": 0.78,
    "foam_generation": 1.0,
    "foam_dissipation": 0.5,
    "water_density": 1025.0,
    "gravity": 9.81,
    "boundary_condition": "absorbing",
    "sponge_layer_width": 50.0
  },
  
  "performance": {
    "compute_device": "GPU",
    "precision": "float32",
    "gpu_device_id": 0,
    "thread_count": 16,
    "memory_limit": 16384,
    "tile_size": 16,
    "optimization_level": "balanced"
  },
  
  "output_configuration": {
    "base_output_dir": "results/malibu_march2023/",
    "output_frequency": 4,
    "compression": "lz4",
    
    "outputs": [
      {
        "type": "height_field",
        "enabled": true,
        "format": "exr",
        "resolution": [1024, 1024],
        "precision": "float32",
        "file_pattern": "height_field_{frame:04d}.exr"
      },
      {
        "type": "velocity_field",
        "enabled": true,
        "format": "npz",
        "dimensions": [1024, 1024, 32],
        "precision": "float32",
        "file_pattern": "velocity_{frame:04d}.npz"
      },
      {
        "type": "foam_particles",
        "enabled": true,
        "format": "csv",
        "max_particles": 1000000,
        "attributes": ["position", "age", "velocity"],
        "file_pattern": "foam_{frame:04d}.csv"
      }
    ]
  },
  
  "analysis": {
    "enabled": true,
    "realtime": false,
    "regions": [
      {
        "name": "PrimaryBreak",
        "type": "box",
        "center": [2500.0, 2300.0, 0.0],
        "size": [200.0, 200.0, 10.0],
        "metrics": ["wave_height", "breaking_intensity", "peel_angle"]
      }
    ],
    "global_metrics": [
      "significant_wave_height",
      "peak_period",
      "breaking_wave_count"
    ],
    "output_frequency": 10
  }
}
```

### Appendix B: Parameter Sensitivity Matrix

| Parameter               | Height Field | Velocity Field | Breaking | Foam | Shoaling | Overall Impact |
|-------------------------|--------------|----------------|----------|------|----------|----------------|
| `significant_wave_height` | ★★★★★        | ★★★★☆          | ★★★★☆    | ★★★☆☆  | ★★★☆☆    | Very High      |
| `peak_period`           | ★★★★☆        | ★★★★☆          | ★★★★☆    | ★★☆☆☆  | ★★★★★    | Very High      |
| `directional_spreading` | ★★★☆☆        | ★★★☆☆          | ★★★★☆    | ★★★☆☆  | ★★★☆☆    | High           |
| `grid_resolution`       | ★★★★☆        | ★★★★★          | ★★★★★    | ★★★★☆  | ★★★★☆    | Very High      |
| `breaking_threshold`    | ★☆☆☆☆        | ★★★☆☆          | ★★★★★    | ★★★★★  | ★☆☆☆☆    | Medium         |
| `foam_generation`       | ☆☆☆☆☆        | ☆☆☆☆☆          | ★☆☆☆☆    | ★★★★★  | ☆☆☆☆☆    | Low            |
| `boundary_condition`    | ★★★☆☆        | ★★★☆☆          | ★☆☆☆☆    | ★☆☆☆☆  | ★★☆☆☆    | Medium         |
| `wind_speed`            | ★★☆☆☆        | ★★★☆☆          | ★★★☆☆    | ★★★☆☆  | ★☆☆☆☆    | Medium         |
| `current_speed`         | ★★★☆☆        | ★★★★☆          | ★★★★☆    | ★★☆☆☆  | ★★★☆☆    | High           |

### Appendix C: Performance Benchmarks

| Configuration       | Grid Resolution | Domain Size   | GPU          | Memory Usage | FPS    | Simulation Speed |
|---------------------|----------------|---------------|--------------|--------------|--------|------------------|
| Preview Quality     | 5m base, 2.5m adaptive | 2km × 2km | RTX 3060     | 4.5 GB       | 30-60  | 2-5× realtime    |
| Standard Quality    | 2m base, 1m adaptive   | 2km × 2km | RTX 3080     | 12 GB        | 15-30  | 1-2× realtime    |
| High Quality        | 1m base, 0.5m adaptive | 2km × 2km | RTX 3090     | 24 GB        | 5-15   | 0.3-1× realtime  |
| Ultra Quality       | 0.5m base, 0.25m adapt | 2km × 2km | RTX 4090     | 45 GB        | 2-5    | 0.1-0.3× realtime|
| Production Rendering| 0.25m base, 0.1m adapt | 2km × 2km | 4× A100      | 160 GB       | 0.5-2  | 0.01-0.1× realtime|

*Document Version: 1.0*
*Last Updated: [Current Date]* 