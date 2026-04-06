# ScaleFast Engine — Design Document

## Part 1: Overview and Philosophy

**ScaleFast** is a modern 3D game engine written in C++20, designed around a single uncompromising principle: **runtime performance is the immovable constraint. Everything else bends to serve it.**

### Core Philosophy

ScaleFast is not Unreal Engine 5. It does not prioritize developer convenience at the expense of the player's experience. Where UE5 says "let the artist do whatever they want and we'll figure out the performance at runtime," ScaleFast says **"the engine provides powerful, optimized systems — the designer is responsible for using them wisely, and the engine gives them the data to make informed decisions."**

This is the philosophy of id Software's idTech engines, of Guerrilla Games' Decima, of the best purpose-built engines in the industry. Build-time cost is acceptable. Runtime cost must be justified.

### Design Principles

1. **Framerate over everything.** The target is 144+ FPS at 1440p on high-end hardware, 60+ FPS at 4K. Neural upscaling (DLSS/FSR/XeSS) exists to democratize access for weaker hardware, never as a crutch for poor optimization.

2. **Designer-driven optimization.** The engine surfaces real-time per-system budget visualization. Designers see exactly what each light, each emitter, each shadow caster costs. The engine optimizes what it can prove is safe. It never silently degrades systems the designer has marked as essential.

3. **No main thread.** Every piece of work is a job in a fiber-based task graph. No thread is special. Any worker can pick up any job. The frame itself is a dependency graph, not a serial chain.

4. **Forward+ over deferred.** No G-buffer. No bandwidth tax. MSAA compatibility (if ever needed), natural transparency support, and critical bandwidth savings on the mid-range GPUs where memory bus widths are shrinking while consumers demand 4K.

5. **Discrete maps, not open worlds.** Bounded, fully-known datasets enable aggressive precomputation — baked shadow maps, pre-built SDFs, optimal texture packing, pre-compiled shader pipelines. Zero runtime shader compilation. Zero streaming hitches. Load times measured in seconds.

6. **Simulation/presentation split from day one.** Fixed-rate simulation (60 Hz) decoupled from variable-rate rendering. This is correct architecture for single-player smoothness AND multiplayer networking. The same code path serves both.

### How ScaleFast Differs from UE5

| Aspect | Unreal Engine 5 | ScaleFast |
|--------|-----------------|-----------|
| **Rendering** | Deferred with heavy G-buffer | Clustered forward+ — no G-buffer bandwidth tax |
| **Geometry** | Nanite (runtime virtualized geometry) | Traditional LOD with artist-authored meshes. Build-time cost, not runtime cost |
| **GI** | Lumen (software ray tracing, expensive) | No runtime GI in base spec. Baked probes + volumetric fog. Path tracing is a distant luxury |
| **Shadows** | Virtual Shadow Maps | Virtual Shadow Maps (sun) + cached shadow atlas (local lights). Designer-budgeted per-light shadow casting |
| **Optimization** | Runtime systems adapt automatically | Designer-driven budgets with real-time cost visualization |
| **Streaming** | Runtime streaming of everything | Geometry fully resident at load. Only texture mips stream at runtime. Zero hitches |
| **Threading** | Task graph with main thread orchestration | Fiber-based job system, no main thread. id Software-style architecture |
| **Physics** | Chaos (built-in) | JoltPhysics — deterministic, open source, designed for custom job system integration |
| **Target framerate** | 30-60 FPS console-first | 144+ FPS PC-first. Console targets are secondary |
| **Developer model** | Convenience-first (artists/designers have minimal constraints) | Performance-first (designers have powerful tools with visible cost feedback) |
| **Shader compilation** | Runtime compilation causes stutter | All PSOs pre-compiled at build time. Zero runtime compilation |
| **Frame generation** | Assumed for high-res targets | Never factored into performance calculations. Pure bonus |

### Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | C++20 | Industry standard for engine development. Rust isn't mature enough for this domain yet |
| Graphics API | Vulkan | Cross-platform (Windows + Linux), explicit threading model, compute shader support |
| Physics | JoltPhysics | MIT license, zero dependencies, deterministic, custom job system interface by design |
| Platform | SDL | Window, input, audio device init. Cross-platform plumbing handled cleanly |
| Math | GLM | Header-only, SIMD-accelerated, Vulkan clip-space compatible |
| Build | CMake | Industry standard for cross-platform C++ with complex dependency management |
| Compression | Zstd | Fast decompression for map loading, saturates NVMe bandwidth |

### Performance Targets

```
4090 + 7800X3D:
  1440p native:  120-144 FPS at Ultra
  4K native:     80-100 FPS at Ultra
  4K native:     60 FPS locked at Ultra (with margin)

4070 Ti + 7800X3D:
  1440p native:  90-120 FPS at Ultra
  1440p native:  144 FPS locked at High
  4K native:     45-55 FPS at High
  4K + DLSS Q:   70-90 FPS at Ultra

5070 Ti + 7800X3D:
  1440p native:  120-144 FPS at Ultra
  4K native:     55-70 FPS at High
  4K + DLSS Q:   90-110 FPS at Ultra

4060 Ti + 7800X3D:
  1440p native:  60-80 FPS at High
  1440p + DLSS:  90-110 FPS at Ultra
  4K + DLSS Q:   50-60 FPS at High
```

DLSS/FSR/XeSS is never required. Native rendering at the target resolution and framerate is the design point. Neural upscaling lets weaker hardware participate or strong hardware push further.

Frame generation (DLSS 3, FSR 3) is never factored into any performance calculation. It is a pure bonus for users who want it.

Path tracing is explicitly out of scope for the base renderer. If ever implemented, it is a luxury feature that assumes neural rendering assistance. It is not part of any budget calculation.

# ScaleFast Engine — Job System

## Part 2: Fiber-Based Job System (No Main Thread)

### Inspiration

DOOM Eternal (2020) and DOOM: The Dark Ages (2025) run on a fiber-based job system with no dedicated main thread. Every piece of work — input, physics, animation, rendering command recording, submission — is a job. No thread is special. Any worker picks up any job. On Linux, this architecture scales to infinity, reaching 1000+ FPS when other bottlenecks are removed.

ScaleFast adopts this architecture wholesale.

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                Job System (no main thread)            │
│                                                      │
│  N worker threads (= hardware thread count)          │
│  Each worker:                                        │
│    - Pulls from local deque (LIFO, cache-warm)       │
│    - Steals from others (FIFO, load balance)         │
│    - Runs fibers, not callbacks                      │
│                                                      │
│  Fibers:                                             │
│    - Lightweight (~64KB stack, pooled)                │
│    - Can yield/wait without blocking the worker      │
│    - When a job waits on a dependency, the fiber     │
│      suspends and the worker picks up new work       │
│                                                      │
│  No thread ever idles. No thread is "special."       │
└──────────────────────────────────────────────────────┘
```

### Why Fibers, Not Callbacks

In a callback-based task system, if Job A needs the result of Job B:
1. Block the worker thread (terrible — one fewer worker)
2. Split Job A into A1 and A2, making A2 depend on B (explosion of small tasks, complex graph)

With fibers, Job A says "wait on this counter." The fiber suspends, the worker immediately picks up another fiber/job, and when B completes (decrementing the counter), A's fiber becomes runnable again. **Any** worker can resume it. Zero idle time.

### The Wait-Free Counter Primitive

Every synchronization point is an atomic counter:

```cpp
struct JobCounter {
    std::atomic<int32_t> value;
    // When value hits 0, all fibers waiting on this become runnable
};
```

- `KickJobs(jobs[], count, &counter)` — enqueues jobs, sets counter to count
- `WaitForCounter(counter, targetValue)` — suspends current fiber, resumes when counter reaches target
- Job completion atomically decrements the counter

No mutexes. No condition variables. No kernel transitions in the hot path.

### Work-Stealing Deques

Each worker has a Chase-Lev lock-free deque:
- Tasks spawned by a fiber push locally (LIFO — cache-warm, the spawning worker likely processes them)
- Other workers steal from the opposite end (FIFO — load balancing)
- Excellent cache locality for the spawning thread while distributing load across cores

### Fiber Context Switching

No Boost. No `ucontext` (unnecessary signal mask save/restore overhead). Minimal inline assembly for x86-64:

```asm
; ~20 instructions total
; save: push rbx, rbp, r12-r15, save rsp to context
; restore: load rsp from new context, pop r15-r12, rbp, rbx, ret
```

Fiber stacks are pre-allocated from a pool (~256-512 fibers, 64KB each). Allocated with `mmap` on Linux, `VirtualAlloc` on Windows, with guard pages.

**Important:** Thread-local storage (TLS) does not work with fibers (a fiber can resume on a different thread). Fiber-local storage is used instead — a pointer stored in the fiber context.

### Frame as a Job Graph

```
Frame N kicks off when Frame N-2 presents (triple buffered):

  ┌──────────┐
  │  Input   │
  └────┬─────┘
       │
  ┌────▼─────┐     ┌───────────┐
  │ Physics  │────▶│ Animation │
  │ Kick     │     │ Update    │
  └────┬─────┘     └─────┬─────┘
       │                 │
  ┌────▼─────────────────▼─────┐
  │     Visibility / Culling    │
  └────────────┬───────────────┘
               │
    ┌──────────▼──────────┐
    │  Light Cluster Build │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │ Cmd Buffer Recording │
    │ + Shadow Maps        │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  Submit + Present    │
    └──────────────────────┘
```

No "submit thread" or "render thread." The job that finishes recording last also does the submit. It's the natural tail of the dependency chain.

### Simulation / Presentation Split

Built in from day one, essential for both smooth rendering and multiplayer:

```cpp
frame_simulate(dt);      // fixed tick rate (60 Hz)
  // physics, AI, physics-coupled IK, game logic
  // In multiplayer: runs on server
  // In single player: runs locally

frame_present(alpha);    // render framerate (variable, 144+ Hz)
  // alpha = interpolation between last two sim states
  // animation blend trees, cosmetic IK, cloth, rendering
  // Always runs on client
```

### Linux Scheduling Advantage

- No DPC/ISR overhead eating frame time
- CFS/EEVDF pins N consistently runnable threads across cores
- `SCHED_FIFO` / `SCHED_DEADLINE` for real-time scheduling of workers
- `madvise(MADV_HUGEPAGE)` for fiber stack pool eliminates TLB misses
- Vulkan on Mesa (RADV, ANV) has less driver overhead than Windows in many cases

### JoltPhysics Job System Integration

Jolt's `JobSystem` is an abstract interface explicitly designed for custom implementations. Their reference thread pool is documented as "an example — provide your own."

```cpp
class EngineJobSystem : public JPH::JobSystemWithBarrier {
    FiberScheduler* scheduler;

    void QueueJob(JPH::Job* job) override {
        job->AddRef();
        scheduler->submit([job] {
            job->Execute();
            job->Release();
        });
    }

    void WaitForJobs(Barrier* barrier) override {
        while (barrier->HasPendingJobs()) {
            if (!try_execute_barrier_job(barrier))
                fiber_yield();
        }
    }
};
```

No TLS dependency in Jolt's hot path. A fiber can start a Jolt job on thread 3 and resume on thread 7. No breakage. This is why PhysX was rejected — its TLS dependency conflicts fundamentally with fiber-based scheduling.

# ScaleFast Engine — Rendering Pipeline

## Part 3: Clustered Forward+ Renderer

### Why Forward+, Not Deferred

A deferred G-buffer at 4K writes ~20 bytes/pixel (albedo + normal + roughness/metallic + velocity + depth) = 166 MB written, then 166 MB read back during the lighting pass. At 60 FPS that's ~20 GB/s of bandwidth consumed by the G-buffer alone. At 144 FPS: ~48 GB/s.

On a 4070 Ti with 504 GB/s total bandwidth, that's 4-10% eaten before doing anything useful. Mid-range GPU memory buses are shrinking (192-bit on the 4070 Ti vs 256-bit on the 3070) while consumers demand 4K. Forward+ skips all of that.

**UE5 comparison:** Unreal's deferred renderer pays this bandwidth tax on every frame. Nanite's visibility buffer helps reduce G-buffer fills for opaque geometry but the fundamental bandwidth cost remains for the lighting pass. ScaleFast avoids the entire problem.

### Clustered Forward+ Architecture

Clustered forward+ splits the camera frustum into a 3D grid of clusters (frustum-aligned voxels). Each cluster is assigned a list of affecting lights and decals. During shading, each fragment looks up its cluster and iterates only the assigned lights.

**Why clustered over tiled:** Tiled forward+ splits the screen into 2D tiles and assigns lights per tile. Lights close to the camera but high in screen space still affect tiles at the bottom of the frustum. Clustered splits into 3D, providing much tighter culling with varying depth complexity.

### The Rendering Pipeline

```
1. Depth pre-pass                          ~0.3-0.5ms
   - All opaque geometry, alpha-tested foliage
   - Outputs depth buffer for: early-Z, cluster bounds,
     Hi-Z pyramid, GTAO, SSR, contact shadows

2. Hi-Z pyramid build (compute)            ~0.05ms
   - Mip chain of min-depth for SSR tracing
     and GPU occlusion culling

3. Particle simulation (compute)            ~0.1ms
   - GPU compute, SoA buffers, 100K+ particles

4. Particle density injection (compute)     ~0.05ms
   - For particle shadow casting and volumetric interaction

5. VSM page analysis + dirty page render    ~0.3-0.5ms
   - Virtual Shadow Maps for directional light (sun)
   - Only re-render invalidated pages

6. Local shadow atlas updates               ~0.3-0.8ms
   - Cached atlas for local lights
   - Only dirty lights re-render (static lights are free)

7. Light + decal cluster build (compute)    ~0.2ms
   - Assign lights, decals, and fog volumes to clusters

8. Volumetric fog pipeline (compute)        ~0.33ms
   - Material injection → light scattering →
     temporal reprojection → ray march integration

9. Forward shading pass                     ~2.0-3.0ms
   - Full PBR materials with Cook-Torrance GGX BRDF
   - Per-fragment: cluster lookup, iterate lights,
     sample shadows, evaluate decals, apply volumetric fog
   - Foliage translucency (thin surface SSS)

10. Particle rendering                      ~0.2-0.4ms
    - Instanced billboards (additive sparks, alpha-blended smoke)
    - Lit via cluster light list

11. SSR trace + resolve (compute)           ~0.5ms
    - Hi-Z ray marching, temporal accumulation
    - Half-res at Medium, full-res at Ultra

12. Contact shadows (compute)               ~0.1ms
    - Short screen-space ray march for fine contact darkening

13. GTAO (compute)                          ~0.2-0.3ms
    - Ground Truth Ambient Occlusion
    - Half-res with bilateral upsample at High preset

14. TAA resolve                             ~0.1ms
    - Variance clipping, velocity-weighted blend
    - OR: DLSS / FSR / XeSS slot (replaces TAA)

15. Post-process                            ~0.5-0.8ms
    - Bloom (dual Kawase downsample/upsample)
    - Tonemapping (ACES)
    - Motion blur (tile-based, configurable samples)
    - CAS sharpening (if TAA active, not with DLSS)

16. Present
```

### PBR Material System

Cook-Torrance microfacet BRDF:
- **NDF:** GGX/Trowbridge-Reitz
- **Geometry:** Smith-GGX height-correlated
- **Fresnel:** Schlick approximation
- **IBL:** Split-sum approximation (pre-filtered environment map + BRDF LUT)

~25 ALU ops per light evaluation. Industry standard, well-documented, fast.

### Foliage Rendering

Foliage uses **thin surface translucency** — a cheap SSS approximation for leaves:

```
Front-lit: standard PBR diffuse + specular
Back-lit:  evaluate lighting with FLIPPED normal,
           modulated by a thickness map (baked per-leaf)
           and tinted by translucency color
```

The thickness map encodes leaf anatomy — veins and stems are opaque (dark), blade is thin and transmits (bright). When backlit, you see vein structure silhouetted. Cost: ~5 extra ALU ops per light per foliage fragment.

Foliage overdraw is managed by alpha testing in the depth pre-pass. The forward pass gets early-Z rejection — only the frontmost leaf at each pixel runs the full lighting shader. Overdraw drops from 5-15x to ~1-2x.

Wind animation is pure vertex shader — world-space noise texture sampled at vertex position, weighted by artist-painted sway weights. Gust propagation is a wave front moving through the noise texture. Same approach as Assassin's Creed Shadows. Zero CPU cost.

### HDR Rendering

Internal render target: R11G11B10F (saves bandwidth vs RGBA16F, alpha not needed for most passes). Full HDR pipeline throughout, tonemapped at the end.

# ScaleFast Engine — Shadow System

## Part 4: Shadows — VSM + Cached Local Shadow Atlas

### Philosophy

Shadows are where frame budgets go to die. Most engines either limit shadow-casting light count severely (id's approach — maximum quality on few lights) or accept uniformly dark shadow regions because local lights don't cast shadows (most UE5 titles).

ScaleFast takes a third path: **every light CAN cast shadows, managed by an automatic budget system with designer overrides.** Static shadow maps are cached and cost nearly nothing at runtime. Only dynamic invalidation costs per-frame budget.

### Directional Light: Virtual Shadow Maps

The sun uses Virtual Shadow Maps — a single massive virtual texture (16K x 16K logical resolution) per light, backed by a page table. Only pages actually visible to the camera get allocated and rendered.

**Page caching is the key optimization.** Static geometry renders into shadow pages once. Pages are cached until invalidated by dynamic objects. In steady state, the per-frame VSM cost is dominated only by the few pages that dynamic objects (players, AI, vehicles) touch.

**Foliage in shadow maps** uses dithered alpha testing. At high shadow resolution (near camera), individual leaf shapes are visible. At low resolution (distant cascades), the dither pattern blurs into soft canopy shadow. Volumetric fog samples these shadow maps, producing light shafts through foliage canopy automatically.

### Local Lights: Cached Shadow Atlas

A single large shadow atlas (e.g., 8K x 8K) shared by all local lights. Each shadow-casting light gets a region. The system that makes "every light casts shadows" affordable:

**Shadow lifecycle:**
1. Light becomes visible → queued for initial shadow render
2. Initial render → full shadow map, stored in atlas, marked CLEAN
3. Steady state → test shadow volume against dynamic objects each frame
4. No overlap with dynamics? → Skip. Cached shadow is pixel-perfect
5. Dynamic object enters shadow volume? → Mark DIRTY, queue re-render
6. Re-render → only dirty cubemap faces, not all 6

**Point light optimization:** Dual paraboloid maps (2 renders) instead of cubemaps (6 renders) for medium-distance point lights. 3x reduction in shadow geometry submission. For wall/ceiling mounted lights, faces pointing into walls are culled entirely — another ~50% reduction.

**Partial face updates:** When the player walks past a lantern, only the 1-2 cubemap faces facing the player need re-rendering. At 256x256 per face for a light 10m away, this costs almost nothing.

### Shadow Budget System

```
Per frame, the engine manages a shadow rendering budget:

  Shadow budget:    ████████░░ 1.2ms / 1.5ms allocated
  
  Directional (VSM):  ~40% of budget (biggest visual impact)
  Local lights:        ~60% of budget (importance-scored)
```

**Importance scoring per light:**
```
score = (intensity * screen_coverage * receiver_count) / distance²
```

Lights are sorted by score. Top N get shadow updates this frame. Everything else shows cached shadows. The designer can override with `ForcedOn` (always updates) or `ForcedOff` (never casts shadows).

### Per-Light Shadow Intensity

Each light has a configurable `shadow_intensity` (0.0 to 1.0):

```glsl
shadow = mix(1.0, shadowMapResult, light.shadowIntensity);
lighting *= shadow;
```

This simulates ambient fill in shadowed regions without requiring global illumination:
- Harsh desert sun: 0.95 (deep, dark shadows)
- Overcast day: 0.4 (soft, subtle shadows)
- Interior lamp: 0.7 (moderate, ambient fill from bounced light)
- Moonlight: 0.3 (barely perceptible shadows)

**UE5 comparison:** UE5 relies on Lumen for shadow fill. ScaleFast achieves visually similar results with a per-light scalar — zero runtime GI cost, full designer control.

### How Shadows Actually Work (Mental Model)

Shadows don't add darkness. Each light contributes independently:

```
finalColor = ambient
for each light:
    shadow = sampleShadowMap(light, worldPos)
    shadow = mix(1.0, shadow, light.shadowIntensity)
    finalColor += lightColor * BRDF * attenuation * shadow
```

A pixel in the sun's shadow but lit by a lantern = warm, lantern-dominated. A pixel in both shadows = dark, both blocked. The shadow map for each light only affects THAT light's contribution. This is why having many shadow-casting local lights creates rich, dimensional lighting — each light adds pools of illumination with their own shadow patterns.

### Particle Shadow Casting

Particle density is injected into a 3D volume texture. Shadow maps sample this volume during rendering, darkening where particle density blocks light. This produces:
- Smoke clouds casting shadows on terrain
- Light shafts cut by passing smoke
- Volumetric self-shadowing for dense effects

The density volume is shared with the volumetric fog system — one data structure, three visual effects (particle shadows, volumetric scattering interaction, self-shadowing).

Designer flag `particle_shadows` on lights controls which lights respond to particle density. Typically the sun + 1-2 dominant interior lights.

### Contact Shadow Fallback

Screen-space contact shadows (short ray march in depth buffer) provide fine-scale contact darkening always-on. Cheap (~0.1ms) and handles the gap between shadow map resolution and sub-pixel contact detail.

### Designer Controls

```cpp
struct LightShadowSettings {
    ShadowMode mode;             // None, Static, Dynamic
    ShadowPriority priority;     // ForcedOn, High, Medium, Low, ForcedOff
    uint16_t max_resolution;     // per-light cap (128-2048)
    uint8_t update_rate;         // 1 = every frame, 2 = half rate
    float shadow_distance;       // max distance from camera
    float shadow_fade_start;     // distance to begin fading
    float shadow_intensity;      // 0.0-1.0, how dark the shadow is
    bool cache_static;           // enable static shadow caching
    bool particle_shadows;       // respond to particle density
};
```

`ForcedOn` guarantees shadow casting. `ForcedOff` means never. Everything between is managed by importance scoring. The engine shows:

```
⚠ 3 lights in sector_07 exceed shadow budget
```

The designer sees the cost. The engine doesn't hide it.

# ScaleFast Engine — Anti-Aliasing

## Part 5: TAA with Neural Upscaler Slot

### The State of AA

Every AA technique is a compromise. The industry converged on temporal approaches not because they're great, but because PBR shading made MSAA insufficient (MSAA only solves geometric aliasing, not shader aliasing from GGX specular, normal maps, or screen-space effects) and temporal upscaling (DLSS/FSR) needed a temporal framework anyway.

MSAA is excluded from ScaleFast. It only affects geometry edges, costs significant fill rate in a forward renderer with many lights per cluster, and adds code complexity for a partial solution. SMAA/FXAA are similarly excluded — post-process morphological AA has been superseded by TAA.

### TAA Implementation

**Jitter:** Halton(2,3) sequence, 16 sample positions. At 144 FPS the full sequence completes in 111ms — fast enough that convergence is never perceived as incomplete.

**History rejection (variance clipping — Salvi 2016):**
```
Per pixel:
1. Reproject to previous frame using motion vectors
2. Sample history buffer at reprojected location
3. Compute color bounding box from 3x3 neighborhood
4. Clamp history sample to bounding box
5. If clamped significantly → increase current frame weight
6. If motion vector magnitude is high → increase weight further
7. Blend: result = lerp(clamped_history, current, weight)
   weight: 0.05-0.15 (static) to 0.3+ (fast motion)
```

Variance clipping eliminates ghosting in almost all cases by rejecting history pixels that are statistically improbable. The cost is slightly more noise in rejection regions, but at 144 FPS reconvergence is near-instant.

**Sharpening:** CAS (Contrast Adaptive Sharpening, AMD open source) after TAA to recover detail softened by temporal accumulation. This is what DOOM Eternal does. Without sharpening, TAA looks like vaseline. With well-tuned CAS, the image is crisp.

### DLSS / FSR / XeSS Integration

All three replace TAA entirely — they ARE temporal AA with neural reconstruction. When active, the TAA pass is disabled.

**Shared infrastructure:** Motion vectors, jitter pattern, depth buffer, and exposure value are generated by the normal pipeline regardless of which AA/upscaler is active. TAA needs them. DLSS needs them. FSR needs them. One infrastructure, multiple consumers.

```
Architecture:

              ┌─────────────┐
              │  Render at   │
              │ internal res │
              └──────┬──────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
   ┌────▼────┐  ┌────▼────┐  ┌───▼────┐
   │  TAA +  │  │  DLSS   │  │  FSR   │  (mutually exclusive)
   │  CAS    │  │         │  │  XeSS  │
   └────┬────┘  └────┬────┘  └───┬────┘
        └────────────┼───────────┘
                     │
              ┌──────▼──────┐
              │ Post-process │
              └─────────────┘
```

**DLSS integration:** The SDK is a DLL + headers. ~200-300 lines wrapping SDK calls behind an interface. The engine provides all inputs DLSS needs (motion vectors, depth, exposure, jitter) because they're already generated for TAA.

**FSR 2.x:** Open source, works on all GPUs. Serves as the vendor-neutral baseline upscaler.

### Philosophy

DLSS is an equalizer, not a crutch:
- **Top-end GPU (4090):** Doesn't need DLSS. Runs native. DLSS available to push 8K or 240Hz
- **Upper-mid (4070 Ti):** 4K60 native at High. DLSS pushes to Ultra or higher framerate
- **Mid (4060 Ti):** DLSS Quality at 1440p→4K makes this GPU punch above its weight

# ScaleFast Engine — Physics

## Part 6: JoltPhysics Integration

### Why Jolt, Not PhysX

| | PhysX 5.x | JoltPhysics |
|---|---|---|
| **License** | BSD 3-Clause | MIT |
| **Dependencies** | Massive SDK, optional CUDA | Zero external dependencies |
| **Threading** | Opaque internal, uses TLS | Abstract JobSystem interface designed for custom implementations |
| **GPU accel** | CUDA only (NVIDIA) | CPU only (by design) |
| **Deterministic** | No | Yes, cross-platform |
| **Build** | Complex CMake + platform presets | Single `add_subdirectory()` |
| **Fiber compatible** | No — TLS breaks fiber migration | Yes — no TLS in hot path |

The decisive factor is threading. PhysX uses thread-local storage internally, which fundamentally conflicts with our fiber-based job system (fibers migrate between threads). Integrating PhysX requires pinning tasks to threads, defeating the purpose of the job system.

Jolt's `JobSystem` is explicitly documented as "an example implementation — provide your own." The integration is ~50 lines of code wrapping our fiber scheduler.

GPU-accelerated physics (PhysX CUDA) was never a consideration. Keep the GPU doing GPU things (rendering). Keep the CPU doing CPU things (physics). The performance rollercoaster of GPU physics competing with rendering for GPU resources is not worth the headache.

### What Jolt Provides

**Fully built-in (zero work needed):**
- Rigid body simulation (static, dynamic, kinematic)
- Continuous collision detection (CCD)
- Full constraint suite (hinge, slider, cone, swing-twist, 6DOF, gear, rack-and-pinion, pulley, path)
- Ragdolls with kinematic and motor-driven poses, auto collision filtering, stabilization
- Character controllers (rigid body and virtual variants with contact callbacks, slope limits, shape switching)
- Wheeled vehicles (full drivetrain, tire slip model, differentials)
- Tracked vehicles (tanks)
- Motorcycles
- Soft bodies (XPBD — edges, bends, volumes, pressure)
- Cloth via skinned constraints with backstop spheres
- Hair/tail/rope via Cosserat rod constraints (RodStretchShear, RodBendTwist)
- Water buoyancy calculations
- Double precision mode for large worlds
- Debug renderer interface

**Must be built by us:**
- Animation system (blend trees, state machines, compression)
- Inverse kinematics (all solvers)
- Procedural animation layers

### Cross-Platform Determinism

Jolt produces identical simulation results across Windows, Linux, x86, ARM. This enables:
- Lockstep networking (if needed)
- Impulse-based rigid body synchronization in multiplayer
- Reproducible bug reports
- Replay systems

This is critical for our multiplayer architecture where ragdolls and physics props are synchronized via impulse events rather than full state snapshots.

### Physics Simulation Budget

On a 7800X3D with 96MB L3 cache:
- Full world simulation: ~1.0ms per tick at 60 Hz
- 50 active rigid bodies: negligible additional cost
- 30 AI character controllers: ~0.2ms
- 15 ragdoll bodies (post-death): negligible once at rest (Jolt sleeps them)
- Job system integration means physics parallelizes across all cores

The 3D V-Cache means Jolt's broad phase data structures fit entirely in L3. Cache misses are nearly impossible for gameplay-scale data.

# ScaleFast Engine — Animation and Inverse Kinematics

## Part 7: Homegrown Animation System + Physics-Coupled IK

### Animation Pipeline (Per Character, Per Frame)

```
1. ANIMATION SYSTEM
   ├── Sample active animations
   ├── Blend tree evaluation (layers, states, transitions)
   ├── Additive pose application
   └── Output: LOCAL SPACE SKELETON POSE

2. IK LAYER (post-animation, pre-physics)
   ├── Foot IK: raycast down from hips, two-bone solve per leg
   ├── Look-at IK: spine/head FABRIK toward gaze target
   ├── Hand IK: two-bone solve for object interaction
   ├── Procedural layers: recoil, breathing, weapon sway
   └── Output: MODIFIED POSE → convert to WORLD SPACE

3. PHYSICS COUPLING
   ├── Feed world-space pose to Jolt ragdoll (DriveToPoseUsingMotors)
   ├── Feed to Jolt soft bodies (cloth/cape skinning updates)
   ├── Jolt simulates, resolves collisions/constraints
   └── Output: PHYSICS-CORRECTED WORLD SPACE POSE

4. FINAL POSE
   ├── Blend between IK pose and physics pose (ragdoll blend weight)
   ├── Generate skinning matrices for GPU
   └── Update character controller root if IK shifted it
```

This pipeline order is essential. Animation produces "intent," IK modifies it for environment interaction, physics corrects for physical plausibility. Each stage has clean inputs/outputs. Changing IK doesn't affect animation. Changing physics doesn't affect IK. The interface between stages is `SkeletonPose` — an array of joint transforms.

### IK Solvers

**Analytical Two-Bone IK (legs, arms):**
Closed-form trigonometric solution. ~50 FLOPs per chain. Handles foot placement, hand reaching, and all limb IK. For quadrupeds, run four chains. For spiders, six or eight. The solver doesn't care about leg count.

**FABRIK (multi-bone chains — spines, tails, tentacles):**
Forward And Backward Reaching IK. Iterative, converges in 3-5 iterations. Works for chains of any length. ~100 lines of core solver code.

**Cascaded Full-Body IK:**
Solve feet first (two-bone), then spine (FABRIK for root accommodation), then arms (two-bone). Each pass uses previous results. This is what Assassin's Creed uses for ledge grabbing, what Uncharted uses for climbing.

### Physics-Coupled IK

**Two modes:**

| Mode | IK drives physics | Can disable at distance |
|------|-------------------|------------------------|
| **Cosmetic** | No — visual only, controller independent | Yes (screen-space threshold) |
| **Physics-coupled** | Yes — IK feeds back into character controller | Only when character is inactive |

Physics-coupled IK feeds foot contact points into ground normal estimation, root height adjustment, and slope traversal rejection. Disabling it doesn't just LOOK wrong — the character BEHAVES wrong (falls through slopes, clips into terrain).

**The engine enforces:** if `ik_drives_physics = true`, the IK chain cannot be disabled while the character is active in gameplay. Period.

### Size-Aware LOD (Not Distance-Based)

The correct LOD metric is **screen-space pixel height**, not distance:

```
2m humanoid at 50m  = ~58 pixels at 1440p
40m mech at 50m     = ~1152 pixels
40m mech at 1000m   = ~58 pixels (NOW equivalent to humanoid at 50m)
```

Every LOD threshold is configurable per character type:

```cpp
struct CharacterLODConfig {
    float ik_disable_pixels;         // below this: no cosmetic IK
    float cloth_disable_pixels;      // below this: no soft body
    float mesh_lod1_pixels;          // LOD transition thresholds
    float mesh_lod2_pixels;
    float mesh_lod3_pixels;

    struct IKChainOverride {
        StringID chain_name;
        bool never_disable;          // true = always solve
        float override_pixels;
    };
    std::vector<IKChainOverride> ik_overrides;

    bool ik_drives_physics;          // if true, IK cannot disable
                                     // while physics is active
};
```

A giant mech's legs are set to `never_disable = true`. The IK cost for 4 legs is ~0.02ms — saving nothing by disabling it. From Software's Elden Ring bug (IK disabling at distance regardless of mesh size) is structurally impossible in ScaleFast.

**UE5 comparison:** UE5 uses distance-based LOD with optional screen-size override. ScaleFast uses screen-size as the primary metric with per-chain physics coupling enforcement. The designer has explicit, per-chain control that the engine cannot silently override.

### Animation Rate: Always Full Framerate

Animation blend trees evaluate at render framerate for every visible character. No half-rate, no quarter-rate. Ever.

At 144 FPS on a 360Hz monitor, animation rate reduction is visible at distances most engines consider "safe." The human eye detects motion discontinuities against contrasting backgrounds, even subconsciously.

What IS safe to LOD at distance (invisible at any refresh rate):
- IK solving (cosmetic only, not physics-coupled)
- Cloth / soft body simulation
- Procedural layers (breathing, weapon sway)
- Animation event processing

These are CPU savings that don't affect visual motion smoothness.

### LOD Transitions: Hard Swap, Not Dithering

No dithered transitions. id Software's approach: hard LOD swaps with carefully authored meshes that preserve silhouette, per-asset transition distances, hysteresis to prevent thrashing, and optional motion masking (swap during camera movement when the eye is distracted).

Dithered transitions (screen-door transparency) draw attention to the transition rather than hiding it. At 1440p on a 27" monitor, the dither pattern is visible. TAA can partially mask it, but at 144 FPS TAA converges too fast.

The engine provides LOD authoring tools:
- Split-screen LOD comparison at transition distance
- Silhouette delta highlighting between LODs
- Per-asset transition distance overrides
- Build-time validation flagging visible silhouette changes

### IK Blend Speed (Physical Realism, Not Concession)

IK blend ramp speed is designer-configurable per chain, simulating physical weight:

```cpp
struct IKChainConfig {
    float blend_ramp_speed;   // 0→1 ramp time in seconds
    float max_correction;     // max IK deviation from animation (meters)
    float raycast_ahead;      // ground probe distance (meters)
    bool physics_coupled;     // feeds back into character controller
};

// Giant mech leg: slow, heavy, deliberate
{ .blend_ramp_speed = 0.25f, .max_correction = 3.0f }

// Human foot: fast, responsive
{ .blend_ramp_speed = 0.08f, .max_correction = 0.4f }

// Spider leg: very fast, precise
{ .blend_ramp_speed = 0.03f, .max_correction = 0.2f }
```

This is animation-driven smoothing during walk cycle phases (swing → plant → stance), not a LOD transition hack.

# ScaleFast Engine — Decal System

## Part 8: Clustered PBR Decals

### Philosophy

Decals have regressed in the modern era. Most engines treat them as simple alpha-blended quads. FEAR (2005) was doing more with decals than most 2025 games. ScaleFast brings decals back as a first-class rendering feature — full PBR, reactive to lighting, adhesive to animated characters, with drip and pool fluid dynamics.

### Clustered Decal Evaluation

Decals are evaluated inline during the forward shading pass using the same cluster structure as lights. No separate decal buffer, no extra render pass:

```
During cluster build (same compute pass as light culling):
  - Test each decal's OBB against each cluster
  - Build per-cluster decal index list

During forward shading:
  - For each fragment, look up cluster
  - Iterate cluster's decal list
  - For each decal: project fragment into decal space
  - If within [0,1] UV range: sample decal textures
  - Blend decal properties with surface material BEFORE lighting
  - Light the combined result
```

This means decals participate fully in PBR shading. A blood splatter on metal has different roughness than blood on concrete. The normal map perturbation catches specular highlights. The decal reacts to every dynamic light.

### Full PBR Decal Assets

```
Per decal:
  albedo.png           — color + alpha (alpha IS the shape)
  normal.png           — tangent-space perturbation (RNM blended)
  arm.png              — ambient occlusion / roughness / metallic
  emissive.png         — optional glow (muzzle burns, magic)

Properties:
  blend_mode           — alpha_blend, additive, multiply
  normal_blend         — reoriented normal mapping
  roughness_bias       — wet blood = low roughness (reflective)
  metallic_override    — blood = non-metallic, bullet holes expose metal
  lifetime             — fade-out duration
  sort_priority        — overlapping decal ordering
```

Fresh blood with low roughness naturally picks up specular highlights from nearby lights. As it dries (roughness increases), reflections fade. This is free from PBR shading — just animate the roughness parameter.

### Character Mesh Decals

Screen-space projection doesn't work for animated characters (decal would slide off). Instead:

**At impact time:**
1. Raycast determines hit triangle
2. Find all triangles within decal radius (adjacency walk)
3. Compute decal UV coordinates in mesh's tangent space
4. Store: vertex indices + decal UVs + texture reference

**At render time:**
5. Vertices are already skinned (deformed by animation)
6. Render affected triangles with decal material using stored UVs
7. UVs are FIXED (computed once). Positions are SKINNED (deform with mesh)
8. Decal deforms with the character perfectly, no stretching

Works on cloth/soft bodies too — same vertex buffer that Jolt writes to.

### Blood Drip System

Not flipbook animation. Surface-tracing particles:

1. Impact creates splatter decal
2. Spawns drip particle that slides down surface via raycasting
3. As it moves, extends a mesh decal trail behind it (tapered quad strip)
4. Trail vertices placed on surface via depth/normal sampling
5. Drip front: small wet particle. Behind: trail that gradually dries (roughness increases over time)
6. 10 active drips = 10 raycasts + 10 small meshes per frame. Negligible.

### Blood Pool System

Expanding SDF-driven decal:
1. Impact on horizontal surface → spawn pool decal
2. Pool expands using projected SDF from depth buffer (won't flow uphill)
3. Properties animate: size grows logarithmically, roughness increases (wet → dry), normal shows surface tension at expanding edge

### Performance

```
100 world decals at ~5% screen coverage each:
  Clustered evaluation overhead: ~0.05ms in forward shader
  Zero extra render passes, zero extra memory

Character decals (2-3 per character):
  Tiny extra draw calls (~50-100 triangles each). Negligible.

Drips/pools:
  Few raycasts + small meshes. Negligible.
```

**UE5 comparison:** UE5 uses deferred decals that modify the G-buffer. ScaleFast evaluates decals inline during forward shading — no intermediate buffer, correct interaction with all lights, lower bandwidth.

# ScaleFast Engine — Particle System

## Part 9: GPU-Driven Particles with Physics and Shadow Casting

### Architecture: GPU-First, CPU-Orchestrated

Particles live on the GPU. The CPU spawns them, sets initial conditions, and manages emitter lifetimes. The GPU simulates, sorts, culls, and renders.

```
CPU (jobs in fiber system):
  - Emitter logic: spawn rates, burst events, transforms
  - Budget management: total particle cap, per-emitter caps

GPU (compute shaders):
  - Particle simulation: position, velocity, forces, drag
  - Collision detection + response against scene
  - Sorting for alpha blending (GPU radix sort)
  - Frustum culling + indirect draw argument generation
  - Rendering (instanced quads or mesh particles)
```

### Particle Buffer (Structure of Arrays)

```
Buffer 0: float3 position[]         (12 bytes)
Buffer 1: float3 velocity[]         (12 bytes)
Buffer 2: float  lifetime[]          (4 bytes)
Buffer 3: float  age[]               (4 bytes)
Buffer 4: float  size[]              (4 bytes)
Buffer 5: float3 rotation[]         (12 bytes)
Buffer 6: float3 angularVelocity[]  (12 bytes)
Buffer 7: uint   emitterID[]         (4 bytes)
Buffer 8: uint   flags[]             (4 bytes)
                                     ~68 bytes/particle

1 million particles = ~68 MB GPU memory (trivial)
```

SoA for cache-coherent compute access — simulation shader accesses position and velocity for ALL particles sequentially.

### Three-Tier Collision

**Tier 1: Depth Buffer Collision (most particles, nearly free)**
Project particle to screen space, sample depth buffer. If behind depth → collision. Reconstruct surface normal from depth neighbors, reflect velocity. One texture sample per particle per frame.

Limitation: only works for camera-visible surfaces. Particles behind the camera have no depth data. For sparks/blood flying off surfaces you're looking at, this covers 99% of cases.

**Tier 2: SDF Volumes (persistent effects, medium cost)**
Pre-baked signed distance fields for important geometry. 128³ SDF per room = 2MB. Sample SDF at particle position — distance gives collision detection, gradient gives response direction. One 3D texture sample.

**Tier 3: Jolt Rigid Bodies (few critical particles)**
Shell casings, grenade fragments, large debris. These are small Jolt rigid bodies, not "particles." Dozens, not thousands. Full collision, friction, restitution. Instanced mesh rendering.

### Sparks — The Reference Effect

Sparks are the gold standard particle:

```
Spawn:  velocity reflected off surface + random spread cone
        high angular velocity, 0.5-2.0s lifetime
        bright orange-white (blackbody 2500-4000K)

Simulate: gravity + air drag + depth buffer collision
          restitution 0.3-0.5, friction 0.6-0.8
          color shifts along blackbody curve as spark cools
          (white → orange → red → dark red → dead)

Render: velocity-stretched billboard (motion streak)
        ADDITIVE blending (order-independent, no sorting needed)
        Pure emissive — sparks don't receive scene lighting
        Near-zero per-fragment cost

Environment lighting: emissive particles registered as
        unshadowed micro-lights in the cluster system.
        50 sparks near a surface = warm orange glow.
        No shadow maps, just intensity/distance² falloff.
```

### Particle Shadow Casting

Density volume injection into shadow maps:

1. **Inject:** Compute shader accumulates particle opacity into a 3D froxel grid (160×90×64). Each particle does one atomic add.
2. **Shadow:** During shadow map rendering, ray march through density volume along light direction. Accumulate opacity. Darken shadow texels.
3. **Result:** Smoke clouds cast shadows on terrain. Light shafts get cut by smoke.

The density volume is shared with the volumetric fog system. One data structure serves particle shadows, volumetric scattering, and self-shadowing.

Per-light `particle_shadows` flag controls which lights sample particle density. Sun always does. Interior lights: designer chooses 1-2 dominant lights.

### Particle Lighting

Particles rendered as quads in the forward pass naturally participate in clustered light lookup. A spark flying through shadow goes dark. Smoke in front of a lantern glows orange. Same shadow maps, same cluster system as all geometry.

### Performance

```
Simulation (compute):     ~0.1-0.3ms (100K particles)
Shadow density injection: ~0.05ms
Shadow sampling:          ~0.1-0.2ms (per participating light)
Rendering:                ~0.2-0.4ms (overdraw dependent)
Total:                    ~0.5-1.0ms for full particle pipeline
```

### What's Excluded

- Particle-particle interaction (cut — O(N²), impractical at scale)
- Particle collision with character meshes (cut — rarely looks good)
- Nanite-style anything (runtime cost too high)

# ScaleFast Engine — Volumetrics and Screen-Space Reflections

## Part 10: Froxel Volumetric Fog + Hi-Z SSR

### Volumetric Fog

Froxel-based (frustum-aligned voxel) volumetric fog. Industry standard since Frostbite's implementation.

**Pipeline:**

1. **Material injection (compute):** Write density + scattering into 3D froxel grid (160×90×128). Sources: global fog, local fog volumes (boxes/spheres/height fog), particle density volume, animated 3D noise for fog movement.

2. **Light scattering (compute):** For each froxel, evaluate affecting lights (from cluster system). Sample shadow maps for volumetric shadows. Apply phase function (Henyey-Greenstein). Output: in-scattered luminance + extinction.

3. **Temporal reprojection:** Blend with previous frame (jittered ray offset, Halton sequence). Accumulate over 8-16 frames for smooth, noise-free result. Critical for quality — without it, 128 depth slices look banded.

4. **Ray march integration (compute):** Front-to-back Beer-Lambert accumulation. Output: integrated in-scatter + transmittance per froxel.

5. **Application (forward pass):** One 3D texture sample per fragment. `finalColor = fragmentColor * transmittance + inScatter`

### Volumetric Light Shafts

Light shafts emerge naturally from the volumetric system. The shadow map encodes what blocks the light. Froxels in shadow get no scattering. The boundary between lit and shadowed froxels IS the visible shaft.

**Foliage interaction:** Shadow maps contain dithered foliage alpha. The volumetric system samples these, producing light shafts streaming between leaves. At distance, dither blurs into soft canopy shadow — physically correct. Wind-swayed foliage re-renders shadow pages when leaves move significantly, causing shafts to dance.

### Designer Fog Volumes

```
Box Volume:    density, scattering color, edge falloff
               Use: smoky room, foggy alley

Sphere Volume: radius, density, falloff
               Use: localized mist, breath in cold air

Height Fog:    exponential density below threshold height
               Use: valley fog, ground mist, swamp

Animated:      3D noise texture scrolling through volume
               Use: swirling smoke, dynamic dust
```

Fog volumes are assigned to clusters alongside lights — only relevant volumes evaluated per froxel.

### Particle-Volumetric Integration

The particle density volume feeds directly into volumetric material injection. A smoke grenade's particles inject density → shadow system darkens terrain → volumetric system blocks light shafts. One data structure, three visual effects, zero extra work.

### Underwater (Future)

Same froxel system with different parameters: higher density, wavelength-dependent absorption (red absorbed first → blue-green tint), strong forward scattering. Surface waves modulate light entering water via animated caustic projection. Jolt provides buoyancy. Water drag via increased body damping.

### Screen-Space Reflections

**Hi-Z ray marching (compute):**
1. Build depth mip pyramid (min-depth per 2×2 texels)
2. Per reflective pixel: reflect view direction around normal
3. March through Hi-Z pyramid — start coarse, refine on intersection
4. Typical: 8 coarse + 8 refinement steps (vs 64-128 linear steps)
5. Temporal accumulation + jittered ray start for stability

**Roughness is the control.** PBR roughness already encodes where reflections should be sharp (low roughness), blurry (medium), or absent (high). No separate "reflectivity" slider. A puddle has low roughness center, higher at drying edges. SSR responds naturally.

**Roughness threshold** is the biggest budget lever:
```
Threshold 0.3: only mirrors/wet surfaces    → ~0.3ms
Threshold 0.5: glossy surfaces too          → ~0.5-0.8ms
Threshold 0.7: nearly everything            → ~0.8-1.3ms
```

**Fallback hierarchy:** SSR miss → reflection probe (cubemap, designer-placed) → environment map. Blend by hit confidence for seamless transition.

**Resolution:** Half-res tracing at Low/Medium (bilateral upsample). Full-res at Ultra. Half-res is indistinguishable from full-res on rough surfaces (reflection is blurred anyway).

### Budget

```
Volumetric fog total:  ~0.33ms
SSR (medium):          ~0.5ms
Combined:              ~0.83ms
```

# ScaleFast Engine — Characters, Gore, and Destruction

## Part 11: GPU Skinning, Dismemberment, and AI Adaptation

### GPU Skinning

All skeletal animation runs on the GPU via compute shader:

1. CPU evaluates blend tree → joint matrices (parallel per character via job system)
2. Upload joint matrices to GPU buffer (~6.4KB per character)
3. Compute shader skins vertices (position + normal + tangent)
4. Render skinned mesh in forward pass — identical to static mesh

30 characters: ~0.3ms compute skinning total.

### Indirect Batched Drawing

No traditional instancing (characters are unique — different meshes, poses, damage states). Instead:

- All skinned vertex buffers concatenated into one large GPU buffer
- Each character's draw recorded as an entry in an indirect draw buffer
- Single `vkCmdDrawIndexedIndirect` renders ALL characters
- Sort entries by material for minimal state changes
- Typical: 3-5 actual draw batches for 30 characters

### Character LOD (Screen-Space, Full-Rate Animation)

```
LOD 0 (large on screen):  20K+ tris, full IK, full cloth
LOD 1 (medium):           8-12K tris, foot IK only, no cloth
LOD 2 (small):            3-5K tris, no cosmetic IK
LOD 3 (tiny):             500-1K tris, simplified material

ALL LODS: full-rate animation blend tree, every frame.
TRANSITIONS: hard swap with hysteresis + motion masking.
No dithering. No animation rate reduction.
```

30 enemies in combat (~5 close, 10 medium, 10 far, 5 very far): ~244K triangles total. A single detailed environment mesh has more. Characters are NOT the rendering bottleneck.

### Gore and Dismemberment

**Pre-authored destruction meshes with runtime swapping:**

Artists create intact mesh + pre-segmented destruction pieces per limb. Each piece includes a "gore cap" (exposed cross-section mesh with flesh/bone material). Each piece has its own collision shape and ragdoll configuration.

**Shader-based limb culling (avoids combinatorial mesh variants):**

```cpp
// Each vertex has a limbID attribute
// Per-character bitmask of present limbs
if (!(limbMask & (1 << vertex.limbID))) {
    gl_Position = vec4(0, 0, 0, 0); // collapse triangle
}
```

One mesh, one bitmask. Any combination of severed limbs with zero additional memory.

### Dismemberment Pipeline

**At severance:**
1. **Mesh:** Clear limb bit in `limbMask`, enable gore cap mesh at stump
2. **Physics:** Severed limb → Jolt rigid body, velocity inherited + damage impulse
3. **Particles:** Blood burst at stump, persistent bleed emitter attached to body
4. **AI:** Behavior tree receives `limb_lost` event

**Post-severance AI adaptation:**
- Leg lost → crawling behavior, lower capsule, different navmesh clearance, altered animation set, IK disabled for missing leg
- Arm lost → can't use two-handed weapons, altered combat, IK disabled for missing arm
- Remaining limbs fully animated with IK
- Ragdoll blend weight can increase as enemy weakens (stumbling = more physics influence)

### MK-Style Viscera (Optional, Designer-Enabled)

Pre-modeled organ meshes placed inside torso (normally invisible inside intact mesh). When torso opens, they become visible. Some are rigid bodies (bounce), some are Jolt Cosserat rods (intestines — rope-like physics). Connective tissue via spring constraints.

The destruction is designer-authored. The aftermath is physical. Different every time because physics is procedural.

### Performance Impact

```
Active severed limbs: ~15 rigid bodies        → negligible (Jolt)
Gore cap meshes: ~10 small meshes             → ~500 extra tris
Stump emitters: ~10 × 50 particles each      → within particle budget
Blood decals: 20-30 on surfaces               → within decal budget
Animation for damaged AI: same cost as normal  → different clips, same eval
IK for damaged AI: CHEAPER                     → fewer limbs to solve

Total gore overhead: zero measurable frame time increase.
All gore systems are absorbed by existing budgets.
```

**UE5 comparison:** UE5's Chaos destruction system focuses on environment destruction with runtime fracturing. ScaleFast's character destruction is artist-driven with physics aftermath — more controlled, more predictable cost, more designer agency over the visual result.

# ScaleFast Engine — Pathfinding and AI Navigation

## Part 12: NavMesh, Dynamic Obstacles, and AI at Scale

### Navigation Meshes

NavMeshes are built at map build time from static collision geometry. Multiple navmesh layers per map for different agent types:

```
Humanoid:      1.8m height, 0.3m radius, 45° max slope
Large creature: 3.0m height, 1.0m radius, 35° max slope
Small creature: 0.5m height, 0.15m radius, 60° max slope
Vehicle:       1.5m height, 2.0m radius, 20° max slope, min turn radius
```

Pathfinding: A* over navmesh adjacency graph, funnel algorithm for shortest path within triangle corridor. Typically < 0.1ms per path.

### Dynamic Obstacle Handling

**Layer 1: Local Avoidance (per-frame, handles most obstacles)**

Each AI casts 3-5 short detection rays in its movement direction via Jolt raycasts. These see ALL geometry including dynamic rigid bodies. AI steers around single crates, NPCs, closing doors without involving the pathfinding system.

50 AIs × 5 raycasts = 250 raycasts/frame. Jolt handles thousands per ms.

**Layer 2: NavMesh Obstacle Carving (periodic, handles blockages)**

Dynamic rigid bodies above a size threshold register as navmesh obstacles. Their footprint is projected onto the navmesh, marking triangles as temporarily non-traversable. A* skips blocked triangles.

When a path is blocked: AI waits 0.5-1.0s (obstacle might move), then repaths. If no path exists, AI enters designer-configured "blocked" behavior state.

### Path Cache (Horde AI Optimization)

Multiple AIs targeting the same area compute nearly identical paths. Cache recent A* results:

```
Key:   (start_region, goal_region, agent_type)
Value: triangle corridor (A* result before string-pulling)

Cache hit: skip A*, do string-pulling only (unique per AI, cheap)
50 AIs all pursuing the player: 1 A* search + 49 cache hits.
Cost reduction: ~98%.
```

### Shared Sensing

**Visibility cache:** Spatial hash of recent "can X see Y?" queries. Nearby AIs reuse results. Invalidated every 0.25-0.5s.

**Raycast batching:** All AIs submit raycast requests to a central queue. Deduplicate similar rays. Single Jolt batch query. 250 rays → ~80-100 unique rays after dedup.

### Flying Enemies

Hybrid approach: sparse 3D waypoint graph for global navigation (designer-placed at transition points — doorways, vertical shafts) + steering behaviors for local movement (maneuver within rooms, dodge, circle).

### Vehicle Pathfinding

Modified A* on vehicle navmesh layer, post-processed with Dubins curves (forward-only, minimum turn radius) or Reeds-Shepp curves (allows reversing). PID controller follows computed curve. Jolt's vehicle constraint handles physics.

### AI Job System Integration

```
Job: AIPerception    — batched raycasts, visibility cache    ~0.3ms
Job: AIDecision      — behavior trees, state machines         ~0.1ms
Job: AIPathfinding   — A* for repath requests, cache lookup   ~0.2ms
Job: AISteering      — desired velocity, avoidance blend      ~0.1ms
Job: NavMeshCarving  — update blocked triangles               ~0.1ms
                                                              ─────
Total for 50 AIs:                                             ~0.8ms
```

### AI LOD (Distance-Based Update Rate)

```
Close (< 20m):   full update every frame
Medium (20-50m): update every 2nd frame, reduced raycasts
Far (50-100m):   update every 4th frame, simplified steering
Very far (100m+): update every 8th frame, interpolated movement
```

500 AIs with LOD: effective per-frame cost of ~120 full AIs. ~1.5ms.

# ScaleFast Engine — Networking

## Part 13: Server-Authoritative Multiplayer with Synchronized Physics

### Architecture: Dedicated Server, Client-Predicted

```
Server: authoritative simulation (physics, AI, IK, game logic)
Client: predicted local movement, interpolated remote entities
Trust: server validates everything, client trusted for nothing
```

### Transport

UDP with custom reliability layer:
- **Unreliable channel:** position snapshots, physics state (lost packet superseded by next)
- **Reliable-ordered channel:** game events (limb severed, door opened, objective captured)
- **Reliable-unordered channel:** asset streaming, chat

### Server Tick (60 Hz)

Each tick:
1. Receive client inputs (timestamped)
2. Process inputs in tick order
3. Run simulation (physics, AI, physics-coupled IK, game logic)
4. Build world snapshot
5. Delta-compress against each client's last ACK'd snapshot
6. Send per-client deltas

### Client-Side Prediction

Local player input is applied immediately — zero perceived latency. The client runs the exact same character controller code as the server (same Jolt physics, same character capsule). Given identical input and state, the result is identical (Jolt is deterministic).

**Why corrections are rare:** Movement is deterministic given input. The client has correct world geometry. The only misprediction source: other entities affecting the local player (explosions, being shot, collision). These are rare compared to frames of normal movement. 95%+ of frames: prediction is perfect.

**When correction is needed:**
1. Snap internal state to server's authoritative position at tick T
2. Replay all inputs from tick T to current tick (from stored history)
3. Visual difference blended over 2-8 frames in presentation layer
4. Simulation snaps immediately; rendering interpolates smoothly

This is why Overwatch feels perfect — same architecture, same principle. The client never perceives hard teleports even during large corrections.

### Remote Entity Interpolation

Other players are NOT predicted. They are interpolated between server snapshots:

```
render_time = current_time - interpolation_delay (2 × tick interval ≈ 33ms)
position = lerp(snapshot_T, snapshot_T+1, alpha)
```

Perfectly smooth, no jitter, no skipping. Cost: you see them 33ms in the past (invisible in gameplay at 60 tick/s). Dropped packet: extrapolate from last known velocity until next packet. One drop = ~16ms extrapolation. Imperceptible.

### Synchronized Rigid Bodies (Breaking the Industry Stigma)

**The industry says:** ragdolls must be client-side. **ScaleFast says:** the designer decides.

**Three sync modes:**

| Mode | Use Case | Bandwidth |
|------|----------|-----------|
| **ServerAuthoritative** | Doors, elevators, vehicles, gameplay-critical props | Full state per tick when active |
| **ImpulseSync** | Ragdolls, physics props, barrels, crates | Impulse events + periodic corrections |
| **ClientSide** | Debris, shell casings, cosmetic physics | Zero networking |

**Engine-enforced rule:** If `blocks_projectiles` or `affects_players` is true, sync mode MUST be ServerAuthoritative or ImpulseSync. The engine rejects ClientSide for gameplay-affecting bodies. A crate that blocks bullets must be synchronized.

### Ragdoll Synchronization (ImpulseSync)

Jolt's cross-platform determinism enables impulse-based sync:

**Death event (reliable, one-time):**
- Character ID, death impulse (direction + force), timestamp
- Client activates ragdoll at server tick, applies impulse locally
- Deterministic physics → nearly identical ragdoll trajectory

**Periodic correction (every 0.5s):**
- Server sends root bone position + velocity
- Client compares, smoothly blends if diverged (over 0.2s)

**Additional impulses (event-based):**
- Ragdoll kicked/shot/exploded → server sends impulse event
- Client applies, both simulate deterministically from there

**At rest:**
- Server sends final pose (all bones, ~400 bytes one-time)
- Client snaps, puts ragdoll to sleep

**Total bandwidth per ragdoll lifetime:** ~640 bytes.
Compare to full state sync: 25.2 KB/s continuous.

### Hit Detection (Valve-Style Lag Compensation)

```
Client: fires at what they SEE (sends tick + aim direction)
        Does NOT apply damage locally
        Plays muzzle flash, sound, tracer immediately (cosmetic)

Server: receives fire command at tick T
        Rewinds all hitboxes to tick T positions (from history buffer)
        Casts ray against rewound hitboxes
        Hit confirmed → apply damage at current tick
        Hit rejected → client was desynced or cheating
```

The server never trusts the client's claim of "I hit X." It verifies independently. Cheat sending "I headshot everyone" → server checks → rejects all impossible hits.

**Server history buffer:** 60 ticks × 34 entities × 30 bytes = ~61 KB. Trivial.
**Maximum rewind:** capped at 200-250ms (configurable). Beyond that, player latency is too high for fair compensation.

**Server-side fire culling:** Server does quick broad-phase check before full rewind. No entities near the ray? Skip rewind. 90% of shots miss — most skip the expensive rewind.

### Bandwidth Budget

```
8-player match, 30 AI:

Per client downstream:
  Player states:     ~8 KB/s  (7 players, delta compressed)
  AI states:         ~20 KB/s (30 enemies, sleep + delta)
  Synced rigid bodies: ~2-5 KB/s (impulse-based)
  Game events:       ~1-3 KB/s
  Total:             ~31-36 KB/s per client (~280 Kbps)

Per client upstream:
  Input packets:     ~6 KB/s (redundant last 3 inputs per packet)
  Fire events:       ~1-2 KB/s
  Total:             ~7-8 KB/s (~64 Kbps)

Server outbound (8 clients):
  ~288 KB/s (~2.3 Mbps)
```

### Server CPU Budget

```
Physics (Jolt):           ~2.0ms
AI (30 enemies):          ~0.8ms
Physics-coupled IK:       ~0.3ms
Character controllers:    ~0.2ms
Game logic:               ~0.5ms
Networking (serialize):   ~0.3ms
                          ──────
Total:                    ~4.1ms per tick (60 Hz = 16.67ms available)
Headroom:                 ~12.5ms
```

No GPU needed on server. Multiple match instances per server machine viable.

### Scaling

- **2-8 players:** trivial, everything works as described
- **8-16 players:** dedicated server preferred, AI may reduce
- **32+ players:** area-of-interest filtering, variable tick rates per distance, dedicated networking optimization work
- **64+ players:** fundamentally different architecture needed (not initial target)

**UE5 comparison:** UE5's networking is built for convenience with replication macros and automatic property sync. ScaleFast is explicit — the designer marks what gets networked, chooses the sync mode, and sees the bandwidth cost. UE5's approach is easier to use but harder to optimize. ScaleFast's approach requires more designer awareness but produces tighter, more predictable networking.

# ScaleFast Engine — World System and Streaming

## Part 14: Discrete Maps, id-Style Loading, Zero Hitches

### Philosophy

Discrete maps with aggressive precomputation at build time, surgical streaming at runtime, and load times so fast they feel like transitions. Open worlds are an engineering tar pit — 80% of budget managing what the player MIGHT see instead of optimizing what they ARE seeing.

**A map is a closed, bounded, fully known dataset.** The engine knows every piece of geometry, every light, every texture at build time. This enables:

- Complete visibility precomputation
- Optimal texture packing per map
- Pre-baked SDF volumes for particle collision
- Pre-built light cluster hints
- Shadow map pre-caching for static lights
- Memory budget fully known and bounded
- Pre-compiled shader pipeline state objects

### Map Structure

```
Sectors:
  Spatial subdivisions defining streaming granularity
  Each sector: bounding box, mesh refs, light refs, entity refs

Geometry:
  Static meshes: pre-batched, sorted by material
  Collision meshes: simplified, fed to Jolt as static bodies
  SDF volumes: pre-baked per sector for particle collision
  Occlusion proxy meshes for software occlusion culling

Visibility:
  Precomputed sector-to-sector visibility sets
  Runtime occlusion culling refines conservatively

Lights:
  All defined, shadow atlas regions pre-assigned
  Static shadow maps PRE-BAKED and loaded as raw data
```

### The Loading Model

**What loads at map start (the 2-4 second load):**
- All geometry (meshes, collision, occlusion proxy)
- All shader PSOs (pre-compiled, zero runtime compilation)
- Low-res texture set (minimum quality, render immediately)
- All light data + pre-baked static shadow maps
- All audio banks
- SDF volumes, visibility data

**What streams during gameplay (invisible):**
- High-res texture mip levels only

**Geometry never streams mid-map.** It's all resident from load. Only texture quality improves. Mip-level upgrades are inherently invisible — a texture going from 512² to 2048² while you walk toward it is imperceptible.

### Load Time Breakdown

```
1. Read header, allocate pools           ~50ms
2. Parallel decompress (Zstd, NVMe)     ~500-1500ms
3. Upload geometry to GPU                ~200-400ms
4. Create Jolt static bodies             ~100-200ms
5. PSO cache validation                  ~0-100ms
6. Initialize entities, map scripts      ~50-100ms
                                         ────────
                                Total:   ~1-2.5 seconds
```

The job system parallelizes decompression — each sector is an independent job. Start rendering as soon as spawn-point sectors load; background sectors finish while the player can't see them.

### Virtual Texturing

```
Build time:
  All map textures packed into virtual texture atlas
  Full mip chains generated, BC7/BC5 compressed
  64KB pages per mip level stored in map file

Runtime:
  GPU feedback buffer: shader writes accessed page IDs + mip levels
  CPU reads feedback (1 frame latency)
  Streaming prioritizes highest-demand pages
  Pages uploaded to physical texture cache (GPU atlas)
  LRU eviction for unused pages

Worst case: slightly blurry texture for 1-2 frames
  (lower mips always resident, loaded at map start)
```

### Pre-Compiled Pipeline State Objects

Every shader permutation compiled at build time (or first-run). Stored in pipeline cache per GPU/driver combination. Zero runtime shader compilation. Zero stutter.

**UE5 comparison:** UE5's shader compilation stutter is one of its most visible failures. ScaleFast knows every material in the map at build time. All PSOs are pre-cached. This problem simply doesn't exist.

### How This Feeds Other Systems

- **Shadows:** Static shadow maps baked at build time, loaded as raw data. First frame has perfect shadows at zero runtime cost
- **Particles:** SDF volumes precomputed per sector, resident immediately
- **Lights:** Cluster hints pre-identify high-density areas for grid optimization
- **Decals:** Static decals baked into map. Only player-caused decals are runtime

# ScaleFast Engine — Designer-Driven Optimization

## Part 15: Putting the Burden on Designers, Not the Runtime

### The Core Principle

UE5's fundamental sin is prioritizing developer convenience over runtime performance. Nanite means artists don't think about polycount. Lumen means designers don't think about light probe placement. The runtime pays for that convenience with frame time.

ScaleFast takes the opposite approach: **the engine provides powerful, optimized tools. The designer is responsible for using them wisely. The engine gives them the data to make informed decisions.** Build-time cost is acceptable. Runtime cost must be justified.

This is not about making designers' lives harder. It's about making the player's experience better. The designer has more control, more visibility, and more agency over the final result.

### Real-Time Budget Visualization

The engine surfaces per-system budget overlays that designers cannot ignore:

```
Shadow budget:    ████████░░ 1.2ms / 1.5ms allocated
Volumetrics:      █████░░░░░ 0.3ms / 0.5ms allocated
Particles:        ██████████ 0.8ms / 0.8ms allocated  ← AT LIMIT
Forward pass:     ████████░░ 3.2ms / 4.0ms allocated
Post-process:     ██████░░░░ 0.9ms / 1.5ms allocated

Frame total:      ████████░░ 6.4ms / 6.9ms (144 FPS)

WARNINGS:
⚠ 3 lights in sector_07 exceed shadow budget
⚠ Emitter "explosion_large" peak particle count: 85K (limit: 100K)
⚠ Decal density in sector_12: 47 overlapping (cluster limit: 32)
```

Not a vague framerate drop — specific, actionable budget violations. The designer knows which light, which emitter, which sector is the problem.

### What Designers Control

**Per-light shadow settings:**
- Shadow mode (none/static/dynamic)
- Priority (ForcedOn through ForcedOff)
- Maximum resolution
- Update rate
- Shadow distance and fade
- Shadow intensity (how dark the shadow is)
- Particle shadow response
- Static caching enabled/disabled

**Per-character LOD configuration:**
- Screen-space pixel thresholds for each LOD level
- Per-IK-chain override (never_disable for critical chains)
- Physics coupling flag (prevents engine from disabling IK)
- IK blend speed (physical weight feel)
- Cloth/soft body disable threshold

**Per-rigid-body networking:**
- Sync mode (ServerAuthoritative / ImpulseSync / ClientSide)
- Correction interval and blend time
- Priority for update frequency
- Engine enforces: gameplay-affecting bodies MUST be synced

**Per-emitter particle settings:**
- Maximum particle count
- Collision tier (depth buffer / SDF / Jolt rigid body)
- Shadow casting enabled

**Per-light shadow intensity:**
- Simulates ambient fill without runtime GI
- Different per light based on context (harsh sun vs soft fill)

### What the Engine Optimizes Automatically

The engine optimizes what it can PROVE is safe:

- **Mesh LOD** based on screen-space size (with designer-set thresholds)
- **Texture mip streaming** based on screen-space UV density
- **Shadow atlas management** — static shadows cached, only dirty faces re-rendered
- **VSM page caching** — only invalidated pages re-rendered
- **Cluster culling** — lights/decals/fog volumes assigned to clusters, irrelevant ones never evaluated
- **Frustum + occlusion culling** — invisible geometry never submitted to GPU
- **Particle frustum culling** — off-screen particles still simulate but don't render
- **AI LOD** — distant AI update less frequently (this is AI logic rate, NOT animation rate)

### What the Engine NEVER Optimizes Without Permission

- **Physics-coupled IK** — if marked `ik_drives_physics`, the chain runs every frame regardless of distance. The engine cannot silently break gameplay.
- **Animation blend tree rate** — always full framerate for every visible character. No exceptions.
- **Shadow-casting for ForcedOn lights** — the designer said this light matters. The engine respects that.
- **Networked rigid body sync** — if the designer marked it as gameplay-relevant, it stays synchronized.

### The Tooling Mandate

Every system exposes its cost to the designer:

```
Character inspector:
  "This mech has IK never-disable on 4 leg chains.
   CPU cost at 500m: 0.02ms. At max count (5 mechs): 0.1ms."

Light inspector:
  "This spotlight: shadow resolution 1024, update every frame.
   GPU cost: 0.15ms. Shadow atlas usage: 4MB."

Emitter inspector:
  "Peak particle count: 85K. GPU sim: 0.08ms.
   Shadow density injection: enabled (adds ~0.05ms to sun shadow).
   Collision: Tier 1 (depth buffer)."

Sector inspector:
  "Total lights: 47 (12 shadow-casting).
   Estimated forward pass at 4K: 3.8ms.
   Decal density peak: 23 overlapping.
   Static geometry: 1.2M triangles."
```

The designer makes informed decisions. The engine executes them efficiently. The player gets the best possible experience.

### UE5 Comparison Summary

| Aspect | UE5 Approach | ScaleFast Approach |
|--------|--------------|-------------------|
| Geometry complexity | Nanite handles it at runtime | Artist authors LODs, engine validates silhouette quality |
| Lighting | Lumen adapts automatically | Designer places lights, sets shadow intensity, manages budget |
| Shadow casting | Limited, engine decides | Every light CAN cast shadows, designer budgets explicitly |
| Shader compilation | Runtime, causes stutter | Build-time, zero runtime compilation |
| Streaming | Runtime for everything, causes hitches | Geometry resident at load, only texture mips stream |
| AI optimization | Mostly manual profiling | Budget overlay shows per-AI cost, cache sharing automatic |
| Physics networking | Automatic replication | Designer chooses sync mode per body, engine enforces rules |
| Performance debugging | Profiler tools (Unreal Insights) | In-viewport budget overlay, always visible during design |
| Optimization burden | Runtime systems adapt (expensive) | Designers optimize at design time (cheap at runtime) |
| Frame target | 30-60 FPS console-first | 144+ FPS PC-first, settings system scales to hardware |

The philosophy: **make it the designer's problem at build time so it's never the player's problem at runtime.**

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
