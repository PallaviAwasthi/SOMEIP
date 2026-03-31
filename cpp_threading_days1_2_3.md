# Modern C++ Multithreading — Days 1–5

---

## Day 1 — Concurrency Fundamentals

### Core Concepts

#### Process vs Thread

| Concept | Process | Thread |
|---------|---------|--------|
| Memory | Own address space | Shared within process |
| Creation | Expensive (fork) | Lightweight |
| Communication | IPC (pipes, sockets) | Shared variables |
| Crash isolation | Yes | No — one thread crash kills all |

A **process** is an independent program in execution. A **thread** is a unit of execution within a process. All threads in a process share heap, globals, and file descriptors, but each has its own stack and registers.

#### Concurrency vs Parallelism

- **Concurrency** — multiple tasks are in progress at the same time (may interleave on one CPU)
- **Parallelism** — multiple tasks execute simultaneously on multiple CPUs

```
Concurrency (1 core):    Task A ---> Task B ---> Task A --->
Parallelism (2 cores):   Core 1: Task A ============>
                         Core 2: Task B ============>
```

#### Context Switching

The OS scheduler saves and restores CPU state (registers, program counter, stack pointer) when switching between threads. Context switches are expensive — avoid creating excessive threads.

#### CPU Cores & Hyperthreading

- **Physical cores** — actual hardware execution units
- **Logical cores** — hyper-threading exposes 2 threads per physical core
- `std::thread::hardware_concurrency()` returns the number of logical cores

#### Data Races

A **data race** occurs when:
1. Two or more threads access the same memory location
2. At least one access is a write
3. There is no synchronization

Data races produce **undefined behavior** in C++. The program may crash, produce wrong results, or appear to work correctly.

```cpp
int counter = 0;

void bad_increment() {
    ++counter;  // NOT atomic: read -> modify -> write (3 operations)
                // Any thread can interrupt between these steps
}
```

#### Determinism vs Non-Determinism

Single-threaded programs are deterministic — same input always gives same output. Multithreaded programs are non-deterministic — thread scheduling varies between runs, which makes bugs hard to reproduce.

---

### std::thread — Practical Usage

#### Creating a Thread

```cpp
#include <iostream>
#include <thread>

void worker(int id) {
    std::cout << "Thread " << id << "\n";
}

int main() {
    std::thread t1(worker, 1);
    t1.join();  // wait for t1 to finish
    return 0;
}
```

#### join() vs detach()

```cpp
std::thread t(worker, 1);

t.join();    // Block until thread finishes. Thread resources cleaned up.
             // Must call this OR detach() before destructor.

t.detach();  // Thread runs independently. You lose control over it.
             // Use only when you don't care about completion or result.
```

If a `std::thread` object is destroyed while still joinable (neither joined nor detached), `std::terminate()` is called — the program aborts.

#### Passing Parameters

```cpp
void print(int x, std::string msg) {
    std::cout << msg << ": " << x << "\n";
}

int main() {
    std::thread t(print, 42, "value");  // args copied by default
    t.join();
}
```

Passing by reference requires `std::ref()`:

```cpp
void increment(int& value) { ++value; }

int main() {
    int x = 0;
    std::thread t(increment, std::ref(x));  // must use std::ref
    t.join();
    std::cout << x;  // 1
}
```

#### Member Function Threads

```cpp
class Worker {
public:
    void run(int id) {
        std::cout << "Worker " << id << "\n";
    }
};

int main() {
    Worker w;
    std::thread t(&Worker::run, &w, 1);  // pointer-to-member, object pointer, args
    t.join();
}
```

#### Lambda Threads

```cpp
int main() {
    std::thread t([]() {
        std::cout << "Lambda thread\n";
    });
    t.join();
}
```

---

### Exercise — 5 Threads Printing IDs

```cpp
#include <iostream>
#include <thread>
#include <vector>

void print_id(int id) {
    std::cout << "Thread ID: " << id << "\n";
}

int main() {
    std::vector<std::thread> threads;

    for (int i = 0; i < 5; ++i)
        threads.emplace_back(print_id, i);

    for (auto& t : threads)
        t.join();

    return 0;
}
```

Run this multiple times — notice the output order changes. This is the nature of concurrency.

---

---

## Day 2 — Thread Lifecycle & RAII

### Thread Lifecycle States

```
         Created
            |
        [started]
            |
         Runnable  <----> Running
            |                |
         [join()]        [detach()]
            |                |
         Joined           Detached
```

#### joinable()

A thread is **joinable** if it represents an active thread of execution.

```cpp
std::thread t;
std::cout << t.joinable();  // false — default constructed, no thread

t = std::thread([]{ /* work */ });
std::cout << t.joinable();  // true

t.join();
std::cout << t.joinable();  // false — already joined
```

A thread becomes non-joinable after:
- `join()` is called
- `detach()` is called
- It is default-constructed (no associated thread)

#### Why Forgetting join() Causes std::terminate

```cpp
void risky() {
    std::thread t([]{ std::cout << "running\n"; });
    // forgot to call t.join() or t.detach()
}   // <-- t's destructor is called here
    // t is still joinable -> std::terminate() -> program aborts

int main() {
    risky();  // program will crash
}
```

The C++ standard mandates this behavior to prevent silent resource leaks.

---

### Move Semantics with Threads

`std::thread` is **move-only** — it cannot be copied, only moved.

```cpp
std::thread t1([]{ std::cout << "hello\n"; });

// std::thread t2 = t1;      // ERROR: copy is deleted
std::thread t2 = std::move(t1);  // OK: t1 no longer owns the thread

std::cout << t1.joinable();  // false — t1 gave up ownership
std::cout << t2.joinable();  // true  — t2 now owns the thread

t2.join();
```

This enables storing threads in containers:

```cpp
std::vector<std::thread> pool;
pool.push_back(std::thread(worker, 1));  // move construction
```

---

### RAII — Resource Acquisition Is Initialization

RAII ties resource lifetime to object lifetime. Destructors guarantee cleanup even when exceptions occur.

Without RAII:
```cpp
void bad() {
    std::thread t(worker);
    if (some_condition)
        throw std::runtime_error("oops");  // t.join() never called -> terminate
    t.join();
}
```

With RAII:
```cpp
void good() {
    ThreadGuard tg(std::thread(worker));  // join happens in destructor
    if (some_condition)
        throw std::runtime_error("oops");  // destructor still runs -> safe
}
```

---

### Build — ThreadGuard Class

```cpp
#include <thread>
#include <stdexcept>

class ThreadGuard {
public:
    explicit ThreadGuard(std::thread t)
        : thread_(std::move(t)) {}

    // No copy
    ThreadGuard(const ThreadGuard&) = delete;
    ThreadGuard& operator=(const ThreadGuard&) = delete;

    ~ThreadGuard() {
        if (thread_.joinable())
            thread_.join();
    }

private:
    std::thread thread_;
};

// Usage:
void safe_thread_usage() {
    ThreadGuard tg(std::thread([]{ std::cout << "safe\n"; }));
    // even if exception thrown here, destructor joins the thread
}
```

Note: C++20 provides `std::jthread` which does this automatically.

---

### Thread IDs

```cpp
std::thread t(worker);
std::cout << t.get_id();           // print thread ID
std::cout << std::this_thread::get_id();  // ID of current thread
t.join();
```

---

---

## Day 3 — Shared Data & Mutex

### Why Race Conditions Occur

`++counter` compiles to three machine instructions:

```asm
MOV EAX, [counter]   ; 1. read
ADD EAX, 1           ; 2. increment
MOV [counter], EAX   ; 3. write
```

If two threads execute this simultaneously:

```
Thread 1 reads counter = 0
Thread 2 reads counter = 0       <- both read before either writes
Thread 1 writes counter = 1
Thread 2 writes counter = 1      <- update lost!
Expected: 2, Got: 1
```

This is a **data race** and is undefined behavior in C++.

---

### Mutual Exclusion — std::mutex

A **mutex** (mutual exclusion) ensures only one thread executes a critical section at a time.

```cpp
#include <mutex>

std::mutex m;
int counter = 0;

void increment() {
    m.lock();      // acquire mutex — blocks if another thread holds it
    ++counter;     // critical section
    m.unlock();    // release mutex — other threads can now acquire
}
```

Never call `lock()`/`unlock()` directly — exceptions between them cause deadlock. Use RAII wrappers instead.

---

### std::lock_guard — Simple RAII Lock

```cpp
std::mutex m;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(m);  // locks on construction
    ++counter;
}   // lock goes out of scope -> mutex released automatically
```

`lock_guard` is the simplest choice. It locks on construction and unlocks on destruction. Cannot be unlocked early.

---

### std::unique_lock — Flexible RAII Lock

```cpp
std::mutex m;

void flexible() {
    std::unique_lock<std::mutex> lock(m);  // locks immediately by default

    // ... do work ...

    lock.unlock();   // can unlock early
    // ... do work without holding lock ...
    lock.lock();     // can re-lock
}   // unlocks again on destruction if still locked
```

`unique_lock` is required for use with `std::condition_variable` (Day 5). It has more overhead than `lock_guard` but is necessary when you need flexible locking.

| Feature | lock_guard | unique_lock |
|---------|-----------|-------------|
| Lock on construct | Yes | Yes (default) |
| Unlock early | No | Yes |
| Re-lock | No | Yes |
| Deferred locking | No | Yes |
| Use with condition_variable | No | Yes |
| Overhead | Lower | Higher |

---

### std::mutex Types

```cpp
std::mutex m;               // basic non-recursive mutex
std::recursive_mutex rm;    // same thread can lock multiple times
std::timed_mutex tm;        // supports try_lock_for / try_lock_until
std::shared_mutex sm;       // multiple readers OR one writer (C++17)
```

For read-heavy workloads, use `shared_mutex` with `shared_lock` (readers) and `unique_lock` (writers):

```cpp
#include <shared_mutex>

std::shared_mutex rw_mutex;
int data = 0;

void read_data() {
    std::shared_lock lock(rw_mutex);  // multiple readers allowed
    std::cout << data;
}

void write_data(int val) {
    std::unique_lock lock(rw_mutex);  // exclusive access
    data = val;
}
```

---

### Exercise — Counter With and Without Mutex

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>

// Without mutex (unsafe)
int unsafe_counter = 0;
void unsafe_increment() {
    for (int i = 0; i < 100000; ++i)
        ++unsafe_counter;  // data race
}

// With mutex (safe)
std::mutex m;
int safe_counter = 0;
void safe_increment() {
    for (int i = 0; i < 100000; ++i) {
        std::lock_guard<std::mutex> lock(m);
        ++safe_counter;
    }
}

void run(void(*fn)(), int& result_ref, const std::string& label) {
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i)
        threads.emplace_back(fn);
    for (auto& t : threads)
        t.join();
    std::cout << label << ": " << result_ref << " (expected 1000000)\n";
}

int main() {
    run(unsafe_increment, unsafe_counter, "Unsafe");
    run(safe_increment,   safe_counter,   "Safe");
}
```

Expected output:
```
Unsafe: 743291 (expected 1000000)   <- varies, always wrong
Safe:  1000000 (expected 1000000)   <- always correct
```

---

### Protecting Class Data

Prefer encapsulating the mutex with the data it protects:

```cpp
class Counter {
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mutex_);
        ++value_;
    }

    int get() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return value_;
    }

private:
    mutable std::mutex mutex_;  // mutable allows locking in const methods
    int value_ = 0;
};
```

---

### Common Pitfalls

**1. Forgetting to lock**
```cpp
std::mutex m;
int x = 0;
void bad()  { ++x; }              // forgot to lock
void good() { std::lock_guard l(m); ++x; }
```

**2. Locking too broadly — hurts performance**
```cpp
void bad() {
    std::lock_guard l(m);
    do_expensive_io();   // IO doesn't need the lock!
    ++counter;
}

void good() {
    do_expensive_io();   // outside lock
    std::lock_guard l(m);
    ++counter;           // only lock what needs protection
}
```

**3. Returning references to protected data**
```cpp
class Bad {
    std::mutex m;
    std::string data;
public:
    std::string& get() {   // returns reference — caller bypasses mutex!
        std::lock_guard l(m);
        return data;
    }
};
```

**4. Calling user-supplied code while holding a lock**
```cpp
// If callback acquires the same mutex -> deadlock
void bad(std::function<void()> callback) {
    std::lock_guard l(m);
    callback();   // dangerous
}
```

---

### Compile & Run

```bash
# Compile with threading support
g++ -std=c++17 -pthread -o program program.cpp

# With thread sanitizer to detect races
g++ -std=c++17 -pthread -fsanitize=thread -o program program.cpp
./program
```

---

## Summary

| Day | Key Takeaway |
|-----|-------------|
| 1 | Threads share memory; unsynchronized writes cause undefined behavior |
| 2 | Always ensure threads are joined or detached; use RAII (ThreadGuard) |
| 3 | Use `std::mutex` with `lock_guard` or `unique_lock` to protect shared data |

---

## Day 4 — Deadlocks & Locking Strategies

### Topics

* Deadlock conditions
* Lock ordering
* `std::scoped_lock`
* `std::try_lock`

### Concepts

#### Deadlock Conditions (Coffman Conditions)

A deadlock occurs when **all four** of these hold simultaneously:

1. **Mutual exclusion** — at least one resource is held in a non-shareable mode.
2. **Hold and wait** — a thread holds a resource while waiting for another.
3. **No preemption** — resources cannot be forcibly taken; only released voluntarily.
4. **Circular wait** — Thread A waits for B's resource, B waits for A's resource.

Break **any one** condition to prevent deadlock.

#### Lock Ordering

Always acquire multiple mutexes in a **consistent global order** across all threads. This eliminates circular wait.

```cpp
// Thread 1 and Thread 2 both lock in the same order: mtx1 → mtx2
std::lock_guard<std::mutex> lk1(mtx1);
std::lock_guard<std::mutex> lk2(mtx2);
```

If Thread 1 locks `mtx1` then `mtx2`, and Thread 2 locks `mtx2` then `mtx1` — that's a deadlock waiting to happen.

#### `std::scoped_lock` (C++17)

Acquires **multiple locks atomically** using a deadlock-avoidance algorithm. You don't need to worry about order.

```cpp
std::scoped_lock lock(mtx1, mtx2); // safe, no deadlock
```

Prefer this over manual lock ordering when locking multiple mutexes at once.

#### `std::try_lock`

Non-blocking attempt to acquire a lock. Returns immediately — either locked or not.

```cpp
if (mtx.try_lock()) {
    // got the lock
    mtx.unlock();
} else {
    // couldn't lock, do something else
}
```

Useful for backoff strategies or when you can do useful work if a lock is unavailable.

### Practical

Create an intentional deadlock (two threads, reversed lock order), then fix it using:

1. **Consistent lock ordering** — enforce a global acquisition order
2. **`std::scoped_lock`** — let the standard library handle it atomically

---

## Day 5 — Condition Variables

### Topics

* `std::condition_variable`
* Producer–Consumer problem
* Spurious wakeups
* Predicate-based waiting

### Concepts

#### `std::condition_variable`

Allows threads to **sleep until a condition becomes true**, avoiding busy-waiting. Must be used with a `std::unique_lock<std::mutex>`.

```cpp
std::condition_variable cv;
std::mutex mtx;
bool ready = false;

// Consumer (waiting thread)
std::unique_lock<std::mutex> lk(mtx);
cv.wait(lk, [] { return ready; }); // sleeps until ready == true

// Producer (signaling thread)
{
    std::lock_guard<std::mutex> lk(mtx);
    ready = true;
}
cv.notify_one(); // wake one waiting thread
```

#### Spurious Wakeups

A thread can wake up from `cv.wait()` without being notified — this is a **spurious wakeup** and is allowed by the C++ standard. Always re-check the condition using a **predicate**:

```cpp
cv.wait(lk, [] { return !queue.empty(); }); // re-checks after every wakeup
```

Without a predicate, you must use a `while` loop manually:

```cpp
while (queue.empty()) {
    cv.wait(lk);
}
```

#### Producer–Consumer Problem

Classic concurrency pattern:

* **Producer** generates data and pushes it to a shared queue.
* **Consumer** waits until data is available, then processes it.
* The condition variable signals the consumer when new data arrives.

#### Thread-Safe Queue (Core Interview Problem)

Combines `mutex` + `condition_variable` to build a safe, blocking queue:

```cpp
template<typename T>
class ThreadSafeQueue {
    std::queue<T> q;
    std::mutex mtx;
    std::condition_variable cv;
public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lk(mtx);
            q.push(std::move(value));
        }
        cv.notify_one();
    }

    T pop() {
        std::unique_lock<std::mutex> lk(mtx);
        cv.wait(lk, [this] { return !q.empty(); });
        T val = std::move(q.front());
        q.pop();
        return val;
    }
};
```

This is one of the most frequently asked C++ concurrency interview problems.

### Phase 2 — Synchronization Primitives (Days 5–8)

Goal: Learn signaling and coordination between threads.

---

| Day | Key Takeaway |
|-----|-------------|
| 1 | Threads share memory; unsynchronized writes cause undefined behavior |
| 2 | Always ensure threads are joined or detached; use RAII (ThreadGuard) |
| 3 | Use `std::mutex` with `lock_guard` or `unique_lock` to protect shared data |
| 4 | Prevent deadlocks via consistent lock ordering or `std::scoped_lock` |
| 5 | Use `condition_variable` with a predicate to safely signal between threads |

**Next:** Day 6 — Futures, Promises & `std::async`

---

## Day 6 — Futures & Async

### Topics

* `std::future`
* `std::promise`
* `std::async`
* `std::packaged_task`

### Concepts

#### The Problem with Raw Threads

`std::thread` provides no built-in mechanism to return a value or propagate exceptions back to the caller. You'd need shared variables and synchronization manually.

```cpp
// The hard way
int result;
std::mutex mtx;
std::thread t([&] {
    std::lock_guard l(mtx);
    result = compute();
});
t.join();
// now result is set, but what about exceptions?
```

`std::future` and related tools solve this cleanly.

---

#### `std::future` and `std::promise`

A **promise** is the write end; a **future** is the read end of a one-shot communication channel.

```cpp
#include <future>

std::promise<int> p;
std::future<int>  f = p.get_future();

std::thread t([&p] {
    p.set_value(42);           // sends the value
});

int result = f.get();          // blocks until value is ready
t.join();

std::cout << result;           // 42
```

`f.get()` blocks the calling thread until the value (or exception) is available. It can only be called **once** — the future is consumed.

Propagating exceptions:
```cpp
std::promise<int> p;
std::future<int>  f = p.get_future();

std::thread t([&p] {
    try {
        throw std::runtime_error("something failed");
    } catch (...) {
        p.set_exception(std::current_exception());
    }
});

try {
    int val = f.get();   // re-throws the stored exception
} catch (const std::exception& e) {
    std::cout << e.what();   // "something failed"
}
t.join();
```

---

#### `std::async` — High-Level Async Execution

`std::async` launches a callable asynchronously and returns a `std::future` for its result. You don't manage threads manually.

```cpp
#include <future>

int compute(int x) { return x * x; }

int main() {
    std::future<int> f = std::async(std::launch::async, compute, 10);
    // ... do other work while compute runs ...
    int result = f.get();   // wait and retrieve
    std::cout << result;    // 100
}
```

##### Launch Policies

```cpp
// std::launch::async  — guaranteed new thread, starts immediately
std::async(std::launch::async, fn);

// std::launch::deferred — lazy evaluation, runs in get() on the calling thread
std::async(std::launch::deferred, fn);

// std::launch::async | std::launch::deferred (default) — implementation decides
std::async(fn);
```

Prefer `std::launch::async` explicitly when you need true parallel execution — the default policy may choose deferred and run everything sequentially.

---

#### `std::packaged_task`

Wraps a callable so it can be executed later, with its result retrievable via a future. Useful for task queues and thread pools.

```cpp
std::packaged_task<int(int, int)> task([](int a, int b) { return a + b; });
std::future<int> f = task.get_future();

std::thread t(std::move(task), 3, 4);  // execute the task in a thread
int result = f.get();                   // 7
t.join();
```

| Tool | Purpose |
|------|---------|
| `promise` / `future` | Manual one-shot value passing between threads |
| `async` | Fire-and-forget async call with automatic future |
| `packaged_task` | Wrap callable for deferred/queued execution |

---

### Exercise — Parallel Sum Using `std::async`

```cpp
#include <iostream>
#include <future>
#include <vector>
#include <numeric>

long long sum_range(const std::vector<int>& v, size_t begin, size_t end) {
    return std::accumulate(v.begin() + begin, v.begin() + end, 0LL);
}

int main() {
    std::vector<int> data(1'000'000, 1);  // 1 million ones

    size_t mid = data.size() / 2;

    // Launch two halves in parallel
    auto f1 = std::async(std::launch::async, sum_range, std::cref(data), 0,   mid);
    auto f2 = std::async(std::launch::async, sum_range, std::cref(data), mid, data.size());

    long long total = f1.get() + f2.get();
    std::cout << "Total: " << total << "\n";  // 1000000
}
```

---

## Day 7 — Thread Pools (Manual Implementation)

### Topics

* Task queue
* Worker threads
* Graceful shutdown
* Work stealing (conceptual)

### Concepts

#### Why Thread Pools

Creating and destroying threads is expensive. A thread pool:
- Pre-creates N worker threads once
- Accepts tasks via a queue
- Reuses threads across many tasks
- Avoids per-task thread creation overhead

#### Design

```
submit(task) ──> [Task Queue] ──> Worker 0
                              ──> Worker 1
                              ──> Worker 2
                              ──> Worker 3
```

Shared state: a `queue<function<void()>>`, protected by a `mutex` and signaled by a `condition_variable`.

#### Graceful Shutdown

Workers block on `cv.wait()` when the queue is empty. On shutdown:
1. Set a `stop` flag.
2. Call `cv.notify_all()` to wake all sleeping workers.
3. Workers check the flag and exit their loop.
4. Main thread joins all worker threads.

#### Work Stealing (Conceptual)

In production thread pools (e.g., Intel TBB), each thread has its own **local deque**. If a thread's queue is empty, it "steals" work from another thread's deque — reducing contention on a single global queue. Not needed for a minimal implementation.

---

### Build — Minimal Thread Pool

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <vector>
#include <future>

class ThreadPool {
public:
    explicit ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }

    // Submit a task and return a future for its result
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using R = std::invoke_result_t<F, Args...>;

        auto task = std::make_shared<std::packaged_task<R()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

        std::future<R> future = task->get_future();

        {
            std::lock_guard<std::mutex> lock(mtx_);
            if (stop_)
                throw std::runtime_error("submit on stopped ThreadPool");
            tasks_.emplace([task]{ (*task)(); });
        }
        cv_.notify_one();
        return future;
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();                  // wake all sleeping workers
        for (auto& t : workers_)
            t.join();
    }

    // Non-copyable
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(mtx_);
                cv_.wait(lock, [this] {
                    return stop_ || !tasks_.empty();
                });
                if (stop_ && tasks_.empty())
                    return;
                task = std::move(tasks_.front());
                tasks_.pop();
            }
            task();   // execute outside the lock
        }
    }

    std::vector<std::thread>          workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex                        mtx_;
    std::condition_variable           cv_;
    bool                              stop_ = false;
};
```

Usage:

```cpp
int main() {
    ThreadPool pool(4);   // 4 worker threads

    std::vector<std::future<int>> results;

    for (int i = 0; i < 8; ++i) {
        results.push_back(pool.submit([i] {
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
            return i * i;
        }));
    }

    for (auto& f : results)
        std::cout << f.get() << " ";   // 0 1 4 9 16 25 36 49
    std::cout << "\n";
}   // pool destructor joins all workers cleanly
```

Key points to note in the implementation:
- `tasks_` holds type-erased `function<void()>` — works with any callable
- `packaged_task` bridges the callable with its `future`
- The worker releases the lock before executing the task — allows other workers to dequeue concurrently
- Destructor sets `stop_`, broadcasts to all workers, then joins — clean shutdown with no tasks dropped

---

## Day 8 — C++20 Additions

### Topics

* `std::jthread` — RAII thread with cooperative cancellation
* `std::stop_token` / `std::stop_source` — cancellation mechanism
* `std::latch` — single-use barrier
* `std::barrier` — reusable barrier
* `std::counting_semaphore` / `std::binary_semaphore`

### Concepts

#### `std::jthread` — The Better Thread

`std::jthread` is `std::thread` with two improvements:
1. **Auto-joins** in its destructor (no more `std::terminate` on missing join)
2. Supports **cooperative cancellation** via `std::stop_token`

```cpp
#include <thread>

int main() {
    std::jthread t([](std::stop_token stoken) {
        while (!stoken.stop_requested()) {
            std::cout << "working...\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
        std::cout << "stopped cleanly\n";
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(350));
    t.request_stop();   // signal the thread to stop
}   // destructor auto-joins — no manual join needed
```

The lambda receives a `stop_token` automatically if its first parameter is `std::stop_token`. The thread checks `stop_requested()` and exits voluntarily — this is **cooperative** cancellation, not forced termination.

---

#### `std::stop_token` / `std::stop_source`

These decouple the cancellation signal from `jthread` so you can use them independently.

```cpp
std::stop_source source;
std::stop_token  token = source.get_token();

std::jthread t([token] {
    while (!token.stop_requested()) {
        // ... work ...
    }
});

source.request_stop();   // signal cancellation from any thread
```

You can also register callbacks:

```cpp
std::stop_callback cb(token, [] {
    std::cout << "cleanup on cancel\n";
});
```

---

#### `std::latch` — Single-Use Countdown

A latch starts at a count N. Threads call `count_down()` to decrement it. Any thread that calls `wait()` blocks until the count reaches zero. **Cannot be reset** — use once and discard.

```cpp
#include <latch>

std::latch ready(3);   // count = 3

void worker(std::latch& l, int id) {
    std::this_thread::sleep_for(std::chrono::milliseconds(id * 50));
    std::cout << "Worker " << id << " ready\n";
    l.count_down();    // decrement; does not block
}

int main() {
    std::latch l(3);
    std::jthread t1(worker, std::ref(l), 1);
    std::jthread t2(worker, std::ref(l), 2);
    std::jthread t3(worker, std::ref(l), 3);

    l.wait();          // blocks until all 3 count down
    std::cout << "All workers ready — proceeding\n";
}
```

Use cases: waiting for all threads to finish initialization, implementing a start gun.

---

#### `std::barrier` — Reusable Synchronization Point

All participating threads must reach the barrier before any can proceed. Unlike `latch`, a `barrier` **resets automatically** for the next phase. Accepts an optional completion function called once per cycle.

```cpp
#include <barrier>

std::barrier sync(3, [] { std::cout << "--- phase complete ---\n"; });

void worker(std::barrier<>& b, int id) {
    for (int phase = 0; phase < 3; ++phase) {
        std::cout << "Thread " << id << " phase " << phase << "\n";
        b.arrive_and_wait();   // wait for all 3 threads, then continue
    }
}

int main() {
    std::barrier b(3, [] { std::cout << "--- barrier ---\n"; });
    std::jthread t1(worker, std::ref(b), 1);
    std::jthread t2(worker, std::ref(b), 2);
    std::jthread t3(worker, std::ref(b), 3);
}
```

Use cases: iterative parallel algorithms where all threads must finish phase N before any starts phase N+1.

---

#### `std::counting_semaphore` / `std::binary_semaphore`

A semaphore controls access to a **pool of N resources**. Threads call `acquire()` (blocks if count is 0) and `release()` (increments count, waking a waiter).

```cpp
#include <semaphore>

// Allow at most 3 threads to access the resource concurrently
std::counting_semaphore<3> sem(3);

void worker(int id) {
    sem.acquire();   // blocks if 3 others are already inside
    std::cout << "Thread " << id << " in critical section\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    sem.release();
}
```

`std::binary_semaphore` is `std::counting_semaphore<1>` — behaves like a mutex but can be released from a **different thread** (unlike a mutex, which must be unlocked by the locking thread).

```cpp
std::binary_semaphore signal(0);   // starts at 0 (locked)

std::jthread producer([&] {
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
    signal.release();   // "post" from producer
});

signal.acquire();   // consumer blocks until producer signals
std::cout << "consumer received signal\n";
```

---

### Comparison Table

| Primitive | Reusable | Direction | Typical Use |
|-----------|----------|-----------|-------------|
| `latch` | No | N→0 | Wait for N tasks to complete once |
| `barrier` | Yes | N arrive, all continue | Phase synchronization in iterations |
| `counting_semaphore` | Yes | N slots | Limit concurrency to N threads |
| `binary_semaphore` | Yes | 0/1 | Cross-thread signal / mutex-like |
| `jthread` | — | — | Replace `thread` everywhere |

---

### Build — Parallel Pipeline with `latch` and `jthread`

```cpp
#include <iostream>
#include <latch>
#include <barrier>
#include <thread>
#include <vector>

int main() {
    const int N = 4;
    std::latch   init_done(N);
    std::barrier phase_sync(N, [] { std::cout << "[barrier]\n"; });

    std::vector<std::jthread> workers;
    for (int i = 0; i < N; ++i) {
        workers.emplace_back([&, i](std::stop_token st) {
            // Init phase
            std::cout << "Init " << i << "\n";
            init_done.count_down();
            init_done.wait();   // wait for all to finish init

            for (int phase = 0; phase < 3 && !st.stop_requested(); ++phase) {
                std::cout << "Worker " << i << " phase " << phase << "\n";
                phase_sync.arrive_and_wait();
            }
        });
    }
}   // jthreads auto-join
```

---

### Summary

| Day | Key Takeaway |
|-----|-------------|
| 1 | Threads share memory; unsynchronized writes cause undefined behavior |
| 2 | Always ensure threads are joined or detached; use RAII (ThreadGuard) |
| 3 | Use `std::mutex` with `lock_guard` or `unique_lock` to protect shared data |
| 4 | Prevent deadlocks via consistent lock ordering or `std::scoped_lock` |
| 5 | Use `condition_variable` with a predicate to safely signal between threads |
| 6 | Use `std::async` / `future` / `promise` to pass results between threads |
| 7 | Thread pools reuse threads; `packaged_task` + `future` enable result retrieval |
| 8 | C++20: `jthread` auto-joins, `stop_token` cancels, `latch`/`barrier`/`semaphore` coordinate |
