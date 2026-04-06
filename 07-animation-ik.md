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
