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
