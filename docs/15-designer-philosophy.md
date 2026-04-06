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
