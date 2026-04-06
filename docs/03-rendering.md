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
