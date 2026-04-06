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
