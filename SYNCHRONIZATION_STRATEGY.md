# Synchronization Strategy for timer_sleep()

## Problem Statement

The original `timer_sleep()` used busy-waiting:
```c
while (timer_elapsed(start) < ticks)
    thread_yield();
```

This wastes CPU cycles and violates Pintos design principles. We need a proper synchronization approach.

## Solution: Semaphore-Based Sleep

### Why Semaphores?

1. **Block Without Wasting CPU:** Semaphores put threads into blocked state
2. **Wake at Precise Time:** Timer interrupt can signal when time is right
3. **No Polling:** No need for thread to check repeatedly
4. **Simple Interface:** Just `sema_down()` to sleep, `sema_up()` to wake

### Synchronization Components

#### 1. Global Data Protection: `sleep_lock`
```c
static struct lock sleep_lock;
```

**Purpose:** Protect the `sleeping_list` from concurrent access

**When Held:**
- During `list_push_back()` in timer_sleep()
- During list iteration in timer_interrupt()
- During `list_remove()` in timer_interrupt()

**Why Needed:**
- Multiple threads call `timer_sleep()` concurrently
- Without lock, list corruption would occur
- Interrupt handler also accesses list

**Duration:**
- Held ONLY during list operations
- Released BEFORE calling `sema_down()` (thread blocks)
- Never held while thread is sleeping

#### 2. Per-Thread Blocking: `struct semaphore`
```c
struct semaphore sema;  // In sleeping_thread structure
```

**Purpose:** Block individual thread until wake time

**How It Works:**
- Initial value: 0 (binary semaphore)
- Thread calls: `sema_down(&st->sema)` → BLOCKS
- Interrupt calls: `sema_up(&st->sema)` → WAKES

**Why This Works:**
- Semaphore atomically blocks without wasting CPU
- Each thread has its own semaphore (independent wake times)
- Signal happens from interrupt (safe, no sleeping required)

---

## Detailed Synchronization Flow

### Thread's View

```
Thread Context (with interrupts ON):
│
├─ timer_sleep(100) is called
│
├─ Allocate sleeping_thread *st
│
├─ Lock {
│   list_push_back(&sleeping_list, &st->elem)
│   } Unlock
│
│  [At this point, thread is in the list but NOT yet sleeping]
│
├─ sema_down(&st->sema) 
│  │
│  └─ BLOCKS HERE (goes into BLOCKED state)
│     CPU switches to other threads
│
│  [Time passes... other threads run...]
│
│  [Timer interrupt fires...]
│  [Interrupt wakes this thread via sema_up()]
│  │
│  └─ WAKES UP (goes into READY state)
│
├─ free(st)
│
└─ Return from timer_sleep()
```

### Interrupt Handler's View

```
Timer Interrupt (with interrupts automatically OFF):
│
├─ ticks++
│
├─ thread_tick()
│
├─ For each sleeping_thread in sleeping_list:
│  │
│  ├─ If (current_ticks >= wake_tick):
│  │  │
│  │  ├─ list_remove(&st->elem)
│  │  │
│  │  ├─ sema_up(&st->sema)
│  │  │   └─ This wakes the thread!
│  │  │
│  │  └─ free() done by thread after it wakes
│  │
│  └─ Else: move to next thread
│
└─ Return from interrupt
```

---

## Preventing Race Conditions

### Race Condition 1: List Corruption

**Scenario:** Thread modifying list while interrupt tries to read

**Solution:**
- Thread uses `sleep_lock` when accessing list
- Interrupt handler runs with interrupts OFF (atomic)
- No actual interrupt while holding lock (interrupts already disabled in interrupt handler)

```
Thread: {
  lock_acquire(&sleep_lock);
  list_push_back(...);
  lock_release(&sleep_lock);  // Release before sleeping
}

// Sleep here (no list access)
sema_down(&st->sema);

Interrupt:
  // Interrupts are automatically disabled
  // Safe to access list
  while (e != list_end(...))
    // Process each thread
```

### Race Condition 2: Wake Signal Lost

**Scenario:** Interrupt tries to wake thread that's not yet sleeping

**Solution:**
- Create list entry BEFORE sleep
- Use semaphore with proper initialization
- Interrupt finds thread in list and signals it

```
Correct Order:
1. Allocate structure
2. Add to list (with lock)
3. Release lock
4. Block on semaphore

Interrupt can then:
- Find thread in list
- Signal semaphore
- Thread wakes
```

### Race Condition 3: Double Wake

**Scenario:** Thread woken twice (shouldn't happen but let's prevent it)

**Solution:**
- Thread removed from list when woken
- Interrupt only processes threads still in list
- Can't wake twice

```
Interrupt checks: if (ticks >= wake_tick)
Removes from list IMMEDIATELY
Signals semaphore

Even if interrupt fires again:
- Thread not in list anymore
- Can't process it again
```

### Race Condition 4: Early Free

**Scenario:** Thread freed before semaphore signal completes

**Solution:**
- Thread doesn't free until AFTER `sema_down()` returns
- Thread in blocked state when free could theoretically happen
- But thread is in BLOCKED state (safe to use structure)

```
Thread:
  sema_down(&st->sema)  // Blocks here
  // ... wake up happens ...
  free(st)              // Free AFTER wake

Interrupt:
  sema_up(&st->sema)    // Wakes thread
  // Thread still valid until sema_down returns
```

---

## Minimum Interrupt Disabling

The implementation minimizes interrupt disabling:

### Where Interrupts Are Disabled
1. **In thread code:** Only during `lock_acquire/release`
   - Lock internally uses `intr_disable()`
   - Very brief (~microseconds)

2. **In interrupt handler:** Already disabled (automatic)
   - Interrupt handlers run with interrupts off
   - No additional disabling needed

### Where Interrupts Are Enabled
1. **Thread sleeping:** YES - `sema_down()` allows interrupts
   - Thread blocks and other threads run
   - Timer interrupt can fire (as needed)
   - This is desired! We want interrupts during sleep

2. **Thread executing:** YES - except during critical sections
   - Lock is held only briefly
   - Most of execution has interrupts on

---

## Correctness Guarantees

### Guarantee 1: Threads Wake Exactly Once
- Thread removed from list when woken
- Interrupt can't find it again
- Won't be woken twice

### Guarantee 2: Wake Time Accuracy
- Interrupt checks: `if (ticks >= wake_tick)`
- Wakes as soon as condition is true
- Accuracy: ±1 timer tick (acceptable per spec)

### Guarantee 3: No Missed Wakes
- Thread added to list BEFORE sleep
- Interrupt processes list every tick
- If thread's wake time comes, it WILL be woken

### Guarantee 4: No Memory Leaks
- Structure allocated on sleep entry
- Structure freed on sleep exit
- One allocation, one deallocation

### Guarantee 5: No Busy-Waiting
- Thread blocks on semaphore
- No loop checking condition repeatedly
- Zero CPU usage while sleeping

---

## Comparison with Other Approaches

### Approach 1: Just Use Lock (WRONG)
```c
sema_init(&global_sema, 0);
sema_down(&global_sema);  // All threads block on same sema!
```
**Problem:** All threads wake when any thread's time comes

### Approach 2: Condition Variable (ALSO WORKS)
```c
struct condition cond;
lock_acquire(&lock);
while (current_time < wake_time)
    cond_wait(&cond, &lock);
lock_release(&lock);
```
**Issue:** More overhead, similar functionality to semaphore approach

### Approach 3: Semaphore Per Thread (OUR APPROACH - BEST)
```c
struct sleeping_thread {
    struct semaphore sema;  // Each thread has its own!
    int64_t wake_tick;
};
```
**Benefit:** Simple, efficient, per-thread control

---

## Deadlock Impossibility

### Potential Deadlock Scenarios

**Scenario 1:** Thread holding lock waits for interrupt signal
- **Not possible:** Lock released before `sema_down()`
- Thread can be interrupted while sleeping

**Scenario 2:** Interrupt tries to acquire lock
- **Not possible:** Interrupt runs with interrupts already off
- Lock acquire would fail if thread already holds it
- But interrupts are off, so thread isn't even running

**Scenario 3:** Circular wait between threads
- **Not possible:** All threads compete for same simple lock
- No multiple locks to create deadlock
- Lock is released before blocking

### Conclusion: **Deadlock-Free** Design

---

## Performance Analysis

### Time Complexity
- `timer_sleep()`: O(1) allocation + O(1) list operation
- `timer_interrupt()`: O(n) where n = number of sleeping threads
  - Only checks threads that are actually sleeping
  - Not checking every thread in system

### Space Complexity
- O(n) where n = number of sleeping threads
- Each sleeping thread adds ~40 bytes to system

### CPU Usage
- **Sleeping threads:** 0% CPU (blocked)
- **Interrupt handler:** ~1% CPU (runs every tick)
- **Comparison:** Busy-wait would be ~100% CPU per sleeping thread

---

## Debugging Hints

### How to Verify Correct Synchronization

1. **Check no busy-wait:**
   ```bash
   grep -c "while.*yield" timer.c
   # Should be 0 (or only in comments)
   ```

2. **Check lock is used:**
   ```bash
   grep -c "sleep_lock" timer.c
   # Should be > 0
   ```

3. **Check semaphore per thread:**
   - Each `sleeping_thread` has own semaphore
   - Prevents global wake-up

4. **Check thread in list before sleep:**
   - `list_push_back()` called with lock
   - `sema_down()` called after lock release
   - Thread findable in list while sleeping

5. **Check interrupt handler:**
   - Iterates list
   - Removes thread when waking
   - Signals individual semaphore

---

## Conclusion

The synchronization strategy is:
- **Safe:** Prevents all race conditions
- **Efficient:** No busy-waiting, minimal lock holding
- **Correct:** Guarantees proper wake times
- **Simple:** Easy to understand and maintain
- **Scalable:** Works with many sleeping threads

This design follows Pintos best practices and fulfills all project requirements.

