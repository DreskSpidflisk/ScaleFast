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
