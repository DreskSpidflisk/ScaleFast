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
