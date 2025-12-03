# Building and Testing the Pintos Threads Project

## Prerequisites

Before building and testing, ensure you have:
- GCC and build tools installed
- Perl installed (for test scripts)
- A QEMU or Bochs emulator for running Pintos

## Building the Project

### Step 1: Navigate to the threads directory
```bash
cd pintos-threads/src/threads
```

### Step 2: Build the threads project
```bash
make
```

This will compile the kernel with your modifications to `devices/timer.c`.

## Running Tests

### Step 3: Navigate to the tests directory
```bash
cd ../tests
```

### Step 4: Run the alarm clock tests

To run a specific alarm test:
```bash
make tests/threads/alarm-single.result
```

To run all alarm tests:
```bash
make -f Make.tests tests/threads/alarm-single.result
make -f Make.tests tests/threads/alarm-zero.result
make -f Make.tests tests/threads/alarm-negative.result
make -f Make.tests tests/threads/alarm-multiple.result
make -f Make.tests tests/threads/alarm-priority.result
make -f Make.tests tests/threads/alarm-simultaneous.result
make -f Make.tests tests/threads/alarm-wait.result
```

Or run all tests with grading:
```bash
make -f Make.tests tests VERBOSE=1
```

## Understanding Test Results

### Successful Test Output
When a test passes, you should see output like:
```
PASS tests/threads/alarm-single
```

### Common Issues and Fixes

1. **Test Hangs:**
   - If a test hangs, the timer interrupt handler may not be working correctly
   - Verify that threads are being properly woken from the sleeping list

2. **Segmentation Fault:**
   - Check that memory allocation (malloc) and deallocation (free) are correct
   - Ensure the sleeping_thread structure is properly initialized

3. **Incorrect Wake Times:**
   - Verify that `wake_tick` is calculated correctly: `timer_ticks() + ticks`
   - Check that the timer interrupt is comparing current `ticks` with `wake_tick`

## Detailed Test Descriptions

### alarm-single
- Basic test: sleeps for a single timer tick
- Should complete quickly without hanging

### alarm-zero
- Tests with zero ticks
- Should return immediately without sleeping

### alarm-negative
- Tests with negative tick values
- Should handle gracefully (e.g., return immediately)

### alarm-multiple
- Multiple threads sleeping at different times
- Tests concurrent sleep functionality

### alarm-priority
- Tests that sleeping threads don't interfere with priority scheduling

### alarm-simultaneous
- Multiple threads sleeping at the same time
- All should wake up together

### alarm-wait
- Tests that sleeping threads properly yield CPU to other threads
- Verifies no busy-waiting is occurring

## Debugging Tips

### Enable Verbose Output
```bash
make -f Make.tests VERBOSE=1
```

### Check Timer Statistics
In your test, you can add:
```c
printf ("Timer: %"PRId64" ticks\n", timer_ticks ());
```

### Use Pintos Debugging Options
- `-s`: Set breakpoints
- `-d` followed by flags: Enable specific debug output
- `-r`: Re-seed random number generator for reproducibility

Example:
```bash
pintos -r 12345 tests/threads/alarm-single
```

## Key Implementation Details to Verify

1. **No Busy-Waiting:** 
   - The thread_yield() loop is completely removed
   - Threads sleep on a semaphore, not in a loop

2. **Proper Synchronization:**
   - The sleeping_list is protected by a lock
   - Each thread has its own semaphore

3. **Correct Wake Times:**
   - Threads wake when current_ticks >= wake_tick
   - Not before, not after (within one tick)

4. **Memory Management:**
   - Sleeping thread structure is freed after waking
   - No memory leaks

## Expected Behavior

### Before Modification (Busy-Wait)
```
Thread calls timer_sleep(100)
  → Loop: while (elapsed < 100) thread_yield();
  → CPU wasted spinning
  → Thread keeps running/yielding
```

### After Modification (Semaphore-Based)
```
Thread calls timer_sleep(100)
  → Calculate wake_tick = current + 100
  → Add to sleeping_list
  → sema_down() blocks thread
  → CPU is free for other threads
  → Timer interrupt wakes thread at wake_tick
  → Thread continues
```

## Troubleshooting Compilation Errors

If you get compilation errors:

1. **Missing stdlib.h:**
   - Verify that `#include <stdlib.h>` is present in timer.c

2. **Undefined reference to malloc/free:**
   - Check that the correct headers are included
   - Ensure the project is linked correctly

3. **Undefined reference to semaphore functions:**
   - Verify `#include "threads/synch.h"` is present
   - Check that synch.c is compiled and linked

## Running Tests with Make Grade Script

If available, use the make-grade script:
```bash
cd tests
perl make-grade
```

This will run all applicable tests and generate a report.

