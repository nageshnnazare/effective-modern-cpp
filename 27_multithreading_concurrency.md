# Chapter 27: Multi-Threading and Concurrency

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 27, pp. 321-326.*

This chapter summarizes supplementary synchronization utilities: **`std::scoped_lock`** for multi-mutex locking, **`std::shared_mutex`**, compile-time lock-free queries via **`std::atomic::is_always_lock_free`**, and **cache-line hint constants** for avoiding false sharing.

---

## 27.1 Supplementary Mutexes and Locks (p. 321)

---

## 27.1.1 `std::scoped_lock` (p. 321)

**Problem:** Locking multiple mutexes in a fixed order everywhere is error-prone; **deadlock** can occur if two threads lock `A` then `B` vs `B` then `A`.

**[OOPS] C++14-style manual fix** — correct acquisition order but verbose and easy to get wrong under maintenance:

```cpp
#include <mutex>

std::mutex a, b;

void legacy() {
    std::lock(a, b);                      // deadlock-free acquisition of both
    std::lock_guard<std::mutex> ga(a, std::adopt_lock);
    std::lock_guard<std::mutex> gb(b, std::adopt_lock);
    // ... critical section ...
}
```

**C++17 solution:** `std::scoped_lock` accepts **one or more mutexes** and locks them **atomically** using `std::lock` underneath (avoiding deadlock via the standard lock algorithm).

```cpp
#include <mutex>
#include <thread>

std::mutex m1, m2, m3;

void work() {
    std::scoped_lock lk{m1, m2, m3};
    // all three locked, released at scope end
}
```

**Replaces:** manual `std::lock(m1,m2,m3);` plus multiple `std::lock_guard` or `std::adopt_lock`.

### Class template argument deduction (CTAD)

```cpp
std::scoped_lock lk(m1, m2);  // types deduced
```

No need to spell `std::scoped_lock<std::mutex, std::mutex>` in many cases.

---

### ASCII: deadlock pattern vs `scoped_lock`

```
[ERROR] inconsistent lock order:

 Thread 1             Thread 2
    │                    │
    ▼                    ▼
 lock(A)              lock(B)
    │                    │
    ▼                    ▼
 lock(B)   waits...    lock(A)   waits...
    └──── deadlock cycle ────┘

[OK] std::scoped_lock{A,B} / std::lock algorithm:

 Both threads request {A,B} as a set -> acquiring all or none
 without requiring manual global ordering discipline in user code.
```

---

## 27.1.2 `std::shared_mutex` (p. 322)

`std::shared_mutex` is a **reader-writer** mutex:

- Many **shared** (read) locks **or** one **exclusive** (write) lock.

### Readers

```cpp
#include <shared_mutex>
#include <vector>

std::shared_mutex rw;
std::vector<int> data{1, 2, 3, 4, 5};

int read_sum() {
    std::shared_lock<std::shared_mutex> lk(rw);
    int s = 0;
    for (int x : data) s += x;
    return s;
}
```

### Writers

```cpp
void write_push(int x) {
    std::unique_lock<std::shared_mutex> lk(rw);
    data.push_back(x);
}
```

### Difference vs `shared_timed_mutex`

- `shared_mutex`: **no** `try_lock_for` / `try_lock_until` on the standard mutex object (timed operations were on timed mutex types).
- Potentially simpler / faster on platforms where timed sharing is unnecessary (implementation-defined).
- **[OK]** Typical pattern: **`std::shared_lock<shared_mutex>`** for **readers** (many concurrent shared locks) and **`std::unique_lock<shared_mutex>`** for **writers** (exclusive access). Together this implements **multiple readers OR one writer**.

### ASCII: reader-writer pattern

```
              shared_mutex state
        ┌─────────────────────────────┐
        │  readers: R (count)         │
        │  writer:  W (exclusive)     │
        └──────────────┬──────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
  many shared_lock   one unique_lock   blocked combos
  allowed while      upgrades path     (writer vs readers)
  writer absent      excludes readers
```

---

## 27.2 `is_always_lock_free` (p. 323)

`std::atomic<T>` provides:

- `bool is_lock_free() const noexcept` -> **runtime** query (constexpr in practice on many implementations but specified behavior is runtime for historical reasons in some cases).
- **`static constexpr bool is_always_lock_free`** -> **compile-time** constant.

```cpp
#include <atomic>

static_assert(std::atomic<int>::is_always_lock_free);
// static_assert(std::atomic<long double>::is_always_lock_free);  // may fail on some platforms
```

**Use cases:**

- **SFINAE** or `static_assert` in generic lock-free code.
- Choosing alternative representations when atomics would silently use locks.

---

## 27.3 Cache Line Sizes (p. 324)

Two **implementation-defined** constants (typically usable with `alignas`):

| Constant | Intent |
|----------|--------|
| `std::hardware_destructive_interference_size` | Minimum offset between objects to avoid **false sharing** (independent atomics mutated by different threads landing in one cache line). |
| `std::hardware_constructive_interference_size` | Maximum size to promote **true sharing** when data is intentionally accessed together. |

**Typical platform note:** `hardware_destructive_interference_size` is **often 64 bytes** on widely deployed CPUs—still treat it as **implementation-defined** and use the constant (do not hard-code `64` in portable logic). **`hardware_constructive_interference_size`** suggests an upper bound for grouping **related** fields so they **fit in one cache line** when you *want* them fetched together (the opposite goal of destructive separation).

```cpp
// Pad hot independent atomics to different lines:
struct alignas(std::hardware_destructive_interference_size) PaddedData {
    std::atomic<int> counter{};
};
```

### False sharing example (conceptual)

```cpp
#include <new>
#include <atomic>
#include <thread>

struct alignas(std::hardware_destructive_interference_size) Counter {
    std::atomic<int> v{0};
};

Counter a, b;

void workers() {
    std::thread t1([&] { for (int i = 0; i < 1'000'000; ++i) ++a.v; });
    std::thread t2([&] { for (int i = 0; i < 1'000'000; ++i) ++b.v; });
    t1.join();
    t2.join();
}
```

Without separation, `a.v` and `b.v` could reside in the **same cache line**, causing cache line bouncing.

---

### ASCII: false sharing layout

```
Without padding:

 cache line (64 bytes typical)
┌──────────────────────────────────────┐
│ atomic a (4-8B) │ atomic b │ padding │
└──────────────────────────────────────┘
  CPU0 writes a          CPU1 writes b
        │                        │
        └── both ping cache line ──┘  [slow: false sharing]

With alignas(destructive_interference_size):

┌──────── cache line 0 ────────┐ ┌──────── cache line 1 ────────┐
│ atomic a ...                 │ │ atomic b ...                 │
└──────────────────────────────┘ └──────────────────────────────┘
```

**Note:** Values are **hints**; portable code should not assume exact sizes beyond "avoid / promote sharing at coarse granularity".

---

## Putting it together: concurrent counter service

```cpp
#include <shared_mutex>
#include <mutex>
#include <map>
#include <string>
#include <optional>

class Config {
    mutable std::shared_mutex rw;
    std::map<std::string, std::string> kv;

public:
    void set(std::string k, std::string v) {
        std::unique_lock<std::shared_mutex> lk(rw);
        kv[std::move(k)] = std::move(v);
    }

    std::optional<std::string> get(const std::string& k) const {
        std::shared_lock<std::shared_mutex> lk(rw);
        auto it = kv.find(k);
        if (it == kv.end()) return std::nullopt;
        return it->second;
    }
};
```

---

## Cross-reference

- **Chapter 22:** parallel algorithms require thread-safe element operations.
- Memory order on atomics: still `<atomic>` / `std::memory_order` (consult memory model chapters in Josuttis beyond p. 326 as your edition organizes them).
