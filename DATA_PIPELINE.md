# OCEANIC DATA PIPELINE SPECIFICATION
## From Data Collection to Visualization

### 1. DATA ACQUISITION LAYER

#### 1.1 Data Sources and Protocols

| Data Type               | Source                  | Update Frequency | Access Protocol          | Format        |
|-------------------------|-------------------------|------------------|--------------------------|---------------|
| Wave Buoy Measurements  | NOAA NDBC               | 30-60 min        | HTTP REST API            | JSON, CSV     |
| Satellite Altimetry     | CMEMS, NASA PODAAC      | 1-24 hours       | OpenDAP, FTP             | NetCDF        |
| Weather Forecasts       | GFS, ECMWF              | 6-12 hours       | GRIB API                 | GRIB2         |
| Coastal Radar           | CODAR SeaSonde          | 15-60 min        | HTTP, FTP                | NetCDF, HDF5  |
| Bathymetry              | GEBCO, NOAA NCEI        | Static           | Web services             | GeoTIFF       |
| Tidal Models            | OTPS, FES2014           | Predictable      | Specialized API          | ASCII, NetCDF |
| River Discharge         | USGS, NOAA              | 15-60 min        | REST API                 | JSON, CSV     |

#### 1.2 Data Collection System

```
┌────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Scheduler      │────>│ Data Collectors │────>│ Raw Data Store  │
└────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        │                       │                       │
        ▼                       ▼                       ▼
┌────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Task Queue     │     │ Error Handling  │     │ Validation      │
└────────────────┘     └─────────────────┘     └─────────────────┘
```

#### 1.3 Data Collection Implementation

```python
# Python pseudo-code for NDBC buoy data collection
def collect_ndbc_buoy_data(buoy_ids, output_dir):
    """Collect real-time data from NDBC buoys."""
    
    results = {}
    
    for buoy_id in buoy_ids:
        try:
            # Construct URL for the buoy's real-time data
            url = f"https://www.ndbc.noaa.gov/data/realtime2/{buoy_id}.spec"
            
            # Download the data
            response = requests.get(url, timeout=30)
            response.raise_for_status()
            
            # Parse the spectral data file
            data = parse_ndbc_spectral_data(response.text)
            
            # Store the result
            results[buoy_id] = data
            
            # Save to file
            output_file = os.path.join(output_dir, f"{buoy_id}_{data['timestamp'].strftime('%Y%m%d_%H%M')}.json")
            with open(output_file, 'w') as f:
                json.dump(data, f)
                
            logging.info(f"Successfully collected data for buoy {buoy_id}")
            
        except Exception as e:
            logging.error(f"Failed to collect data for buoy {buoy_id}: {str(e)}")
            continue
            
    return results

def parse_ndbc_spectral_data(text_data):
    """Parse NDBC spectral data format."""
    lines = text_data.strip().split('\n')
    
    # Extract header information
    header = lines[0].split()
    
    # Parse the data rows
    data_rows = []
    for line in lines[1:]:
        if line.strip() and not line.startswith('#'):
            values = line.split()
            if len(values) >= 15:  # Typical NDBC spectral files have at least 15 columns
                data_rows.append([float(v) if is_numeric(v) else v for v in values])
    
    # Create structured data
    structured_data = {
        'timestamp': datetime.strptime(f"{data_rows[0][0]} {data_rows[0][1]}", "%Y%m%d %H%M"),
        'latitude': float(header[1]),
        'longitude': float(header[2]),
        'significant_wave_height': data_rows[0][5],
        'peak_period': data_rows[0][7],
        'average_period': data_rows[0][9],
        'mean_wave_direction': data_rows[0][11],
        'spectral_data': [
            {
                'frequency': row[2],
                'energy': row[3],
                'r1': row[4],
                'r2': row[5],
                'direction': row[6]
            }
            for row in data_rows
        ]
    }
    
    return structured_data
```

#### 1.4 Data Collection Architecture

- **AWS Implementation**:
  - Lambda functions for periodic API calls
  - S3 for raw data storage
  - EventBridge for scheduling
  - SQS for task queue management
  - CloudWatch for monitoring and error alerting

- **Error Handling Strategy**:
  - Exponential backoff for temporary failures
  - Dead letter queues for manual inspection
  - Redundant data sources where possible
  - Alerting for consistent failures

### 2. DATA PROCESSING LAYER

#### 2.1 Pre-Processing and Validation

```
┌─────────────────┐     ┌────────────────────┐     ┌───────────────────┐
│ Raw Data        │────>│ Quality Control    │────>│ Outlier Detection │
└─────────────────┘     └────────────────────┘     └───────────────────┘
                                  │                           │
                                  ▼                           ▼
                         ┌────────────────────┐     ┌───────────────────┐
                         │ Unit Conversion    │────>│ Data Interpolation│
                         └────────────────────┘     └───────────────────┘
                                                              │
                                                              ▼
                                                    ┌───────────────────┐
                                                    │ Processed Data    │
                                                    └───────────────────┘
```

#### 2.2 Data Fusion and Integration

```python
# Python pseudo-code for data fusion
def fuse_ocean_data(buoy_data, satellite_data, weather_data, bathymetry):
    """Fuse multiple data sources into a unified ocean state."""
    
    # Create a grid for the analysis domain
    grid = create_spatial_grid(
        lat_min=domain_bounds['lat_min'],
        lat_max=domain_bounds['lat_max'],
        lon_min=domain_bounds['lon_min'],
        lon_max=domain_bounds['lon_max'],
        resolution=1000  # 1km grid
    )
    
    # Initialize ocean state grid
    ocean_state = initialize_ocean_state(grid)
    
    # Integrate bathymetry
    ocean_state = integrate_bathymetry(ocean_state, bathymetry)
    
    # Apply weather forcing (wind, pressure)
    ocean_state = apply_weather_forcing(ocean_state, weather_data)
    
    # Integrate buoy observations with spatial interpolation
    ocean_state = integrate_buoy_data(ocean_state, buoy_data)
    
    # Apply satellite observations (with appropriate weighting)
    ocean_state = integrate_satellite_data(ocean_state, satellite_data)
    
    # Ensure physical consistency
    ocean_state = enforce_physical_constraints(ocean_state)
    
    return ocean_state

def integrate_buoy_data(ocean_state, buoy_data):
    """Integrate buoy observations into the ocean state grid."""
    
    # For each buoy
    for buoy_id, data in buoy_data.items():
        # Get buoy location
        lat, lon = data['latitude'], data['longitude']
        
        # Find grid cells within influence radius
        influenced_cells = find_cells_within_radius(ocean_state.grid, lat, lon, influence_radius=50000)  # 50km
        
        # Calculate distance-based weights
        weights = calculate_distance_weights(ocean_state.grid, influenced_cells, lat, lon)
        
        # Apply buoy data with distance-based weighting
        for cell_idx, weight in zip(influenced_cells, weights):
            # Update significant wave height
            ocean_state.significant_wave_height[cell_idx] = (
                (1 - weight) * ocean_state.significant_wave_height[cell_idx] +
                weight * data['significant_wave_height']
            )
            
            # Update peak period
            ocean_state.peak_period[cell_idx] = (
                (1 - weight) * ocean_state.peak_period[cell_idx] +
                weight * data['peak_period']
            )
            
            # Update wave direction
            ocean_state.mean_direction[cell_idx] = circular_weighted_average(
                ocean_state.mean_direction[cell_idx],
                data['mean_wave_direction'],
                1 - weight,
                weight
            )
    
    return ocean_state
```

#### 2.3 Integration with External Models

- **WAVEWATCH III Integration**:
  - Boundary condition import
  - Output comparison for validation
  - Two-way nesting capabilities

- **NOAA ESTOFS (Extratropical Surge and Tide Operational Forecast System)**:
  - Water level and current integration
  - Tidal input for shallow water simulation

- **USGS CoSMoS (Coastal Storm Modeling System)**:
  - Extreme event validation
  - Runup and inundation comparison

### 3. SIMULATION INTERFACE LAYER

#### 3.1 Genesis Input Preparation

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────────────┐
│ Processed Data  │────>│ Domain Extraction  │────>│ Parameter Conversion │
└─────────────────┘     └────────────────────┘     └─────────────────────┘
                                  │                           │
                                  ▼                           ▼
                         ┌────────────────────┐     ┌─────────────────────┐
                         │ Boundary Condition │────>│ Genesis Input Format│
                         └────────────────────┘     └─────────────────────┘
```

#### 3.2 Simulation Parameter Mapping

| Environmental Data        | Genesis Parameter            | Conversion/Mapping                         |
|---------------------------|------------------------------|-------------------------------------------|
| Significant Wave Height   | `wave_amplitude_scale`       | `Hs / 4.0`                                |
| Peak Period               | `wave_period`                | Direct mapping                            |
| Mean Direction            | `wave_direction`             | Convert from meteorological to math angle |
| Wind Speed                | `wind_speed`                 | Convert to m/s if needed                  |
| Wind Direction            | `wind_direction`             | Convert from meteorological to math angle |
| Current Speed             | `current_velocity_field`     | Convert to vector field                   |
| Bathymetry                | `terrain_height_field`       | Invert and scale appropriately            |
| Water Temperature         | `water_density`              | Use equation of state for seawater        |
| Spectral Data             | `wave_spectrum_components`   | Convert to amplitude/phase components     |

#### 3.3 Batch Processing System

```python
# Python pseudo-code for batch generation
def generate_simulation_batch(ocean_states, target_locations, output_dir):
    """Generate a batch of Genesis simulations from ocean states."""
    
    # Create a batch configuration
    batch_config = {
        "batch_id": generate_unique_id(),
        "timestamp": datetime.now().isoformat(),
        "simulation_count": len(target_locations),
        "ocean_state_time": ocean_states["timestamp"],
        "simulations": []
    }
    
    # For each target location (surf spot)
    for location_id, location in target_locations.items():
        # Extract domain centered on target location
        domain_config = extract_simulation_domain(
            ocean_states,
            center_lat=location["latitude"],
            center_lon=location["longitude"],
            domain_size=location["domain_size"]
        )
        
        # Create simulation configuration
        sim_config = {
            "simulation_id": f"{batch_config['batch_id']}_{location_id}",
            "location_name": location["name"],
            "latitude": location["latitude"],
            "longitude": location["longitude"],
            "domain_config": domain_config,
            "resolution": location["resolution"],
            "duration": location["simulation_duration"],
            "output_frequency": location["output_frequency"]
        }
        
        # Add to batch
        batch_config["simulations"].append(sim_config)
        
        # Write individual simulation configuration
        sim_output_path = os.path.join(output_dir, f"{sim_config['simulation_id']}.json")
        with open(sim_output_path, 'w') as f:
            json.dump(sim_config, f, indent=2)
    
    # Write batch configuration
    batch_output_path = os.path.join(output_dir, f"{batch_config['batch_id']}_batch.json")
    with open(batch_output_path, 'w') as f:
        json.dump(batch_config, f, indent=2)
    
    return batch_config
```

### 4. SIMULATION EXECUTION LAYER

#### 4.1 Job Distribution System

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────────────┐
│ Batch Generator │────>│ Job Queue          │────>│ Worker Nodes        │
└─────────────────┘     └────────────────────┘     └─────────────────────┘
                                  │                           │
                                  ▼                           ▼
                         ┌────────────────────┐     ┌─────────────────────┐
                         │ Resource Manager   │<────│ Simulation Progress │
                         └────────────────────┘     └─────────────────────┘
                                  │                           │
                                  ▼                           ▼
                         ┌────────────────────┐     ┌─────────────────────┐
                         │ Spot Instance      │────>│ Result Collection   │
                         │ Manager            │     │                     │
                         └────────────────────┘     └─────────────────────┘
```

#### 4.2 AWS Batch Configuration

```json
{
  "jobDefinitionName": "GenesisWaveSimulation",
  "type": "container",
  "containerProperties": {
    "image": "genesis-wave-simulator:latest",
    "vcpus": 16,
    "memory": 32768,
    "command": [
      "python",
      "run_simulation.py",
      "Ref::configFile"
    ],
    "volumes": [
      {
        "host": {
          "sourcePath": "/data"
        },
        "name": "data"
      }
    ],
    "environment": [
      {
        "name": "AWS_REGION",
        "value": "us-west-2"
      }
    ],
    "mountPoints": [
      {
        "containerPath": "/data",
        "sourceVolume": "data"
      }
    ],
    "resourceRequirements": [
      {
        "type": "GPU",
        "value": "1"
      }
    ]
  },
  "retryStrategy": {
    "attempts": 2
  },
  "platformCapabilities": [
    "EC2"
  ]
}
```

#### 4.3 Spot Instance Management

```python
# Python pseudo-code for spot instance management
def manage_spot_instances(simulation_queue, max_instances=20):
    """Manage spot instances based on simulation queue."""
    
    # Get current running instances
    running_instances = get_running_spot_instances(tag="WaveSimulation")
    
    # Get pending jobs
    pending_jobs = get_queue_length(simulation_queue)
    
    # Calculate target instance count
    target_instances = min(max_instances, math.ceil(pending_jobs / JOBS_PER_INSTANCE))
    
    # Current instance count
    current_instances = len(running_instances)
    
    if target_instances > current_instances:
        # Need to launch more instances
        instances_to_launch = target_instances - current_instances
        launch_spot_instances(
            count=instances_to_launch,
            instance_type="g5.2xlarge",  # Adjust based on simulation needs
            max_price="1.50",  # Maximum bid price per hour
            ami_id=GENESIS_AMI_ID,
            user_data=generate_worker_userdata(simulation_queue),
            tags={"Name": "WaveSimulation", "AutoTerminate": "true"}
        )
        logger.info(f"Launched {instances_to_launch} new spot instances")
    
    elif target_instances < current_instances:
        # Can terminate some instances
        instances_to_terminate = current_instances - target_instances
        
        # Find idle instances or oldest instances
        instances_to_stop = find_instances_to_terminate(
            running_instances, 
            count=instances_to_terminate
        )
        
        terminate_spot_instances(instances_to_stop)
        logger.info(f"Terminated {len(instances_to_stop)} spot instances")
    
    return {
        "running_instances": current_instances,
        "target_instances": target_instances,
        "pending_jobs": pending_jobs
    }
```

### 5. SIMULATION RESULTS LAYER

#### 5.1 Data Collection and Storage

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────────────┐
│ Genesis Output  │────>│ Result Collector   │────>│ Hierarchical Storage│
└─────────────────┘     └────────────────────┘     └─────────────────────┘
                                  │                           │
                                  ▼                           ▼
                         ┌────────────────────┐     ┌─────────────────────┐
                         │ Result Validation  │────>│ Metadata Indexing   │
                         └────────────────────┘     └─────────────────────┘
                                                              │
                                                              ▼
                                                    ┌─────────────────────┐
                                                    │ Data Cataloging     │
                                                    └─────────────────────┘
```

#### 5.2 Storage Hierarchy

| Data Type               | Storage Tier               | Retention Policy                     | Access Pattern        |
|-------------------------|----------------------------|--------------------------------------|----------------------|
| Raw Simulation Data     | S3 Standard -> Glacier     | Full: 1 week, Sampled: 1 year       | Sequential, Batch    |
| Processed Height Fields | S3 Standard                | 2 weeks                              | Random, Frequent     |
| Visualization Assets    | CloudFront + S3            | 1 week                               | Random, Very Frequent|
| Metadata                | DynamoDB                   | Indefinite                           | Random, Frequent     |
| Analysis Results        | S3 Standard                | 1 month                              | Random, Moderate     |

#### 5.3 S3 Storage Strategy

```
s3://wave-simulation-data/
├── raw/
│   ├── YYYY-MM-DD/
│   │   ├── batch_id/
│   │   │   ├── location_id/
│   │   │   │   ├── height_fields/
│   │   │   │   ├── velocity_fields/
│   │   │   │   ├── foam_particles/
│   │   │   │   └── metadata.json
├── processed/
│   ├── YYYY-MM-DD/
│   │   ├── location_id/
│   │   │   ├── height_maps/
│   │   │   ├── normal_maps/
│   │   │   ├── foam_maps/
│   │   │   └── wave_metrics.json
├── visualization/
│   ├── YYYY-MM-DD/
│   │   ├── location_id/
│   │   │   ├── preview.mp4
│   │   │   ├── thumbnail.jpg
│   │   │   ├── time_series.json
│   │   │   └── unreal_assets/
└── analyses/
    ├── YYYY-MM-DD/
    │   ├── regional_overview/
    │   ├── historical_comparison/
    │   └── alert_conditions/
```

### 6. VISUALIZATION PREPARATION LAYER

#### 6.1 Conversion to Unreal Engine Format

```python
# Python pseudo-code for preparing data for Unreal Engine
def prepare_for_unreal_engine(simulation_results, output_dir):
    """Convert Genesis simulation results to Unreal Engine format."""
    
    # Create output directory structure
    os.makedirs(os.path.join(output_dir, "HeightMaps"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "NormalMaps"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "VelocityMaps"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "FoamMaps"), exist_ok=True)
    os.makedirs(os.path.join(output_dir, "Metadata"), exist_ok=True)
    
    # Process height fields
    for i, height_field_path in enumerate(simulation_results["height_fields"]):
        # Load height field (assuming EXR format)
        height_field = load_exr(height_field_path)
        
        # Convert to normalized range suitable for Unreal
        normalized_height = normalize_height_field(height_field)
        
        # Save as PNG (16-bit)
        output_path = os.path.join(output_dir, "HeightMaps", f"height_{i:04d}.png")
        save_16bit_png(normalized_height, output_path)
        
        # Generate normal map
        normal_map = generate_normal_map(height_field)
        
        # Save normal map
        normal_path = os.path.join(output_dir, "NormalMaps", f"normal_{i:04d}.png")
        save_rgb_16bit_png(normal_map, normal_path)
    
    # Process velocity fields
    for i, velocity_field_path in enumerate(simulation_results["velocity_fields"]):
        # Load velocity field
        velocity_field = load_npz(velocity_field_path)
        
        # Convert to 2D flow map (suitable for Unreal materials)
        flow_map = convert_to_flow_map(velocity_field)
        
        # Save as PNG
        output_path = os.path.join(output_dir, "VelocityMaps", f"flow_{i:04d}.png")
        save_rgb_16bit_png(flow_map, output_path)
    
    # Process foam particles
    for i, foam_path in enumerate(simulation_results["foam_particles"]):
        # Load foam particle data
        foam_particles = load_csv(foam_path)
        
        # Convert to density map
        foam_map = particles_to_density_map(foam_particles, width=1024, height=1024)
        
        # Save as PNG
        output_path = os.path.join(output_dir, "FoamMaps", f"foam_{i:04d}.png")
        save_8bit_png(foam_map, output_path)
    
    # Create metadata file
    metadata = {
        "frame_count": len(simulation_results["height_fields"]),
        "physical_scale": simulation_results["metadata"]["physical_scale"],
        "simulation_duration": simulation_results["metadata"]["duration"],
        "time_step": simulation_results["metadata"]["time_step"],
        "wave_statistics": simulation_results["metadata"]["wave_statistics"],
        "georeferencing": {
            "latitude": simulation_results["metadata"]["latitude"],
            "longitude": simulation_results["metadata"]["longitude"],
            "orientation": simulation_results["metadata"]["orientation"],
            "scale": simulation_results["metadata"]["scale"]
        }
    }
    
    # Save metadata
    with open(os.path.join(output_dir, "Metadata", "simulation_metadata.json"), 'w') as f:
        json.dump(metadata, f, indent=2)
    
    return {
        "output_dir": output_dir,
        "frame_count": metadata["frame_count"],
        "metadata_path": os.path.join(output_dir, "Metadata", "simulation_metadata.json")
    }
```

#### 6.2 Batch Conversion System

- **Input**: Genesis simulation result directories
- **Output**: Unreal Engine-compatible asset packages
- **Processing Steps**:
  1. Parallel conversion of multiple simulations
  2. Automatic LOD generation
  3. Metadata extraction and enhancement
  4. Quality validation and verification
  5. Creation of Unreal datatable assets

#### 6.3 Packaging for Delivery

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────────────┐
│ Converted Assets│────>│ Asset Packaging    │────>│ Compression         │
└─────────────────┘     └────────────────────┘     └─────────────────────┘
                                  │                           │
                                  ▼                           ▼
                         ┌────────────────────┐     ┌─────────────────────┐
                         │ Metadata Embedding │────>│ Delivery Package    │
                         └────────────────────┘     └─────────────────────┘
```

### 7. USER DELIVERY LAYER

#### 7.1 Content Delivery Network

```
┌─────────────────┐     ┌────────────────────┐     ┌─────────────────────┐
│ S3 Storage      │────>│ CloudFront CDN     │────>│ Edge Locations      │
└─────────────────┘     └────────────────────┘     └─────────────────────┘
                                                              │
                                                              ▼
                                                    ┌─────────────────────┐
                                                    │ End Users           │
                                                    └─────────────────────┘
```

#### 7.2 Progressive Loading Strategy

- **Initial Data**: Low-resolution overview, metadata
- **Secondary Load**: Medium-resolution time steps, essential metrics
- **On-Demand**: High-resolution detail, specialized analysis

#### 7.3 Response Time Optimization

| Content Type          | Target Response Time | Caching Strategy                     | Optimization Technique         |
|-----------------------|----------------------|--------------------------------------|--------------------------------|
| Thumbnails & Previews | <100ms               | Edge caching, long TTL               | Pre-generate, compress         |
| Time-series Data      | <500ms               | Regional caching, medium TTL         | Incremental loading            |
| Full Visualization    | <2s                  | Origin caching, short TTL            | Progressive enhancement        |
| On-demand Analysis    | <5s                  | Minimal caching, computed            | Background processing          |

---

## APPENDICES

### Appendix A: Data Schema Definitions

#### A.1 Ocean State Schema
```json
{
  "timestamp": "2023-03-15T12:00:00Z",
  "domain": {
    "lat_min": 32.5,
    "lat_max": 33.5,
    "lon_min": -118.5,
    "lon_max": -117.5,
    "resolution": 0.01
  },
  "grid_dimensions": [101, 101],
  "parameters": {
    "significant_wave_height": {
      "units": "m",
      "data": [[1.2, 1.3, ...], [...], ...]
    },
    "peak_period": {
      "units": "s",
      "data": [[8.5, 8.6, ...], [...], ...]
    },
    "mean_direction": {
      "units": "deg",
      "data": [[270, 272, ...], [...], ...]
    },
    "wind_speed": {
      "units": "m/s",
      "data": [[5.2, 5.3, ...], [...], ...]
    },
    "wind_direction": {
      "units": "deg",
      "data": [[240, 242, ...], [...], ...]
    },
    "bathymetry": {
      "units": "m",
      "data": [[-50, -52, ...], [...], ...]
    }
  },
  "source_data": {
    "buoys": ["46222", "46231"],
    "satellites": ["sentinel-6"],
    "models": ["gfs", "wavewatch"]
  }
}
```

#### A.2 Genesis Input Schema
```json
{
  "simulation_id": "sim_20230315_santamonica",
  "domain": {
    "size_x": 10000,
    "size_y": 10000,
    "size_z": 1000,
    "resolution": 10.0,
    "physical_scale": 1.0
  },
  "adaptive_regions": [
    {
      "center_x": 5000,
      "center_y": 5000,
      "center_z": 0,
      "size_x": 2000,
      "size_y": 2000,
      "size_z": 200,
      "resolution": 1.0
    }
  ],
  "initial_conditions": {
    "wave_spectrum": {
      "components": [
        {
          "amplitude": 0.5,
          "period": 10.0,
          "direction": 270.0,
          "phase": 0.0
        },
        {
          "amplitude": 0.3,
          "period": 8.0,
          "direction": 260.0,
          "phase": 0.5
        }
      ]
    },
    "wind": {
      "speed": 5.0,
      "direction": 240.0
    },
    "current": {
      "field_path": "current_field.npz"
    },
    "bathymetry": {
      "field_path": "bathymetry.exr"
    }
  },
  "simulation_parameters": {
    "duration": 300.0,
    "time_step": 0.01,
    "output_frequency": 4
  },
  "exporters": {
    "height_field": {
      "enabled": true,
      "resolution": [1024, 1024],
      "format": "exr"
    },
    "velocity_field": {
      "enabled": true,
      "resolution": [1024, 1024, 32],
      "format": "npz"
    },
    "foam_particles": {
      "enabled": true,
      "format": "csv"
    },
    "metadata": {
      "enabled": true,
      "format": "json"
    }
  }
}
```

### Appendix B: AWS Infrastructure Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Amazon VPC                              │
│                                                                 │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐  │
│  │  Data         │     │  Simulation   │     │  Rendering    │  │
│  │  Collection   │     │  Cluster      │     │  Cluster      │  │
│  │  Subnet       │     │  Subnet       │     │  Subnet       │  │
│  │               │     │               │     │               │  │
│  │  ┌─────────┐  │     │  ┌─────────┐  │     │  ┌─────────┐  │  │
│  │  │ Lambda  │  │     │  │ Batch   │  │     │  │ EC2     │  │  │
│  │  │ Function│  │     │  │ Workers │  │     │  │ Unreal  │  │  │
│  │  └─────────┘  │     │  └─────────┘  │     │  │ Renderer│  │  │
│  │       │       │     │       │       │     │  └─────────┘  │  │
│  │       ▼       │     │       ▼       │     │       │       │  │
│  │  ┌─────────┐  │     │  ┌─────────┐  │     │       ▼       │  │
│  │  │ SQS     │  │     │  │ EFS     │  │     │  ┌─────────┐  │  │
│  │  │ Queue   │──┼─────┼─>│ Storage │──┼─────┼─>│ S3      │  │  │
│  │  └─────────┘  │     │  └─────────┘  │     │  │ Storage │  │  │
│  │               │     │               │     │  └─────────┘  │  │
│  └───────────────┘     └───────────────┘     └───────────────┘  │
│                                                          │      │
└──────────────────────────────────────────────────────────┼──────┘
                                                           │
                                                           ▼
                                                     ┌──────────┐
                                                     │CloudFront│
                                                     │   CDN    │
                                                     └──────────┘
                                                           │
                                                           ▼
                                                     ┌──────────┐
                                                     │  Users   │
                                                     └──────────┘
```

### Appendix C: Performance Benchmarks

| Operation                | Average Duration | Input Size  | Output Size | Notes                         |
|--------------------------|------------------|-------------|-------------|-------------------------------|
| Data Collection          | 3-5 minutes      | N/A         | ~100MB      | Per global update cycle       |
| Data Processing          | 5-10 minutes     | ~100MB      | ~500MB      | Includes fusion and validation|
| Simulation Generation    | 10-30 minutes    | ~10MB       | ~2-5GB      | Per premium surf spot         |
| Visualization Conversion | 3-5 minutes      | ~2-5GB      | ~500MB      | Optimized for Unreal Engine   |
| Rendering                | 5-15 minutes     | ~500MB      | ~50-200MB   | For videos and previews       |
| Full Pipeline            | 30-60 minutes    | N/A         | ~50-200MB   | End-to-end for single spot    |

---

*Document Version: 1.0*
*Last Updated: [Current Date]* 