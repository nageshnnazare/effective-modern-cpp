# Chapter 22: Parallel STL Algorithms

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 22, pp. 259-277.*

Parallel algorithms allow standard library algorithms to exploit multiple threads (and, for `par_unseq`, vectorization hints) while sharing a uniform interface through **execution policies** in `<execution>`.

---

## 22.0 `Timer` helper for micro-benchmarks (p. 259)

Josuttis pairs the parallel examples with a small **`Timer`** based on **`std::chrono::steady_clock`** (monotonic, suited to elapsed-time measurement).

```cpp
#include <chrono>
#include <iostream>

class Timer {
    std::chrono::steady_clock::time_point start_;
public:
    Timer() : start_{std::chrono::steady_clock::now()} {}

    void printDiff(const char* msg) const {
        const auto stop = std::chrono::steady_clock::now();
        const auto dur  = stop - start_;
        const double ms = std::chrono::duration<double, std::milli>(dur).count();
        std::cout << msg << ms << " msec\n";
    }
};
```

- [OK] Use **`steady_clock`**, not `system_clock`, so wall-clock adjustments do not skew timings.
- [OOPS] Comparing runs on different machines or under heavy load still requires care; this is a **relative** helper for A/B on one setup.

---

## 22.1 Using Parallel Algorithms (p. 260)

The first parameter is an **execution policy** object that selects sequential, parallel, or parallel-unsequenced execution.

### Basic pattern

```cpp
#include <vector>
#include <algorithm>
#include <execution>

void sort_parallel(std::vector<int>& v) {
    std::sort(std::execution::par, v.begin(), v.end());
}
```

**Requirements:**

- Include `<execution>` for policy names.
- Iterator requirements depend on the algorithm (typically forward or better; parallel versions may impose stronger requirements---consult the standard for each algorithm).
- **Correctness is your responsibility:** overlapping writes, unsynchronized shared mutable state, and iterator invalidation rules still apply.

---

## 22.1.1 Parallel `for_each()` (p. 260)

```cpp
#include <vector>
#include <algorithm>
#include <execution>
#include <atomic>

void scale_ok(std::vector<double>& v) {
    std::for_each(std::execution::par, v.begin(), v.end(), [](double& x) {
        x *= 2.0;  // independent elements: OK
    });
}

void counter_problem() {
    std::vector<int> v(1000, 1);
    int bad = 0;  // [ERROR] data race if updated from multiple tasks
    std::for_each(std::execution::par, v.begin(), v.end(), [&](int) {
        ++bad;   // undefined behavior
    });
}

void counter_fixed() {
    std::vector<int> v(1000, 1);
    std::atomic<int> ok{0};
    std::for_each(std::execution::par, v.begin(), v.end(), [&](int) {
        ok.fetch_add(1, std::memory_order_relaxed);
    });
}
```

**WARNING (p. 260):** Any **shared mutable state** updated from the body must use atomics, locks, or other synchronization. Unsynchronized updates are **data races** -> undefined behavior.

### Full `parforeachloop` example with timings (pp. 260-263)

Pattern (after Josuttis ``parforeachloop.cpp``): fill a vector, then time **`for_each`** with **`std::execution::seq`** vs **`std::execution::par`** for varying **`numElems`**, using the **`Timer`** from [22.0](#220-timer-helper-for-micro-benchmarks-p-259).

```cpp
#include <vector>
#include <algorithm>
#include <execution>
#include <cmath>

// Timer as in section 22.0

void foreach_test(std::size_t numElems) {
    std::vector<double> coll;
    coll.reserve(numElems);
    for (std::size_t i = 0; i < numElems; ++i) {
        coll.push_back(static_cast<double>(i % 1000));
    }

    auto bulk_op = [](double& val) {
        for (int i = 0; i < 100; ++i) {
            val = std::sqrt(val * val * val);
        }
    };

    // sequenced
    {
        Timer t;
        std::for_each(std::execution::seq, coll.begin(), coll.end(), bulk_op);
        t.printDiff("sequenced for_each: ");
    }
    // parallel
    {
        Timer t;
        std::for_each(std::execution::par, coll.begin(), coll.end(), bulk_op);
        t.printDiff("parallel   for_each: ");
    }
}
```

**Representative outcomes (Josuttis, one machine; qualitative):**

```
+------------+--------------------------+---------------------------+
| numElems   | faster policy            | comment                   |
+------------+--------------------------+---------------------------+
| 100        | sequential (~10x)        | scheduling dwarfs work    |
| 10 000     | roughly break-even         | crossover region          |
| 1 000 000  | parallel (~3x)           | bulk work amortizes cost  |
+------------+--------------------------+---------------------------+
```

- [OK] Large ranges with **CPU-heavy** per-element bodies favor **`par`**.
- [ERROR] Tiny ranges: **`par`** often loses badly to **`seq`**.

### `count_if` and trivial predicates (p. 263)

For a **simple** predicate (e.g. `val > 3`) on plain scalars, **`count_if(..., par, ...)`** is typically **not** worth it: counting is memory-bandwidth light and the parallel split/merge overhead dominates. Josuttis notes this pattern explicitly----**parallel `count_if` for cheap predicates never pays** on the examples he tried.

---

## 22.1.2 Parallel `sort()` (p. 263)

```cpp
std::sort(std::execution::par, v.begin(), v.end());
```

- May partition work across threads for large ranges.
- **Not a stable sort** when you need stability use `stable_sort` (also has parallel overload).
- **Small inputs:** threading overhead may dominate; see 22.4.

### Sorting strings like `"id0"`, `"ID0"`, `"id1"`, ... (pp. 263-265)

Josuttis builds many strings of the form **`id0` / `ID0` / `id1` / `ID1` / ...**, then compares:

1. **`std::sort(seq, ...)`** vs **`std::sort(par, ...)`** (same comparator on `std::string`).
2. Storing views: sort **`std::vector<std::string>`** but compare **`std::string_view`** fragments (e.g. **`substr` / substring-style key**) to reduce comparison cost.
3. **Combined:** **`par` + cheaper comparisons** (often via **`std::string_view`**) yields the largest win----on the book's setup, **up to ~10x** speedup vs the naive sequential `std::string` sort (exact factor platform-dependent).

```cpp
#include <algorithm>
#include <execution>
#include <cctype>
#include <string>
#include <string_view>
#include <vector>

// Sketch: large vector of std::string keys "id0", "ID0", ...
// Compare using string_view to avoid repeated internal access patterns
// the book micro-optimizes; key idea: cheap view + parallel sort
bool sv_case_insensitive(std::string_view a, std::string_view b) {
    // book uses a concrete tolerant comparison; keep locale/policy in mind
    return std::lexicographical_compare(
        a.begin(), a.end(), b.begin(), b.end(),
        [](char x, char y) {
            return std::tolower(static_cast<unsigned char>(x))
                 < std::tolower(static_cast<unsigned char>(y));
        });
}

void sort_strings_parallel(std::vector<std::string>& coll) {
    std::sort(std::execution::par, coll.begin(), coll.end(), sv_case_insensitive);
}
```

- [OK] **`string_view`** in the comparator reduces per-comparison overhead when strings share structure.
- [OK] **`par`** scales when **`N`** is large enough.
- [OOPS] Picking the wrong comparator (locale, Unicode) changes correctness; this is still **your** contract.

---

## 22.2 Execution Policies (p. 265)

| Policy | Meaning |
|--------|---------|
| `std::execution::seq` | **Sequenced policy:** runs like a classic serial algorithm in the **calling thread**; invocations are **indeterminately sequenced** with respect to each other only as the standard already allows, but you keep the usual **deterministic algorithmic order** expectations for element processing as documented for that algorithm. |
| `std::execution::par` | **Parallel policy:** library **may** run element work on **multiple threads**; elements must not introduce **data races** or conflicting side effects. |
| `std::execution::par_unseq` | **Parallel-unsequenced policy:** **may** vectorize (**SIMD**) **and** execute across threads; invocations may be **interleaved** even inside a single thread. Callables **must not** use **mutexes**, **blocking** calls, or other synchronization that assumes a strict thread-like sequencing of operations. |

Header: **`<execution>`** (defines `std::execution::seq`, `std::execution::par`, `std::execution::par_unseq`).

**Details (Josuttis p. 265):**

- **`seq`:** [OK] Same-thread sequential flavor; **deterministic** processing order from the **algorithm's** point of view (as with the non-policy overload). Good baseline and fallback.
- **`par`:** [OK] Threads may cooperate; **you** guarantee the element function is safe under **concurrent** execution on **different** elements. [ERROR] Shared unsynchronized mutable state -> data race.
- **`par_unseq`:** [OK] Maximum freedom for the implementation (vectorization + threading). [ERROR] **`std::mutex`**, **`scoped_lock`**, condition variables, I/O waits, or **blocking** in the element function -> **[OOPS]** you violated the policy's requirements; **undefined behavior** territory.

**Semantic sketch:**

- `seq`: one logical thread, strict induction order expectations for element access side effects (as per algorithm).
- `par`: multiple threads; function objects must tolerate **concurrent** invocation on **different** elements.
- `par_unseq`: may interleave vector lanes; callable must be safe for **vectorization** (no conflicting unsequenced memory dependencies the compiler/library cannot reason about).

---

### ASCII: execution policy decision tree

```
                    ┌─────────────────────────────┐
                    │ Need parallel speedup?      │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              │ no                                      │ [OK] use seq
              ▼                                         │
     ┌────────────────┐                                 │
     │ Large range +  │                                 │
     │ CPU-bound work │                                 │
     └────────┬───────┘                                 │
              │ yes                                     │
              ▼                                         │
     ┌────────────────────────────┐                     │
     │ Callable side-effect free  │                     │
     │ per element (no races)?    │                     │
     └────────────┬───────────────┘                     │
                  │ no -> [ERROR] fix design            │
                  │ yes                                 │
                  ▼                                     │
     ┌────────────────────────────┐                     │
     │ Need SIMD / unsequenced?   │                     │
     └────────────┬───────────────┘                     │
            yes   │   no                                │
               ┌──┴──┐                                  │
               ▼     ▼                                  │
          par_unseq par                                 │
              │     │                                   │
              └─────┴───────────────────────────────────┘
                        -> pick policy, measure
```

---

## 22.3 Exception Handling (p. 266)

Parallel algorithm **overloads** (the ones whose **first** argument is an **`ExecutionPolicy`**) treat escaping exceptions from **element access** differently from **classical** overloads with **no** policy.

- [OK] **Non-policy** algorithms (e.g. plain **`std::for_each(first, last, f)`**): if **`f`** throws, the exception **propagates** normally out of **`for_each`**----familiar **`try` / `catch`** works.
- [ERROR] **Policy** algorithms (**`std::for_each(execution::seq, ...)`**, **`par`**, **`par_unseq`**, ...): if an **element access** exits via an **uncaught** exception, the program ends by calling **`std::terminate()`**----there is **no** portable guarantee that your **`catch`** around the algorithm will run.

Josuttis stresses this contrast: **`seq`** still **disables** actual parallelism, but you are in the **ExecutionPolicy overload set**, so you are still subject to the **terminate-on-exception** rule for these algorithms.

**Practical guidance:**

- Prefer `noexcept` lambdas and validate inputs **before** calling parallel algorithms.
- Do not rely on exceptions for control flow inside parallel element bodies.
- Treat element bodies like **thread pools**: return error codes to shared state only if that state is **race-free**.

---

## 22.4 Benefit of NOT Using Parallel Algorithms (p. 266)

Parallelism is **not** free.

- **Overhead:** thread scheduling, work partitioning, synchronization inside the library.
- **Small `N`:** `seq` or plain algorithm can be faster.
- **Not all algorithms scale:** memory bandwidth, contention, Amdahl limits.
- **Developer cost:** reasoning about races is harder.

Rule of thumb: **benchmark** on target hardware; start with `seq`, promote hot paths to `par` when justified.

---

### ASCII: performance tradeoff (schematic)

```
  time
    ^
    │  parallel sort total cost
    │     ┌───────────────────────────────
    │    /
    │   /   region A: parallel wins (large N)
    │  /
    │ / region B: seq wins (small N - overhead dominates)
    │/──────────────────────────────────────-> N (problem size)
         ^
         └─ crossover point (platform-specific)
```

---

## 22.5 Overview of Parallel Algorithms (p. 267)

C++17 adds overloads taking **`ExecutionPolicy&&` as the first parameter** for the algorithms enumerated below (complete as in Josuttis **pp. 267-269** / ISO C++17 parallel algorithm set). Headers: **`<algorithm>`** and **`<numeric>`** (plus **`<execution>`** for the policies).

**`<algorithm>`---complete list (alphabetical):**

- `adjacent_find`
- `all_of`, `any_of`, `none_of`
- `copy`, `copy_if`, `copy_n`, `move`
- `count`, `count_if`
- `equal`
- `fill`, `fill_n`
- `find`, `find_end`, `find_first_of`, `find_if`, `find_if_not`
- `for_each`
- `generate`, `generate_n`
- `inplace_merge`
- `is_heap`, `is_heap_until`
- `is_sorted`, `is_sorted_until`
- `lexicographical_compare`
- `max_element`, `min_element`, `minmax_element`
- `merge`
- `mismatch`
- `next_permutation`, `prev_permutation`
- `nth_element`
- `partial_sort`, `partial_sort_copy`
- `partition`, `stable_partition`, `partition_copy`
- `remove`, `remove_if`, `remove_copy`, `remove_copy_if`
- `replace`, `replace_if`, `replace_copy`, `replace_copy_if`
- `reverse`, `reverse_copy`
- `rotate`, `rotate_copy`
- `search`, `search_n`
- `sort`, `stable_sort`
- `transform`
- `unique`, `unique_copy`

**`<numeric>`---complete list (alphabetical):**

- `exclusive_scan`
- `inclusive_scan`
- `reduce`
- `transform_exclusive_scan`
- `transform_inclusive_scan`
- `transform_reduce`
- `for_each_n`

**ASCII: where to look first**

```
  <execution>  --------->  policy tags (seq, par, par_unseq)
        |
        v
  first argument of algorithm overload
        |
   +----+-----+
   |          |
<algorithm>  <numeric>   (lists above)
```

**Note:** Parallel overloads impose **extra iterator / numeric / complexity** requirements (e.g. mutable iterators for mutating algorithms, associativity assumptions for `reduce`/`scan` under `par`). **`std::accumulate`** has **no** parallel policy overload in C++17----use **`reduce`** instead when reordering is acceptable.

---

## 22.6 Motivation for New Algorithms: `reduce` vs `accumulate` (p. 269)

### `std::accumulate` (historical default)

- **Order is fixed:** evaluation proceeds **strictly left-to-right** over the range, applying the binary operation to the running result and the next element.
- **Deterministic** for the same input range and initial value----even for **floating-point**, you get the **one** canonical left fold (modulo `-O0`/`std::`-exact details you already rely on today).
- A parallel policy overload **does not exist** for `accumulate`----by design, because **reordering is incompatible** with the defined evaluation order.

### `std::reduce` (C++17)

- **Reordering permitted:** the implementation may combine subranges in **tree** fashion (especially under **`par`** / **`par_unseq`**); the **binary operation need not see operands in left-to-right order**.
- For **non-associative** operations (notably **`double` addition**), **different evaluation trees -> different rounding** -> **different observable results** compared to `accumulate`.
- With **`std::execution::seq`**, `reduce` is closer to "like `accumulate`" in spirit, but read the **[algorithm.requirements]** clauses for the exact contract on your iterator category and `T`.

```cpp
#include <vector>
#include <numeric>
#include <execution>

double sum_par(const std::vector<double>& v) {
    return std::reduce(std::execution::par, v.begin(), v.end(), 0.0);
}
```

### Integer example: `accumulate` vs `reduce` still illustrates ordering freedom

```cpp
#include <iostream>
#include <numeric>
#include <vector>

int main() {
    std::vector<long> v{1, 2, 3, 4, 5};
    long a = std::accumulate(v.begin(), v.end(), 0L);              // forced left fold
    long r = std::reduce(std::execution::seq, v.begin(), v.end(), 0L);
    std::cout << "accumulate: " << a << "\n";  // 15
    std::cout << "reduce(seq): " << r << "\n"; // 15 for + on long (associative)
}
```

For **`+`** on **`long`**, `accumulate` and **`reduce(..., seq, ...)`** agree in ordinary situations. Under **`par`**, integer addition still yields the mathematically exact sum (no rounding), but the **`reduce` contract** is what permits **out-of-order combination** of subresults.

### Floating-point caveat (DETAILED)

For **`float`/`double`**, **associativity fails in the reals they approximate**:

- [OK] **`accumulate`** always means **`(((...(init + v[0]) + v[1]) + v[2]) ...)`**----**one** rounding pattern.
- [ERROR] **`reduce(..., par, ...)`** may group as **`(v[0]+v[1]) + (v[2]+v[3]) + ...`** or other trees----**another** rounding pattern.
- [OOPS] **Bit-identical** reproducibility across compilers/machines -> **[ERROR]** do not bet on **`par` reduce** with raw `+` on floats.

**When to prefer which**

| Goal | Prefer |
|------|--------|
| Bit-reproducible left fold | `accumulate` (or `reduce` with `seq`) |
| Speed on large arrays, acceptable numerical drift | `reduce` with `par` |
| Parallel + deterministic requirement | Must use `seq` or a compensated summation strategy (Kahan, etc.) |

### `std::transform_reduce` (pp. 269-277)

**`transform_reduce`** fuses **unary/unary-or-binary transform + reduction** into a **single** parallel-friendly pass: first **map** elements (and optionally pair them with a second range), then **reduce** with your associative/commutative op where required by the policy.

```cpp
#include <vector>
#include <numeric>
#include <functional>
#include <execution>

double sum_of_squares(const std::vector<double>& v) {
    return std::transform_reduce(
        std::execution::par,
        v.begin(), v.end(),
        0.0,
        std::plus<>(),
        [](double x) { return x * x; });
}
```

- [OK] One traversal, good cache behavior, exposes parallelism to the library.
- [ERROR] Same **floating-point non-determinism** caveats as **`reduce`** when you use non-exact ops under **`par`**.

---

## Minimal complete example

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <execution>
#include <numeric>

int main() {
    std::vector<int> v{3, 1, 4, 1, 5, 9, 2, 6, 5};

    long s = std::reduce(std::execution::par, v.begin(), v.end(), 0L);
    std::sort(std::execution::par, v.begin(), v.end());

    std::cout << "sum=" << s << "\n";
    for (int x : v) std::cout << x << ' ';
    std::cout << '\n';
    return 0;
}
```

---

## Cross-reference

- **Chapter 23:** `reduce`, scans, and `for_each_n` in detail (pp. 279-291).
- **Chapter 22 (this file):** policies, pitfalls, algorithm overview (pp. 259-277).
