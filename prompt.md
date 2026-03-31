

# 🚀 18-Day Modern C++ Multithreading Mastery Plan

---

# 🔹 PHASE 1 — Foundations (Days 1–4)

Goal: Understand *what concurrency really is* and how C++ models it.

---

## ✅ Day 1 — Concurrency Fundamentals

### Topics

* Process vs Thread
* Concurrency vs Parallelism
* Context switching
* CPU cores, hyperthreading
* Data races
* Undefined behavior

### Concepts to Understand

* Why race conditions happen
* What makes multithreading hard
* Determinism vs non-determinism

### Practical

* Create a basic `std::thread`
* Join vs detach
* Passing parameters
* Member function threads

### Example

```cpp
#include <iostream>
#include <thread>

void worker(int id) {
    std::cout << "Thread " << id << "\n";
}

int main() {
    std::thread t1(worker, 1);
    t1.join();
}
```

### Exercise

* Launch 5 threads printing IDs
* Observe interleaving behavior

---

## ✅ Day 2 — Thread Lifecycle & RAII

### Topics

* `joinable()`
* Destructor behavior
* Thread ownership
* Move semantics with threads

### Learn

* Why forgetting `join()` causes `std::terminate`
* How to create RAII wrapper for threads

### Build

Create your own `ThreadGuard` class.

---

## ✅ Day 3 — Shared Data & Mutex

### Topics

* Critical sections
* `std::mutex`
* `std::lock_guard`
* `std::unique_lock`

### Understand

* Why race condition occurs
* Mutual exclusion
* Deadlock basics

### Example

```cpp
std::mutex m;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(m);
    ++counter;
}
```

### Exercise

* 10 threads increment counter 1M times
* Compare with and without mutex

---

## ✅ Day 4 — Deadlocks & Locking Strategies

### Topics

* Deadlock conditions
* Lock ordering
* `std::scoped_lock`
* `std::try_lock`

### Practical

Create intentional deadlock → Fix using:

* Consistent lock ordering
* `std::scoped_lock`

---

# 🔹 PHASE 2 — Synchronization Primitives (Days 5–8)

Goal: Learn signaling and coordination.

---

## ✅ Day 5 — Condition Variables

### Topics

* `std::condition_variable`
* Producer–Consumer problem
* Spurious wakeups
* Predicate-based waiting

### Build

Thread-safe queue using:

* `mutex`
* `condition_variable`

This is a core interview problem.

---

## ✅ Day 6 — Futures & Async

### Topics

* `std::future`
* `std::promise`
* `std::async`
* `std::packaged_task`

### Understand

* Synchronization via future
* Lazy vs eager execution
* Launch policies

### Exercise

Implement parallel sum using `std::async`.

---

## ✅ Day 7 — Thread Pools (Manual Implementation)

### Topics

* Task queue
* Worker threads
* Graceful shutdown
* Work stealing (conceptual)

### Build

Your own minimal thread pool:

```cpp
class ThreadPool {
    // vector<thread>
    // queue<function<void()>>
    // mutex + condition_variable
};
```

This is **advanced interview level**.

---

## ✅ Day 8 — C++20 Additions

### Topics

* `std::jthread`
* Stop tokens
* `std::latch`
* `std::barrier`
* `std::counting_semaphore`

### Practice

Implement:

* Parallel task using `std::barrier`
* Cooperative cancellation with `stop_token`

---

# 🔹 PHASE 3 — Atomics & Memory Model (Days 9–12)

This is where most developers fail interviews.

---

## ✅ Day 9 — std::atomic Basics

### Topics

* Atomic types
* `fetch_add`
* `compare_exchange`
* lock-free vs wait-free

### Exercise

Replace mutex counter with atomic counter.

---

## ✅ Day 10 — C++ Memory Model

### Topics

* Sequential consistency
* Acquire / Release
* Relaxed ordering
* Happens-before relationship

### Learn Deeply

* Why CPU reordering happens
* Compiler reordering
* Visibility guarantees

### Practice

Implement:

* Producer sets flag
* Consumer waits using atomic flag

---

## ✅ Day 11 — Lock-Free Programming

### Topics

* CAS loop
* ABA problem
* Memory ordering correctness
* False sharing

### Build

Lock-free stack using atomic pointer.

---

## ✅ Day 12 — Performance & False Sharing

### Topics

* Cache line size
* Padding
* `std::hardware_destructive_interference_size`
* Profiling multithreaded code

### Experiment

Measure:

* Performance with shared variable
* Performance with padded struct

---

# 🔹 PHASE 4 — High-Level Concurrency Design (Days 13–16)

---

## ✅ Day 13 — Parallel STL

### Topics

* Execution policies
* `std::execution::par`
* `std::execution::par_unseq`

### Practice

Parallel transform / reduce

---

## ✅ Day 14 — Designing Thread-Safe Classes

### Topics

* Const correctness in multithreading
* Thread-safe vs thread-compatible
* Immutable design
* Copy vs move safety

### Design

Thread-safe LRU cache.

---

## ✅ Day 15 — Real-World Patterns

### Topics

* Active Object
* Reactor
* Leader–Follower
* Double buffering

### Implement

Active Object pattern in C++.

---

## ✅ Day 16 — Debugging Multithreaded Code

### Topics

* Thread Sanitizer (TSAN)
* Address Sanitizer
* Valgrind Helgrind
* Logging strategies

### Practice

Create intentional race → Detect via TSAN.

---

# 🔹 PHASE 5 — Advanced Mastery (Days 17–18)

---

## ✅ Day 17 — Coroutines (C++20)

### Topics

* Coroutine basics
* Awaitable types
* Async IO model
* Coroutines vs threads

Understand:

* Cooperative vs preemptive multitasking

---

## ✅ Day 18 — System-Level Thinking

### Topics

* NUMA awareness
* Work stealing algorithms
* Thread affinity
* When NOT to use threads
* Actor model
* Message passing vs shared memory

---

# 🧠 Daily Structure (Follow This)

Each day:

1. 1–2 hours theory
2. 2 hours coding
3. 1 debugging exercise
4. Write summary notes
5. Re-implement from scratch next day

---

# 📚 Recommended References

* Anthony Williams — *C++ Concurrency in Action*
* cppreference.com
* Herb Sutter talks (YouTube)
* CppCon multithreading talks

---

# 🎯 Interview Readiness Checklist

By the end, you must confidently answer:

* Explain C++ memory model
* What is happens-before?
* Difference between mutex and atomic?
* How does condition_variable avoid busy wait?
* What is ABA problem?
* What is false sharing?
* How to design thread pool?
* What is lock-free programming?
* What is difference between acquire and release?

---

# 🏆 Final Project (Mandatory)

Build:

### 🔥 High-Performance Task Scheduler

Requirements:

* Thread pool
* Work queue
* Futures support
* Stop token support
* Lock-free queue (optional)
* Benchmark performance

---
