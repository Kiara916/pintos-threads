# pintos-threads
Extend pintos timer functionality to not use busy wait.


# Project 4: Alarm Clock (CSE134 - Fall 2025)

---

> 📅 **Due:** Friday Dec **04**$^{\text{th}}$, 2025 at 11:59 PM.
> 
> **Learning Objectives:**
> 1. Get comfortable extending Pintos.
> 2. Practice using synchronization primitives in Pintos.

---

## Requirements

In this assignment, you will make changes to extend the Pintos OS. Instructions on how to start with Pintos, the file structure, and the debugging methods available can be found at the official documentation:

[https://web.stanford.edu/class/cs140/projects/pintos/pintos.html](https://web.stanford.edu/class/cs140/projects/pintos/pintos.html)

### To Do

> 1. **Reimplement `timer_sleep()`**
> 
> The current working implementation defined in `devices/timer.c` uses **"busy waiting"**—it spins in a loop checking the current time and calling `thread_yield()` until enough time has passed. **You must reimplement it to avoid busy waiting.**
> 
> **Function Signature:** `void timer_sleep (int64_t ticks)`
> 
> Suspends execution of the calling thread until time has advanced by at least **x** timer ticks. Unless the system is otherwise idle, the thread need not wake up after exactly **x** ticks. Just put it on the ready queue after they have waited for the right amount of time.
> 
> * `timer_sleep()` is useful for threads that operate in real-time, e.g., for blinking the cursor once per second.
> * The argument is expressed in **timer ticks**, not milliseconds. There are `TIMER_FREQ` timer ticks per second, defined in `devices/timer.h` (default is 100). Do not change this value.
> * You **do not** need to modify the existing `timer_msleep()`, `timer_usleep()`, and `timer_nsleep()` functions.
> * If your delays seem incorrect, reread the explanation of the `-r` option to `pintos` (see section 1.1.4 Debugging versus Testing).

---

## Rubric

In keeping with the CSE134 Projects 1-3, we have removed the requirement to submit a Design Document. Hence, the rubric for this assignment will only include the code for the Alarm clock part.

| Category | Percentage |
| :--- | ---: |
| **Testing** | 100% |

### Testing

We will use the provided tests for the alarm clock functionality. The tests will account for **100% of the grade** of this assignment. You can run the tests by following the steps specified in the official repo.

* The official repo has additional requirements and tests that are **not applicable** to us. Only tests related to the timer and alarm clock requirement are applicable.
* **Note:** You will not receive any points if your code uses the provided busy wait solution. You ***must*** attempt to remove busy waiting from the logic to receive credit, regardless of the output from your makefile.

---

## Hints

The [pintos website](https://web.stanford.edu/~ouster/cgi-bin/cs140-spring20/pintos/pintos_2.html#SEC16) provides a number of tips and hints for this assignment.

One important resource is **Section 2.1.2**, which outlines the various files in each of the directories that you might want to interact with for this assignment.

### Synchronization

Proper synchronization is crucial for working with shared resources correctly and robustly.

The simplest solution for synchronization issues in an operating system is to **turn off interrupts**. With interrupts off, there's no timer interrupt, no task switching, no concurrency, and hence, no race conditions between tasks or between tasks and interrupts.

However, this destroys real-time response and the determinism of timing, so it **cannot be the go-to solution**. We should always use synchronization primitives (i.e., **semaphores, locks, and condition variables**).

The only class of problem best solved by disabling interrupts is coordinating data shared between a **kernel thread and an interrupt handler**. Because interrupt handlers can't sleep, they can't acquire locks (think about why this is!). This means that data shared between kernel threads and an interrupt handler must be protected within a kernel thread by turning off interrupts. You will probably want to turn off interrupts when you handle timer interrupts; but, **try to have them off for as little code as possible.**

There should be **no busy waiting** in your submission. A tight loop that calls `thread_yield()` is one form of busy waiting.
