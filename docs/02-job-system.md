# ScaleFast Engine — Job System

## Part 2: Fiber-Based Job System (No Main Thread)

### Inspiration

DOOM Eternal (2020) and DOOM: The Dark Ages (2025) run on a fiber-based job system with no dedicated main thread. Every piece of work — input, physics, animation, rendering command recording, submission — is a job. No thread is special. Any worker picks up any job. On Linux, this architecture scales to infinity, reaching 1000+ FPS when other bottlenecks are removed.

ScaleFast adopts this architecture wholesale.

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                Job System (no main thread)            │
│                                                      │
│  N worker threads (= hardware thread count)          │
│  Each worker:                                        │
│    - Pulls from local deque (LIFO, cache-warm)       │
│    - Steals from others (FIFO, load balance)         │
│    - Runs fibers, not callbacks                      │
│                                                      │
│  Fibers:                                             │
│    - Lightweight (~64KB stack, pooled)                │
│    - Can yield/wait without blocking the worker      │
│    - When a job waits on a dependency, the fiber     │
│      suspends and the worker picks up new work       │
│                                                      │
│  No thread ever idles. No thread is "special."       │
└──────────────────────────────────────────────────────┘
```

### Why Fibers, Not Callbacks

In a callback-based task system, if Job A needs the result of Job B:
1. Block the worker thread (terrible — one fewer worker)
2. Split Job A into A1 and A2, making A2 depend on B (explosion of small tasks, complex graph)

With fibers, Job A says "wait on this counter." The fiber suspends, the worker immediately picks up another fiber/job, and when B completes (decrementing the counter), A's fiber becomes runnable again. **Any** worker can resume it. Zero idle time.

### The Wait-Free Counter Primitive

Every synchronization point is an atomic counter:

```cpp
struct JobCounter {
    std::atomic<int32_t> value;
    // When value hits 0, all fibers waiting on this become runnable
};
```

- `KickJobs(jobs[], count, &counter)` — enqueues jobs, sets counter to count
- `WaitForCounter(counter, targetValue)` — suspends current fiber, resumes when counter reaches target
- Job completion atomically decrements the counter

No mutexes. No condition variables. No kernel transitions in the hot path.

### Work-Stealing Deques

Each worker has a Chase-Lev lock-free deque:
- Tasks spawned by a fiber push locally (LIFO — cache-warm, the spawning worker likely processes them)
- Other workers steal from the opposite end (FIFO — load balancing)
- Excellent cache locality for the spawning thread while distributing load across cores

### Fiber Context Switching

No Boost. No `ucontext` (unnecessary signal mask save/restore overhead). Minimal inline assembly for x86-64:

```asm
; ~20 instructions total
; save: push rbx, rbp, r12-r15, save rsp to context
; restore: load rsp from new context, pop r15-r12, rbp, rbx, ret
```

Fiber stacks are pre-allocated from a pool (~256-512 fibers, 64KB each). Allocated with `mmap` on Linux, `VirtualAlloc` on Windows, with guard pages.

**Important:** Thread-local storage (TLS) does not work with fibers (a fiber can resume on a different thread). Fiber-local storage is used instead — a pointer stored in the fiber context.

### Frame as a Job Graph

```
Frame N kicks off when Frame N-2 presents (triple buffered):

  ┌──────────┐
  │  Input   │
  └────┬─────┘
       │
  ┌────▼─────┐     ┌───────────┐
  │ Physics  │────▶│ Animation │
  │ Kick     │     │ Update    │
  └────┬─────┘     └─────┬─────┘
       │                 │
  ┌────▼─────────────────▼─────┐
  │     Visibility / Culling    │
  └────────────┬───────────────┘
               │
    ┌──────────▼──────────┐
    │  Light Cluster Build │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │ Cmd Buffer Recording │
    │ + Shadow Maps        │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │  Submit + Present    │
    └──────────────────────┘
```

No "submit thread" or "render thread." The job that finishes recording last also does the submit. It's the natural tail of the dependency chain.

### Simulation / Presentation Split

Built in from day one, essential for both smooth rendering and multiplayer:

```cpp
frame_simulate(dt);      // fixed tick rate (60 Hz)
  // physics, AI, physics-coupled IK, game logic
  // In multiplayer: runs on server
  // In single player: runs locally

frame_present(alpha);    // render framerate (variable, 144+ Hz)
  // alpha = interpolation between last two sim states
  // animation blend trees, cosmetic IK, cloth, rendering
  // Always runs on client
```

### Linux Scheduling Advantage

- No DPC/ISR overhead eating frame time
- CFS/EEVDF pins N consistently runnable threads across cores
- `SCHED_FIFO` / `SCHED_DEADLINE` for real-time scheduling of workers
- `madvise(MADV_HUGEPAGE)` for fiber stack pool eliminates TLB misses
- Vulkan on Mesa (RADV, ANV) has less driver overhead than Windows in many cases

### JoltPhysics Job System Integration

Jolt's `JobSystem` is an abstract interface explicitly designed for custom implementations. Their reference thread pool is documented as "an example — provide your own."

```cpp
class EngineJobSystem : public JPH::JobSystemWithBarrier {
    FiberScheduler* scheduler;

    void QueueJob(JPH::Job* job) override {
        job->AddRef();
        scheduler->submit([job] {
            job->Execute();
            job->Release();
        });
    }

    void WaitForJobs(Barrier* barrier) override {
        while (barrier->HasPendingJobs()) {
            if (!try_execute_barrier_job(barrier))
                fiber_yield();
        }
    }
};
```

No TLS dependency in Jolt's hot path. A fiber can start a Jolt job on thread 3 and resume on thread 7. No breakage. This is why PhysX was rejected — its TLS dependency conflicts fundamentally with fiber-based scheduling.
