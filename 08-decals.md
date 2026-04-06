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
