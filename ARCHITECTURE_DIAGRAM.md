# Architecture Diagram and Data Flow

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      PINTOS KERNEL                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────┐              ┌──────────────────────┐     │
│  │  Thread 1        │              │  Thread 2            │     │
│  │                  │              │                      │     │
│  │ calls            │              │ calls                │     │
│  │ timer_sleep(100) │              │ timer_sleep(200)     │     │
│  │                  │              │                      │     │
│  │ BLOCKED on       │              │ BLOCKED on           │     │
│  │ semaphore        │              │ semaphore            │     │
│  └──────────────────┘              └──────────────────────┘     │
│           ▲                                 ▲                    │
│           │                                 │                    │
│           └─────────────┬────────────────────┘                   │
│                         │                                        │
│           ┌─────────────▼──────────────────┐                   │
│           │     sleeping_list               │                   │
│           ├──────────────────────────────────┤                   │
│           │ [1] → [2] → [3] → ... → [null]  │                   │
│           │                                  │                   │
│           │ Each node:                       │                   │
│           │ - wake_tick: time to wake       │                   │
│           │ - sema: semaphore to signal    │                   │
│           │ - elem: list linkage           │                   │
│           └─────────────▲──────────────────┘                   │
│                         │                                        │
│           protected by:  │                                       │
│           ┌─────────────┴──────────────┐                       │
│           │    sleep_lock (mutex)       │                       │
│           └────────────────────────────┘                       │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐      │
│  │        Timer Interrupt Handler                         │      │
│  ├───────────────────────────────────────────────────────┤      │
│  │ (fires every timer tick)                              │      │
│  │                                                        │      │
│  │ ┌─ ticks++                                           │      │
│  │ ├─ thread_tick()                                     │      │
│  │ │                                                    │      │
│  │ ├─ for each thread in sleeping_list:               │      │
│  │ │  ├─ if (ticks >= wake_tick):                     │      │
│  │ │  │  ├─ list_remove(thread)                       │      │
│  │ │  │  ├─ sema_up(&thread->sema) ◄──┐             │      │
│  │ │  │  │                             │             │      │
│  │ │  │  └─ Wake thread!              │             │      │
│  │ │  └─ else: continue               │             │      │
│  │ │                                   │             │      │
│  │ └─ return from interrupt            │             │      │
│  │                                      │             │      │
│  └──────────────────────────────────────┼─────────────┘      │
│                                          │                     │
│                 (interrupts automatically OFF)                  │
│                                          │                     │
│  ┌───────────────────────────────────────┼─────────────┐      │
│  │  Thread wakes up (BLOCKED → READY)    │             │      │
│  │                                        ▼             │      │
│  │  ┌──────────────────────────────────────────────┐   │      │
│  │  │ Thread receives signal from sema_up          │   │      │
│  │  ├──────────────────────────────────────────────┤   │      │
│  │  │ - Removed from sleeping_list                 │   │      │
│  │  │ - free(sleeping_thread structure)            │   │      │
│  │  │ - Resumes execution                          │   │      │
│  │  │ - Returns from timer_sleep()                 │   │      │
│  │  │ - Back to normal execution                   │   │      │
│  │  └──────────────────────────────────────────────┘   │      │
│  │                                                       │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Call Flow Diagram

### Thread's Execution Path

```
User Thread
    │
    └─ timer_sleep(100 ticks)
        │
        ├─ Allocate sleeping_thread structure
        │
        ├─ Calculate wake_tick = current_ticks + 100
        │
        ├─ Initialize semaphore (value = 0)
        │
        ├─ Acquire sleep_lock
        │   │
        │   ├─ list_push_back(&sleeping_list, &st->elem)
        │   │
        │   └─ Release sleep_lock
        │
        ├─ sema_down(&st->sema)
        │   │
        │   └─ BLOCKS HERE ◄─────────────────┐
        │       (thread state = BLOCKED)      │
        │       (CPU switches to other work)  │
        │                                     │
        │   [Time passes]                   │
        │   [Other threads run]             │
        │   [Timer interrupts every tick]   │
        │                                     │
        │   [Eventually...]                 │
        │   [Timer tick >= wake_tick]       │
        │                                     │
        │                                     │
        ├─ [Signal received from interrupt] ◄┘
        │
        ├─ free(st)
        │
        └─ Return from timer_sleep()
```

### Timer Interrupt Path

```
Hardware Timer
    │
    └─ Fire Interrupt (every 10ms by default)
        │
        └─ timer_interrupt() handler
            │
            ├─ ticks++
            │
            ├─ thread_tick()
            │
            ├─ for (e = list_begin; e != list_end; e = list_next)
            │   │
            │   ├─ st = list_entry(e, struct sleeping_thread, elem)
            │   │
            │   ├─ if (ticks >= st->wake_tick)
            │   │   │
            │   │   ├─ next = list_next(e)
            │   │   │
            │   │   ├─ list_remove(e)  ◄─ Remove from sleeping list
            │   │   │
            │   │   ├─ sema_up(&st->sema)  ◄─ Wake the thread!
            │   │   │
            │   │   └─ e = next
            │   │
            │   └─ else
            │       └─ e = list_next(e)
            │
            └─ Return from interrupt
                (thread now READY, will be scheduled)
```

## Memory Layout

```
┌───────────────────────────────────────────────────────────┐
│                    Kernel Memory                           │
├───────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │ Global Variables in devices/timer.c               │   │
│  ├────────────────────────────────────────────────────┤   │
│  │ static int64_t ticks                               │   │ 8 bytes
│  │ static unsigned loops_per_tick                     │   │ 4 bytes
│  │ static struct list sleeping_list                   │   │ 16 bytes
│  │ static struct lock sleep_lock                      │   │ ~40 bytes
│  └────────────────────────────────────────────────────┘   │
│                         ▼                                   │
│  ┌────────────────────────────────────────────────────┐   │
│  │ sleeping_list (list head)                          │   │
│  ├────────────────────────────────────────────────────┤   │
│  │ head ─┐                                             │   │
│  │ tail ─┼──────┐                                      │   │
│  └──────┼───┬──┼──────────────────────────────────────┘   │
│         │   │  │                                           │
│         │   │  └─────────────────────────────────┐        │
│         │   │                                   │        │
│         ▼   ▼                                   ▼        │
│  ┌─────────────────────┐               ┌─────────────────────┐
│  │ sleeping_thread [1] │────────────── │ sleeping_thread [2] │
│  ├─────────────────────┤               ├─────────────────────┤
│  │ wake_tick: 250      │               │ wake_tick: 500      │
│  │ sema {              │               │ sema {              │
│  │   value: 0          │               │   value: 0          │
│  │   waiters: []       │               │   waiters: [T2]     │
│  │ }                   │               │ }                   │
│  │ elem: {             │               │ elem: {             │
│  │   prev: [1]         │               │   prev: [2]         │
│  │   next: null        │               │   next: null        │
│  │ }                   │               │ }                   │
│  └─────────────────────┘               └─────────────────────┘
│  (allocated on heap)                   (allocated on heap)
└───────────────────────────────────────────────────────────┘
```

## Synchronization Points

```
┌──────────────────────────────────────────────┐
│       SYNCHRONIZATION POINTS                  │
├──────────────────────────────────────────────┤
│                                               │
│  1. LIST ACCESS (Protected by sleep_lock)   │
│     ┌─────────────────────────────────────┐ │
│     │ lock_acquire(&sleep_lock)           │ │
│     │   list_push_back(...)               │ │
│     │ lock_release(&sleep_lock)           │ │
│     │                                     │ │
│     │ Time in lock: ~microseconds         │ │
│     │ Interrupts: Disabled during lock    │ │
│     └─────────────────────────────────────┘ │
│                                               │
│  2. THREAD BLOCKING (Atomic operation)      │
│     ┌─────────────────────────────────────┐ │
│     │ sema_down(&st->sema)                │ │
│     │   - Atomically checks count         │ │
│     │   - If 0: blocks thread             │ │
│     │   - If >0: decrements and continues │ │
│     │                                     │ │
│     │ Thread state: BLOCKED               │ │
│     │ CPU available for other work        │ │
│     └─────────────────────────────────────┘ │
│                                               │
│  3. INTERRUPT HANDLER (Atomic by default)   │
│     ┌─────────────────────────────────────┐ │
│     │ timer_interrupt()                   │ │
│     │   - Interrupts already disabled     │ │
│     │   - List access is safe             │ │
│     │   - sema_up(&st->sema)              │ │
│     │     atomically increments count     │ │
│     │                                     │ │
│     │ Woken thread: state = READY         │ │
│     │ Will be scheduled by kernel         │ │
│     └─────────────────────────────────────┘ │
│                                               │
└──────────────────────────────────────────────┘
```

## Before vs After Comparison

### BEFORE (Busy-Wait - INEFFICIENT)

```
┌─────────────────────────────────────┐
│  timer_sleep(100)                   │
├─────────────────────────────────────┤
│                                     │
│ start = timer_ticks()    [say: 350] │
│                                     │
│ while (timer_elapsed < 100)         │
│   ├─ Check: 350+100=450             │
│   ├─ Current: 350, not yet >= 450   │
│   ├─ yield() ─ THREAD KEEPS RUNNING │
│   ├─ Check: 360, not yet >= 450     │
│   ├─ yield() ─ THREAD KEEPS RUNNING │
│   ├─ ... (many times) ...           │
│   ├─ Check: 449, not yet >= 450     │
│   ├─ yield() ─ THREAD KEEPS RUNNING │
│   ├─ Check: 450, YES! >= 450        │
│   └─ Exit loop                      │
│                                     │
│ RESULT:                             │
│ ✗ CPU wasted spinning               │
│ ✗ Other threads starved             │
│ ✗ 100% CPU usage for one thread     │
│ ✓ Wake time accurate                │
└─────────────────────────────────────┘
```

### AFTER (Semaphore-Based - EFFICIENT)

```
┌──────────────────────────────────────┐
│  timer_sleep(100)                    │
├──────────────────────────────────────┤
│                                      │
│ wake_tick = timer_ticks() + 100      │
│           = 350 + 100 = 450          │
│                                      │
│ Create sleeping_thread               │
│ Add to sleeping_list                 │
│                                      │
│ sema_down() ─ THREAD BLOCKS          │
│                                      │
│ [CPU switches to other threads]      │
│ [Time passes: ticks go 350→450]      │
│ [Other work gets done]               │
│ [Timer interrupts periodically]      │
│                                      │
│ [Timer tick 450]                     │
│ timer_interrupt():                   │
│   if (ticks >= wake_tick):           │
│     sema_up() ─ WAKE THREAD          │
│                                      │
│ [Thread wakes, freed, continues]     │
│                                      │
│ RESULT:                              │
│ ✓ 0% CPU while sleeping              │
│ ✓ Other threads can run              │
│ ✓ System efficient                   │
│ ✓ Wake time accurate                 │
└──────────────────────────────────────┘
```

## Data Structure Details

### struct sleeping_thread

```
┌─────────────────────────────────────────────┐
│         sleeping_thread                      │
├─────────────────────────────────────────────┤
│                                              │
│ +0  ┌──────────────────────────────────┐   │
│     │ wake_tick (int64_t)              │ 8 │
│     │ = Current time + sleep duration  │   │
│     └──────────────────────────────────┘   │
│                                              │
│ +8  ┌──────────────────────────────────┐   │
│     │ sema (struct semaphore)          │ ~ │
│     │ {                                │ 1 │
│     │   value: 0                       │ 6 │
│     │   waiters: [thread ptr]          │   │
│     │ }                                │ b │
│     └──────────────────────────────────┘   │
│                                              │
│ +24 ┌──────────────────────────────────┐   │
│     │ elem (struct list_elem)          │ ~ │
│     │ {                                │ 1 │
│     │   prev: [previous node]          │ 6 │
│     │   next: [next node]              │   │
│     │ }                                │ b │
│     └──────────────────────────────────┘   │
│                                              │
│ Total size: ~40-50 bytes                    │
│                                              │
└─────────────────────────────────────────────┘
```

## State Transitions

```
                  THREAD LIFECYCLE DURING SLEEP

    ┌──────────────────────────────────────────┐
    │  RUNNING STATE                           │
    │                                          │
    │  Thread executes timer_sleep()           │
    └──────────────┬───────────────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────────────┐
    │  READY STATE (briefly)                   │
    │                                          │
    │  - Allocate sleeping_thread              │
    │  - Calculate wake_tick                   │
    │  - Add to sleeping_list                  │
    │  - Initialize semaphore                  │
    └──────────────┬───────────────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────────────┐
    │  BLOCKED STATE (SLEEPING)                │
    │                                          │
    │  - sema_down() blocks thread             │
    │  - Removed from ready queue              │
    │  - In sleeping_list                      │
    │  - Waiting on semaphore                  │
    │                                          │
    │  Duration: wake_tick - current_ticks     │
    │  CPU: 0% (can run other threads)         │
    └──────────────┬───────────────────────────┘
                   │
    [Time passes - timer ticks]
    [When current_ticks >= wake_tick]
                   │
                   ▼
    ┌──────────────────────────────────────────┐
    │  INTERRUPT HANDLER                       │
    │                                          │
    │  - Checks sleeping_list                  │
    │  - Finds thread with matching wake_tick  │
    │  - Calls sema_up()                       │
    └──────────────┬───────────────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────────────┐
    │  READY STATE (WOKEN)                     │
    │                                          │
    │  - Removed from sleeping_list            │
    │  - Added back to ready queue             │
    │  - sema_down() returns                   │
    │  - Freed from blocked state              │
    └──────────────┬───────────────────────────┘
                   │
                   ▼
    ┌──────────────────────────────────────────┐
    │  RUNNING STATE                           │
    │                                          │
    │  - free(sleeping_thread structure)       │
    │  - Return from timer_sleep()             │
    │  - Continue normal execution             │
    └──────────────────────────────────────────┘
```

## Efficiency Comparison

```
CPU USAGE GRAPH

               │  Busy-Wait
            100│   ┌─────────────
               │   │
CPU Usage (%)  │   │  Semaphore-Based
            50│ ┌─┘
               │ │
             0└─┴─────────────
               └──────────────────── Time
               0   50    100   150

Busy-Wait:     100% for entire sleep duration
Semaphore:       0% after sema_down()
               (0% CPU usage per sleeping thread!)

Other threads have 100% CPU available with semaphore approach!
```

---

This architecture ensures:
- ✅ No CPU waste (0% while sleeping)
- ✅ Proper thread state management
- ✅ Safe synchronization
- ✅ Efficient system operation
- ✅ Correct wake timing

