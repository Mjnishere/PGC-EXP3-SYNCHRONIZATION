# Experiment 5: Synchronization Mechanisms in Parallel Programs

- **Course:** Parallel & Distributed / GPU Computing Laboratory
- **Target OS:** Ubuntu 22.04 LTS (Linux / VMware / WSL)
- **Compiler:** GCC (build-essential) with OpenMP (`libgomp`) & POSIX Threads (`pthread`)
- **Workload:** 1,000,000 Shared Counter Increments across 1, 2, 4, and 8 Threads

---

## 1. Objective

To implement and study common synchronization mechanisms used in shared-memory parallel programs:
1. **Unsynchronized (Race Condition) update**
2. **Critical sections (`#pragma omp critical`)**
3. **Atomic operations (`#pragma omp atomic`)**
4. **Barriers (`#pragma omp barrier`)**
5. **Semaphores / Locks (`sem_t`, `omp_lock_t`)**

The goal is to evaluate their correctness, execution time, and scalability as the thread count increases from 1 to 8.

---

## 2. Theory & Synchronization Concepts

### 2.1 Why Synchronization is Needed
In shared-memory parallel architectures, multiple concurrent threads read and write to the same memory addresses. When multiple threads execute a read-modify-write operation (such as `counter++`) concurrently without coordination:
1. Thread A and Thread B read the same initial value of `counter` simultaneously.
2. Both threads compute `value + 1` in their local registers.
3. Both write back the same value to memory.

As a result, one or more increments are silently lost. Synchronization primitives coordinate memory access to ensure consistent and correct execution regardless of thread scheduling.

### 2.2 Mechanism Overview

```mermaid
flowchart TD
    T0["Thread 0"] --> Sync["Synchronization Mechanism"]
    T1["Thread 1"] --> Sync
    T2["Thread 2"] --> Sync
    T3["Thread 3"] --> Sync
    Sync --> Res[("Shared Resource<br>(counter variable)")]

    subgraph Primitives["Evaluated Mechanisms"]
        M1["None (Race Condition)<br>• No locking overhead<br>• Fast but silent data loss"]
        M2["Critical Section<br>• Serializes entire block<br>• Mutual exclusion via lock"]
        M3["Atomic Operation<br>• Hardware CPU instructions<br>• Low overhead, fine-grained"]
        M4["Lock / Semaphore<br>• OS/runtime mutex tokens<br>• Explicit lock acquire/release"]
    end
```

- **Race Condition (`race.c`):** No synchronization is applied. Counter updates collide across threads, leading to non-deterministic, incorrect results.
- **Critical Section (`critical.c`):** Enforced via `#pragma omp critical`. Only one thread can enter the designated block at any moment. While guaranteed to be correct, all competing threads serialize on the lock, inducing contention.
- **Atomic Operation (`atomic.c`):** Enforced via `#pragma omp atomic`. Tells the compiler to emit indivisible, hardware-level CPU instructions (e.g., atomic fetch-and-add / compare-and-swap). This avoids OS mutex queuing overhead while preserving correctness.
- **Barrier (`barrier.c`):** Enforced via `#pragma omp barrier`. Acts as a phase barrier where every thread must arrive before any thread is allowed to continue past the barrier point.
- **POSIX Semaphore (`semaphore.c`):** Counter-based synchronization via `sem_wait()` and `sem_post()`. Configured as a binary semaphore (`sem_init(&sem, 0, 1)`) to ensure mutual exclusion across POSIX threads.
- **Comprehensive Benchmark (`compare_sync.c`):** Directly compares `None`, `Critical`, `Atomic`, and `Lock` (`omp_lock_t`) across 1, 2, 4, and 8 threads for a shared workload of 1,000,000 iterations.

---

## 3. Repository Structure

```text
Exp_5_Synchronization_Mechanisms/
├── race.c             # Program 1: Race Condition (Unsynchronized)
├── critical.c         # Program 2: Critical Section (#pragma omp critical)
├── atomic.c           # Program 3: Atomic Operation (#pragma omp atomic)
├── barrier.c          # Program 4: Barrier Synchronization (#pragma omp barrier)
├── semaphore.c        # Program 5: POSIX Semaphore (pthread + semaphore.h)
├── compare_sync.c     # Program 6: Multi-thread comparison benchmark
├── screenshot_pgc.png # Terminal execution screenshot from Ubuntu VM
└── README.md          # Experiment documentation and analysis
```

---

## 4. Compilation & Execution

### 4.1 Compilation Steps

Compile each program directly using GCC:

```bash
# 1. Compile Race Condition program
gcc race.c -fopenmp -o race

# 2. Compile Critical Section program
gcc critical.c -fopenmp -o critical

# 3. Compile Atomic Operation program
gcc atomic.c -fopenmp -o atomic

# 4. Compile Barrier program
gcc barrier.c -fopenmp -o barrier

# 5. Compile Semaphore program
gcc semaphore.c -pthread -o semaphore

# 6. Compile Comparison Benchmark
gcc compare_sync.c -fopenmp -o compare_sync
```

### 4.2 Running the Programs

To run the comparison benchmark:
```bash
./compare_sync
```

To run individual OpenMP programs with a specific thread count:
```bash
OMP_NUM_THREADS=4 ./race
OMP_NUM_THREADS=4 ./critical
OMP_NUM_THREADS=4 ./atomic
OMP_NUM_THREADS=4 ./barrier
./semaphore
```

### 4.3 Handbook Errata & Import Fixes

The original source code provided in the lab handbook for `compare_sync.c` contained minor omissions in header inclusions:
1. **Missing Standard Utility Headers:** The handbook snippet only included `<stdio.h>` and `<omp.h>`, omitting `<stdlib.h>`, `<string.h>`, and `<stdbool.h>`. Standard headers have been added across all source files for clean standards compliance (`stdlib.h` for `NULL`/process utilities, `string.h`, and `stdbool.h`).
2. **OpenMP Include & Compilation Guard:** When compiled without the `-fopenmp` compiler flag, `omp.h` can cause undefined reference errors during linking. An explicit preprocessor guard (`#ifndef _OPENMP`) was added to `compare_sync.c` to produce a clean, informative error message instructing the user to supply `-fopenmp`.

---

## 5. Experimental Results

The experiment was compiled and executed on an Ubuntu Linux environment (`user@master:~/pgc`).

### 5.1 Execution Output Screenshot

![Benchmark Execution on Ubuntu](Screenshot From 2026-10-22 08-48-08.png)

### 5.2 Recorded Benchmark Data (`compare_sync`)

Workload: **1,000,000 shared increments**

| Mechanism | Threads | Final Count | Correctness | Execution Time (s) |
| :--- | :---: | :---: | :---: | :---: |
| **None** | 1 | 1,000,000 | **Correct** | 0.0004 |
| **Critical** | 1 | 1,000,000 | **Correct** | 0.0092 |
| **Atomic** | 1 | 1,000,000 | **Correct** | 0.0055 |
| **Lock** | 1 | 1,000,000 | **Correct** | 0.0091 |
| *---* | *---* | *---* | *---* | *---* |
| **None** | 2 | 522,490 | <span style="color:red">**INCORRECT**</span> | 0.0009 |
| **Critical** | 2 | 1,000,000 | **Correct** | 0.0173 |
| **Atomic** | 2 | 1,000,000 | **Correct** | 0.0069 |
| **Lock** | 2 | 1,000,000 | **Correct** | 0.0275 |
| *---* | *---* | *---* | *---* | *---* |
| **None** | 4 | 559,781 | <span style="color:red">**INCORRECT**</span> | 0.0024 |
| **Critical** | 4 | 1,000,000 | **Correct** | 0.0179 |
| **Atomic** | 4 | 1,000,000 | **Correct** | 0.0071 |
| **Lock** | 4 | 1,000,000 | **Correct** | 0.0236 |
| *---* | *---* | *---* | *---* | *---* |
| **None** | 8 | 549,322 | <span style="color:red">**INCORRECT**</span> | 0.0010 |
| **Critical** | 8 | 1,000,000 | **Correct** | 0.0201 |
| **Atomic** | 8 | 1,000,000 | **Correct** | 0.0069 |
| **Lock** | 8 | 1,000,000 | **Correct** | 0.0264 |

---

## 6. Analysis & Discussion

1. **Correctness & Data Hazards:**
   - **None (No Synchronization):** Only produces the correct count with 1 thread. As soon as threads $\ge 2$, over 45% of updates are lost (final counts range between $522,490$ and $559,781$). This clearly demonstrates the severe impact of unsynchronized read-modify-write data races.
   - **Critical, Atomic, and Lock:** All three synchronization primitives successfully guarantee correctness ($1,000,000$ final count) across all thread configurations (1, 2, 4, and 8 threads).

2. **Performance Comparison:**
   - **Atomic Operations** represent the best performing synchronization mechanism, executing in **$0.0055\text{s} - 0.0071\text{s}$**. Because atomic instructions map directly to hardware CPU primitives (like `LOCK XADD` or compare-and-swap), they bypass heavy software mutex runtime overhead.
   - **Critical Sections** require **$0.0173\text{s} - 0.0201\text{s}$**, approximately **$2.5\times - 3\times$ slower** than Atomic.
   - **Coarse-Grained Locks (`omp_lock_t`)** require **$0.0236\text{s} - 0.0275\text{s}$**, approximately **$3.5\times - 4\times$ slower** than Atomic due to the overhead of explicit lock acquisition and release on every iteration.

3. **Scalability & Contention:**
   - In workloads where all threads contend for a single global shared variable, increasing thread count does not produce linear speedup because the operation is fundamentally serialized.
   - While `Critical` and `Lock` execution time increases significantly when transitioning from 1 to 2 threads (due to lock contention and thread switching), `Atomic` retains stable execution times around **$0.0069\text{s} - 0.0071\text{s}$** even at 8 threads.

---

## 7. Observation & Comparison Summary

| Mechanism | Correctness | Relative Performance | Scalability Characteristics |
| :--- | :---: | :---: | :--- |
| **None (Race Condition)** | <span style="color:red">**Incorrect**</span> | Fastest (No synchronization overhead) | **Poor:** Silent data loss worsens as thread concurrency increases. |
| **Critical Section** | **Correct** | Slow (Serializes thread access) | **Poor to Moderate:** Contention bottlenecks increase with thread count. |
| **Atomic Operation** | **Correct** | **Fastest Correct** (Hardware-level instruction) | **Good:** Minimal overhead and stable throughput across thread counts. |
| **Lock (`omp_lock_t`)** | **Correct** | Slow (Explicit lock acquisition overhead) | **Poor to Moderate:** Heavy contention and lock management overhead. |
| **Barrier** | *Phase Sync Only* | Dependent on slowest thread arrival | Synchronizes parallel phases; introduces idle wait times. |
| **Semaphore** | **Correct** | Moderate (OS system call overhead) | General counting/mutual exclusion primitive. |

---

## 8. Conclusion

- **Atomic operations (`#pragma omp atomic`)** provide the superior balance of safety, high performance, and scalability for updating simple shared variables because they leverage hardware-assisted instructions rather than OS-level mutex locks.
- **Critical sections (`#pragma omp critical`) and Locks (`omp_lock_t`)** effectively preserve correctness but introduce substantial lock contention overhead for fine-grained operations.
- **Barriers (`#pragma omp barrier`)** provide phase coordination across parallel threads without directly guarding data structures.
- Unsynchronized multi-threaded memory access is fast but completely unreliable due to data races, reinforcing that choosing the right synchronization mechanism is a critical trade-off between correctness guarantees, execution latency, and scalability.
