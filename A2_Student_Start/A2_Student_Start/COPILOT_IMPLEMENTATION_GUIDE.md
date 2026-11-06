# Copilot Implementation Guide - SYSC 4001 Assignment 2

This file is a secondary README for GitHub Copilot (or any coding assistant) working in VS Code.
It explains exactly what to change in this folder to complete Assignment 2, "Synchronization
Primitives and Bounded Work Queue". Follow the steps in order. The original `README.md` and the
assignment handout remain the specification; this guide does not override them.

---

## 0. Ground rules (read before editing anything)

1. **Edit only `src/student_sync.c`.** Do not modify `include/sync.h`, `src/test_driver.c`,
   `src/benchmark.c`, `Makefile`, `test_harness.py`, or anything under `tests/`.
2. **Keep every original comment in `src/student_sync.c`.** The starter file's comment blocks
   (the top "Complete one section at a time..." block, the "Spinlock", "Semaphore" and
   "Work queue" section comments, and the `// ~~ Note this include...` comment) must stay
   exactly as they are. Add new comments next to them; never delete or reword them.
3. **Do not change any function signature, struct, or the CSV output format.**
4. **The build must stay warning-free** under the supplied flags:
   `-std=c11 -D_XOPEN_SOURCE=700 -Wall -Wextra -Wpedantic -O2 -pthread`.
5. **Return 0 on success and an errno-style code on failure** (`EINVAL`, `ENOMEM`, `EOVERFLOW`,
   or the code a pthread function returned). NULL object pointers return `EINVAL`.
6. **No sleep-based polling** (`sleep`, `usleep`, `nanosleep`) anywhere in the library.
7. **Spinlock:** only the C11 `atomic_flag` in the struct. No `pthread_mutex_t`, no
   `StudentSemaphore`, no `sem_t`. `sched_yield()` is allowed inside the spin loop.
8. **Semaphore:** only the struct's `pthread_mutex_t`, `pthread_cond_t` and `count`. No `sem_t`
   and no POSIX `sem_*` functions. Waiting must use `pthread_cond_wait()` in a `while` loop.
9. **Work queue:** built from the student semaphore (`slots`, `items_available`) and the student
   spinlock (`guard`). Never wait on a semaphore while holding `guard`.

---

## 1. Names borrowed from the lecture slides

The struct field names are fixed by `sync.h`, so lecture names are used for local variables and
comments only. This is the mapping between the Chapter 6/7 slides and this assignment:

| Lecture (Ch. 6 / Ch. 7)                     | This assignment                          |
|---------------------------------------------|------------------------------------------|
| `test_and_set(&lock)` / `lock = false`      | `atomic_flag_test_and_set_explicit` / `atomic_flag_clear_explicit` |
| semaphore `S`, `wait(S)`, `signal(S)`       | `semaphore->count`, `student_sem_wait`, `student_sem_post` |
| `buffer[BUFFER_SIZE]`                       | `queue->items`, `queue->capacity`        |
| `in`, `out`                                 | `queue->tail`, `queue->head` (local copies named `in`, `out`) |
| `empty` (initialized to n)                  | `queue->slots` (initialized to capacity) |
| `full` (initialized to 0)                   | `queue->items_available` (initialized to 0) |
| `mutex` (initialized to 1)                  | `queue->guard` (spinlock)                |
| `next_consumed`                             | local in `work_queue_take`               |

---

## 2. Step-by-step implementation

Work in the order below. After each step, build and run the listed public test.

### Step 1 - Includes

Keep the existing includes (including the `<limits.h>` line and its comment). Add one include
directly below it:

```c
#include <sched.h>
```

`<limits.h>` is needed for `UINT_MAX` (semaphore overflow check and queue capacity limit).
`<sched.h>` is needed for `sched_yield()` in the spinlock.

### Step 2 - Spinlock (15 marks)

Requirements:
- `student_spin_init`: `NULL` -> `EINVAL`. Clear the flag (unlocked), set `initialized = true`.
- `student_spin_lock`: `NULL` or not initialized -> `EINVAL`. Loop on
  `atomic_flag_test_and_set_explicit(&lock->flag, memory_order_acquire)` until it returns
  `false`; call `sched_yield()` inside the loop.
- `student_spin_unlock`: `NULL` or not initialized -> `EINVAL`. Call
  `atomic_flag_clear_explicit(&lock->flag, memory_order_release)`.

Why acquire/release: acquire on lock stops critical-section reads and writes from moving above
the lock; release on unlock makes all critical-section writes visible before the flag is seen
clear. This is the memory-barrier example from the Chapter 6 slides.

Test: `python3 test_harness.py --case spin_basic`

### Step 3 - Blocking counting semaphore (25 marks)

Requirements:
- `student_sem_init`: `NULL` -> `EINVAL`. `pthread_mutex_init`, then `pthread_cond_init`
  (if the second fails, destroy the mutex). Store `initial_count` in `count`; set
  `initialized = true`. Return any pthread error code.
- `student_sem_wait`: `NULL` or not initialized -> `EINVAL`. Lock the mutex.
  `while (count == 0) pthread_cond_wait(...)` - a **while loop, not an if** (spurious wakeups,
  and another thread may take the permit first). Decrement `count`. Unlock. If
  `pthread_cond_wait` fails, unlock the mutex before returning the error.
- `student_sem_post`: `NULL` or not initialized -> `EINVAL`. Lock the mutex. If
  `count == UINT_MAX`, unlock and return `EOVERFLOW` without changing `count`. Otherwise
  increment `count`, `pthread_cond_signal`, unlock.
- `student_sem_destroy`: `NULL` or not initialized -> `EINVAL`. Destroy the mutex (return its
  error, e.g. `EBUSY`, if it fails), then the condition variable. Set `initialized = false`.

Tests: `python3 test_harness.py --case semaphore_blocking` then `--case semaphore_counting`

### Step 4 - Bounded concurrent work queue (30 marks)

Requirements:
- `work_queue_init`: `NULL`, `capacity == 0`, or `capacity > UINT_MAX` -> `EINVAL` (check these
  before touching the struct). `malloc` exactly `capacity` ints (`ENOMEM` on failure). Set
  `head = tail = count = 0`. Init `slots` to `capacity`, `items_available` to `0`, and `guard`.
  If any step fails, undo the earlier steps and return the error. Set `initialized = true`.
- `work_queue_submit` (producer):
  1. `student_sem_wait(&queue->slots)` - lecture `wait(empty)`; blocks while full.
  2. `student_spin_lock(&queue->guard)` - lecture `wait(mutex)`.
  3. `items[tail] = job; tail = (tail + 1) % capacity; count++;`
  4. `student_spin_unlock(&queue->guard)` - lecture `signal(mutex)`.
  5. `student_sem_post(&queue->items_available)` - lecture `signal(full)`.
- `work_queue_take` (consumer): `NULL` queue **or `NULL` job** -> `EINVAL`.
  1. `student_sem_wait(&queue->items_available)` - lecture `wait(full)`; blocks while empty.
  2. Lock `guard`.
  3. `next_consumed = items[head]; head = (head + 1) % capacity; count--;`
  4. Unlock `guard`, write `*job = next_consumed`.
  5. `student_sem_post(&queue->slots)` - lecture `signal(empty)`.
- `work_queue_destroy`: `NULL` or not initialized -> `EINVAL`. Destroy both semaphores, free
  `items`, reset fields, set `initialized = false`. Return the first error seen.

Order matters: always take the semaphore **before** the guard, and post the opposite semaphore
**after** releasing the guard. Waiting while holding the guard deadlocks a capacity-one queue.

Tests: `--case queue_fifo`, then `--case queue_basic` and `--case queue_capacity_one`

---

## 3. Final `src/student_sync.c` (reference target)

When finished, `src/student_sync.c` should be equivalent to the code below. It keeps every
original starter comment. This exact code was built with the supplied Makefile (no warnings),
passed 6/6 public tests, passed the driver's `invalid_arguments` and `semaphore_overflow` modes,
and showed no data races under ThreadSanitizer.

```c
#include "sync.h"
#include <stdlib.h>
#include <errno.h>
#include <limits.h> // ~~ Note this include... Why might we need this? 
#include <sched.h>

/*
 * Complete one section at a time. It is fine to leave later functions
 * returning ENOSYS while you work on the spinlock or semaphore.
 *
 * Suggested public-test order:
 *   spin_basic
 *   semaphore_blocking, then semaphore_counting
 *   queue_fifo, then queue_basic and queue_capacity_one
 *
 * You may add private helpers or source files. Do not change sync.h.
 */

/*
 * Spinlock
 *
 * Decide which atomic operation claims the flag and which memory ordering is
 * needed on lock and unlock. Check the pointer and initialized flag before
 * touching the atomic flag.
 */

int student_spin_init(StudentSpinlock *lock)
{
    if (lock == NULL) {
        return EINVAL;
    }

    /* Start in the unlocked state: the flag is clear ("lock = false"). */
    atomic_flag_clear_explicit(&lock->flag, memory_order_relaxed);
    lock->initialized = true;
    return 0;
}

int student_spin_lock(StudentSpinlock *lock)
{
    if (lock == NULL || !lock->initialized) {
        return EINVAL;
    }

    /*
     * Lecture: while (test_and_set(&lock)) ; // do nothing
     * test_and_set returns the old value. If it was already set, another
     * thread holds the lock, so keep polling. Acquire ordering stops reads
     * and writes in the critical section from moving above this point.
     */
    while (atomic_flag_test_and_set_explicit(&lock->flag, memory_order_acquire)) {
        sched_yield();
    }
    return 0;
}

int student_spin_unlock(StudentSpinlock *lock)
{
    if (lock == NULL || !lock->initialized) {
        return EINVAL;
    }

    /*
     * Lecture: lock = false;
     * Release ordering makes every write in the critical section visible
     * before another thread can see the flag clear.
     */
    atomic_flag_clear_explicit(&lock->flag, memory_order_release);
    return 0;
}

/*
 * Semaphore
 *
 * Keep every access to count protected by the pthread_mutex semaphore->mutex. 
 */

int student_sem_init(StudentSemaphore *semaphore, unsigned int initial_count)
{
    if (semaphore == NULL) {
        return EINVAL;
    }

    int status = pthread_mutex_init(&semaphore->mutex, NULL);
    if (status != 0) {
        return status;
    }

    status = pthread_cond_init(&semaphore->condition, NULL);
    if (status != 0) {
        pthread_mutex_destroy(&semaphore->mutex);
        return status;
    }

    semaphore->count = initial_count;
    semaphore->initialized = true;
    return 0;
}

int student_sem_wait(StudentSemaphore *semaphore)
{
    if (semaphore == NULL || !semaphore->initialized) {
        return EINVAL;
    }

    int status = pthread_mutex_lock(&semaphore->mutex);
    if (status != 0) {
        return status;
    }

    /*
     * Lecture: while (S <= 0) ; S--;
     * Instead of busy waiting, sleep on the condition variable. The while
     * loop rechecks the predicate after every wakeup, because a wakeup can be
     * spurious or another thread may have taken the permit first.
     */
    while (semaphore->count == 0) {
        status = pthread_cond_wait(&semaphore->condition, &semaphore->mutex);
        if (status != 0) {
            pthread_mutex_unlock(&semaphore->mutex);
            return status;
        }
    }

    semaphore->count--;
    return pthread_mutex_unlock(&semaphore->mutex);
}

int student_sem_post(StudentSemaphore *semaphore)
{
    if (semaphore == NULL || !semaphore->initialized) {
        return EINVAL;
    }

    int status = pthread_mutex_lock(&semaphore->mutex);
    if (status != 0) {
        return status;
    }

    /* count is unsigned, so adding to UINT_MAX would wrap to zero. */
    if (semaphore->count == UINT_MAX) {
        pthread_mutex_unlock(&semaphore->mutex);
        return EOVERFLOW;
    }

    /* Lecture: S++; then wake one waiting thread. */
    semaphore->count++;
    status = pthread_cond_signal(&semaphore->condition);

    int unlock_status = pthread_mutex_unlock(&semaphore->mutex);
    return status != 0 ? status : unlock_status;
}

int student_sem_destroy(StudentSemaphore *semaphore)
{
    if (semaphore == NULL || !semaphore->initialized) {
        return EINVAL;
    }

    int status = pthread_mutex_destroy(&semaphore->mutex);
    if (status != 0) {
        return status;
    }

    status = pthread_cond_destroy(&semaphore->condition);
    semaphore->initialized = false;
    return status;
}

/*
 * Work queue
 *
 * Check the WorkQueue definition in the header file for details on its structure and 
 * synchronization requirements.
 *
 * Lecture bounded-buffer mapping:
 *   buffer          -> queue->items
 *   BUFFER_SIZE     -> queue->capacity
 *   in / out        -> queue->tail / queue->head
 *   empty (n)       -> queue->slots (starts at capacity)
 *   full (0)        -> queue->items_available (starts at 0)
 *   mutex           -> queue->guard (spinlock)
 */
int work_queue_init(WorkQueue *queue, size_t capacity)
{
    if (queue == NULL || capacity == 0 || capacity > UINT_MAX) {
        return EINVAL;
    }

    queue->items = malloc(capacity * sizeof(*queue->items));
    if (queue->items == NULL) {
        return ENOMEM;
    }

    queue->capacity = capacity;
    queue->head = 0;
    queue->tail = 0;
    queue->count = 0;

    int status = student_sem_init(&queue->slots, (unsigned int)capacity);
    if (status != 0) {
        free(queue->items);
        queue->items = NULL;
        return status;
    }

    status = student_sem_init(&queue->items_available, 0);
    if (status != 0) {
        student_sem_destroy(&queue->slots);
        free(queue->items);
        queue->items = NULL;
        return status;
    }

    status = student_spin_init(&queue->guard);
    if (status != 0) {
        student_sem_destroy(&queue->items_available);
        student_sem_destroy(&queue->slots);
        free(queue->items);
        queue->items = NULL;
        return status;
    }

    queue->initialized = true;
    return 0;
}

int work_queue_submit(WorkQueue *queue, int job)
{
    if (queue == NULL || !queue->initialized) {
        return EINVAL;
    }

    /* Lecture: wait(empty). Blocks while full, without holding guard. */
    int status = student_sem_wait(&queue->slots);
    if (status != 0) {
        return status;
    }

    /* Lecture: wait(mutex). */
    status = student_spin_lock(&queue->guard);
    if (status != 0) {
        student_sem_post(&queue->slots);
        return status;
    }

    /* Lecture: buffer[in] = next_produced; in = (in + 1) % BUFFER_SIZE; */
    size_t in = queue->tail;
    queue->items[in] = job;
    queue->tail = (in + 1) % queue->capacity;
    queue->count++;

    /* Lecture: signal(mutex); signal(full); */
    status = student_spin_unlock(&queue->guard);
    if (status != 0) {
        return status;
    }
    return student_sem_post(&queue->items_available);
}

int work_queue_take(WorkQueue *queue, int *job)
{
    if (queue == NULL || job == NULL || !queue->initialized) {
        return EINVAL;
    }

    /* Lecture: wait(full). Blocks while empty, without holding guard. */
    int status = student_sem_wait(&queue->items_available);
    if (status != 0) {
        return status;
    }

    /* Lecture: wait(mutex). */
    status = student_spin_lock(&queue->guard);
    if (status != 0) {
        student_sem_post(&queue->items_available);
        return status;
    }

    /* Lecture: next_consumed = buffer[out]; out = (out + 1) % BUFFER_SIZE; */
    size_t out = queue->head;
    int next_consumed = queue->items[out];
    queue->head = (out + 1) % queue->capacity;
    queue->count--;

    /* Lecture: signal(mutex); signal(empty); */
    status = student_spin_unlock(&queue->guard);
    if (status != 0) {
        return status;
    }

    *job = next_consumed;
    return student_sem_post(&queue->slots);
}

int work_queue_destroy(WorkQueue *queue)
{
    if (queue == NULL || !queue->initialized) {
        return EINVAL;
    }

    int status = student_sem_destroy(&queue->slots);
    int other = student_sem_destroy(&queue->items_available);
    if (status == 0) {
        status = other;
    }

    free(queue->items);
    queue->items = NULL;
    queue->capacity = 0;
    queue->head = 0;
    queue->tail = 0;
    queue->count = 0;
    queue->guard.initialized = false;
    queue->initialized = false;
    return status;
}
```

---

## 4. Verify the implementation

Run these from the `A2_Student_Start/` folder on Linux or WSL:

```bash
make clean && make
```

```bash
python3 test_harness.py --no-compile
```

Expected: no compiler warnings, and `Result: 6/6 test cases passed`.

The driver has two extra modes that the public cases do not cover but hidden tests may. Check
both:

```bash
./sync_runner --mode invalid_arguments --output out_invalid && cat out_invalid/results.csv
```

```bash
./sync_runner --mode semaphore_overflow --output out_overflow && cat out_overflow/results.csv
```

Expected data rows:

```text
invalid_arguments,7,7,0,0,0,0,0
semaphore_overflow,1,1,0,0,0,0,0
```

Optional stress check (larger, more contended than the public cases):

```bash
./sync_runner --mode queue --producers 8 --consumers 3 --jobs 5000 --capacity 1 --output out_stress && cat out_stress/results.csv
```

Expected: `queue,5000,5000,0,0,0,12502500,12502500`

Delete the `out_*` folders afterwards; they are not part of the submission.

---

## 5. Self-check against the marking rubric

| Rubric item (marks)            | What to confirm in the code                                        |
|--------------------------------|--------------------------------------------------------------------|
| Build and public interface (5) | Only `src/student_sync.c` changed; clean, warning-free build.     |
| Atomic spinlock (15)           | Test-and-set with `memory_order_acquire`; clear with `memory_order_release`; init clears the flag; NULL/uninitialized -> `EINVAL`; reusable. |
| Counting semaphore (25)        | Every `count` access under `mutex`; `while` predicate loop around `pthread_cond_wait`; signal on post; `EOVERFLOW` at `UINT_MAX` without changing `count`; mutex unlocked on every error path; destroy cleans up. |
| Bounded work queue (30)        | Circular buffer with `% capacity`; wait `slots`/`items_available` before `guard`; post the opposite semaphore after releasing `guard`; capacity 1 works; `take(NULL, NULL)` and `init(&q, 0)` -> `EINVAL`. |

---

## 6. Part 2 - Synchronization analysis (`report.pdf`, 25 marks)

Copilot must **not invent timing numbers**. The student runs the benchmark on their own machine
and fills in the table. Under two pages, no introduction or conclusion.

### 6.1 Collect the data

Machine details:

```bash
lscpu | grep "Model name"
```

```bash
nproc
```

```bash
cat /etc/os-release | head -2; uname -r
```

Run each configuration at least five times:

```bash
for p in spin semaphore; do for t in 1 8; do for r in 1 2 3 4 5; do ./sync_benchmark --primitive $p --threads $t --iterations 10000 | tail -1; done; done; done
```

Each output line is `primitive,threads,iterations_per_thread,elapsed_ms,expected,observed,errors`.
Confirm `expected == observed` and `errors == 0`, then take the median `elapsed_ms` of the five
runs for each configuration. The 8-thread semaphore run can take tens of seconds on some machines.

### 6.2 Report template

```text
Part 2A - Low and High Contention

CPU model: <from lscpu>
Logical CPUs: <from nproc>
OS: <e.g. Ubuntu 22.04 on WSL2>

Commands:
./sync_benchmark --primitive spin --threads 1 --iterations 10000
./sync_benchmark --primitive spin --threads 8 --iterations 10000
./sync_benchmark --primitive semaphore --threads 1 --iterations 10000
./sync_benchmark --primitive semaphore --threads 8 --iterations 10000

| Primitive | Threads | Iterations per thread | Runs | Median elapsed_ms |
|-----------|---------|-----------------------|------|-------------------|
| Spinlock  | 1       | 10,000                | 5    | <median>          |
| Spinlock  | 8       | 10,000                | 5    | <median>          |
| Semaphore | 1       | 10,000                | 5    | <median>          |
| Semaphore | 8       | 10,000                | 5    | <median>          |
```

### 6.3 Points the Part 2A explanation should cover (max 200 words)

Write in your own words; check each point against your actual numbers.

- **1 thread (no contention):** the spinlock is one atomic test-and-set plus one clear, all in
  user space. The semaphore does a mutex lock, a count check, a mutex unlock, and a condition
  signal per operation, so it costs more even when nobody waits.
- **8 threads (high contention):** both get slower per operation because the lock's cache line
  bounces between cores. The spinlock usually stays relatively cheap because the critical
  section is a single increment, so the holder releases it almost immediately, and
  `sched_yield()` hands the CPU over instead of burning a full time slice.
- **Semaphore under contention:** waiters sleep in `pthread_cond_wait`. Each post must wake a
  sleeping thread through the kernel (futex syscall plus context switch), and a woken thread
  often finds the permit already taken and goes back to sleep (the predicate loop). This is
  why the 8-thread semaphore can be orders of magnitude slower than the 8-thread spinlock.
- **Expectation:** say whether this matched what you predicted. Spinlocks win for very short
  critical sections; blocking primitives win when the critical section is long or when there
  are more threads than CPUs, because they do not waste CPU time spinning.
- If your thread count (8) exceeds `nproc`, mention that oversubscription hurts spinning more.

### 6.4 Points the Part 2B explanation should cover (max 200 words)

- **Fairness guaranteed: none.** The spinlock gives no FIFO order and no bounded waiting:
  whichever thread's test-and-set wins gets the lock, so a thread can starve in theory.
  The semaphore uses `pthread_cond_signal`, which wakes *some* waiter, not the oldest, and a
  newly arriving thread can take the permit before the woken thread reacquires the mutex
  (barging). The work queue inherits this: jobs leave in FIFO order, but which producer or
  worker gets to go next is not ordered.
- What is guaranteed: mutual exclusion, progress (no deadlock when used correctly), and
  exactly-once delivery of each accepted job.
- **Observer effect:** `printf` takes an internal stdio lock and may make a system call, and a
  debugger breakpoint pauses one thread while others keep running. Both change timing and
  scheduling, so they can hide a race (the bad interleaving no longer happens) or create a
  new one (Heisenbug). Results seen while observing may not reflect normal runs.

---

## 7. Optional `README.txt` for the marker

```text
SYSC 4001 A2 - implementation notes
- Spinlock: atomic_flag test-and-set (acquire) / clear (release), sched_yield while spinning.
- Semaphore: mutex-protected count, pthread_cond_wait in a while loop, signal on post,
  EOVERFLOW at UINT_MAX.
- Work queue: circular buffer; slots/items_available semaphores taken before the guard
  spinlock and the opposite semaphore posted after the guard is released.
- All public tests pass; clean build under the supplied Makefile.
```

---

## 8. Submission checklist

Create `SYSC4001_A2_<student-number>.zip` containing only:

- `Makefile`
- `include/`
- `src/`
- `report.pdf`
- `README.txt` (optional)

Do not include `sync_runner`, `sync_benchmark`, `out_*` folders, or this guide. Before
submitting, extract the ZIP into a clean folder, run `make clean && make` there, then from the
original `A2_Student_Start/` folder:

```bash
python3 test_harness.py --student-dir /path/to/extracted/submission --tests "$PWD/tests/public" --no-compile
```

---

## 9. Academic integrity note

The handout says you must understand and be able to explain every part of the code you submit,
and to follow the course's AI-use policy. Use this guide to compare against and learn from, not
as a replacement for your own work.
