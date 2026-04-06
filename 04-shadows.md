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
