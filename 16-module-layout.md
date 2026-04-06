# ScaleFast Engine — Module Layout

## Part 16: Engine Architecture and Module Map

### Module Hierarchy

```
engine/
├── core/
│   ├── job_system/          Fiber-based, no main thread, Chase-Lev work stealing
│   ├── memory/              Frame allocators, pool allocators, per-map budgets
│   └── io/                  Async file I/O, Zstd decompression, NVMe-aware
│
├── platform/
│   └── sdl_wrapper/         Window, input, audio device init (cross-platform)
│
├── renderer/
│   ├── vulkan_rhi/          Vulkan abstraction (device, swapchain, commands)
│   ├── pipeline_cache/      Pre-compiled PSO management
│   ├── cluster_system/      Light + decal + fog volume clustering
│   ├── shadow_system/       VSM (directional) + cached atlas (local)
│   ├── texture_stream/      Virtual texturing, page cache, feedback
│   ├── post_process/        Bloom, tonemap, GTAO, motion blur, TAA, CAS, SSR
│   ├── particle/            GPU compute sim, density volume, rendering
│   └── volumetric/          Froxel fog, light scattering, temporal reprojection
│
├── physics/
│   ├── jolt_adapter/        Job system integration, world setup, debug vis
│   └── character/           Controller wrapper, ground detection, IK feedback
│
├── animation/
│   ├── skeleton/            Poses, blend trees, state machines, compression
│   ├── ik/                  Two-bone, FABRIK, cascaded full-body, foot placement
│   └── procedural/          Breathing, recoil, secondary motion, look-at
│
├── world/
│   ├── map_loader/          Parallel decompression, asset upload, sector management
│   ├── visibility/          Precomputed sector visibility, runtime occlusion culling
│   ├── entity_system/       Game objects, components, simulation/presentation split
│   ├── decal_system/        Clustered world decals, character mesh decals, drips, pools
│   └── navigation/          NavMesh, A*, obstacle carving, path cache, AI LOD
│
├── audio/
│   ├── mixer/               2.1 output, active panning, distance attenuation
│   └── spatial/             Occlusion queries against world geometry
│
├── networking/
│   ├── transport/           UDP, custom reliability, channels
│   ├── prediction/          Client-side prediction, input history, reconciliation
│   ├── interpolation/       Remote entity interpolation, extrapolation
│   ├── replication/         Entity state sync, delta compression, impulse events
│   └── lag_compensation/    Server-side hit detection, world state history, rewind
│
└── tools/
    ├── budget_overlay/      Real-time per-system cost visualization
    ├── lod_inspector/       LOD comparison, silhouette validation
    ├── light_inspector/     Per-light shadow cost display
    └── profiler/            Frame timeline, per-job timing, GPU timestamps
```

### Dependency Graph (What Knows About What)

```
core/job_system      → nothing (foundation, no dependencies)
core/memory          → nothing
core/io              → core/job_system

platform/sdl         → nothing

renderer/*           → core/*, platform/*
physics/jolt_adapter → core/job_system
physics/character    → physics/jolt_adapter

animation/*          → core/job_system (parallel evaluation)
animation/ik         → physics/character (foot contact feedback)

world/map_loader     → core/io, renderer/*, physics/jolt_adapter
world/entity_system  → core/job_system
world/navigation     → physics/jolt_adapter (raycasts)
world/decal_system   → renderer/cluster_system

audio/*              → core/job_system, world/entity_system

networking/*         → core/job_system, world/entity_system, physics/*
```

Physics doesn't know about rendering. Rendering doesn't know about physics. The world module connects them. IK reads from animation and physics, writes poses back. The job system underlies everything.

### Frame Timeline (Complete)

```
CPU JOBS (parallel, fiber system):
  ├── Input polling
  ├── Physics step (Jolt, via job adapter)
  ├── Animation update + IK solve (parallel per character)
  ├── Emitter updates (particles)
  ├── AI perception + decision + pathfinding + steering
  ├── Entity logic / gameplay
  ├── Visibility culling (frustum + precomputed sectors)
  └── Command buffer recording (parallel across workers)

GPU TIMELINE:
   1. Depth pre-pass + foliage alpha        0.3-0.5ms
   2. Hi-Z pyramid build                    0.05ms
   3. Particle simulation (compute)          0.1ms
   4. Particle density injection             0.05ms
   5. VSM page analysis + dirty render       0.3-0.5ms
   6. Local shadow atlas updates             0.3-0.8ms
   7. Light + decal cluster build            0.2ms
   8. Volumetric fog pipeline                0.33ms
   9. Forward shading pass                   2.0-3.0ms
  10. Particle rendering                     0.2-0.4ms
  11. SSR trace + resolve                    0.5ms
  12. Contact shadows                        0.1ms
  13. GTAO                                   0.2-0.3ms
  14. TAA / DLSS / FSR / XeSS               0.1ms
  15. Post-process (bloom, tonemap,          0.5-0.8ms
      motion blur, CAS)
  16. Present
                                             ──────────
  Typical total:                             5.5-7.8ms
  Target (144 FPS):                          6.9ms
  Target (120 FPS):                          8.3ms
  Target (60 FPS at 4K):                     16.67ms
```

### Quality Presets

```
Ultra (4090 tier):
  Full-res GTAO, full volumetric (160×90×128)
  Maximum shadow budget, full SSR
  100K+ particle cap, all effects maxed

High (4070 Ti tier):
  Half-res GTAO + bilateral upsample
  Reduced volumetric grid (80×45×64)
  Shadow budget: 4 local updates/frame
  50K particle cap, shorter contact shadows

Medium (4060 Ti tier):
  Half-res GTAO, reduced volumetrics
  Shadow budget: 2 local updates/frame
  25K particles, SSR at low or off
  DLSS/FSR recommended

Low (entry-level):
  Quarter-res GTAO, minimal volumetrics
  Sun shadows only, no local shadow casting
  10K particles, SSR off
  DLSS/FSR required for reasonable framerates
```

Framerate is the immovable constraint. Compromise order: shadow update rate → shadow resolution → GTAO resolution → volumetric resolution → SSR quality → particle count → never framerate.
