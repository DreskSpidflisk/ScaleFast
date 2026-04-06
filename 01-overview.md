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
