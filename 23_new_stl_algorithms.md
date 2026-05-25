# Chapter 23: New STL Algorithms in Detail

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 23, pp. 279-291.*

This chapter focuses on algorithms **new or redesigned** in C++17: `for_each_n`, numeric reductions and scans (`reduce`, `transform_reduce`, inclusive/exclusive scans and transform variants), including interactions with parallel execution policies from Chapter 22.

---

## 23.1 `std::for_each_n()` (p. 279)

Applies a function to **the first `n` elements** starting at an iterator.

**[OK]** Return value: an iterator **one past the last processed element** (i.e. the position in the sequence after applying `f` to `n` elements), not merely an opaque completion flag.

### Signature (conceptual)

```cpp
template<class InputIt, class Size, class Function>
InputIt for_each_n(InputIt first, Size n, Function f);
```

- **`f`** is invoked for each of the `n` elements.
- **Returns** an iterator **past the last processed element** (`first + n` if `first` is a random-access iterator and `n` is valid; in general, after advancing `n` times).

### Example

```cpp
#include <vector>
#include <algorithm>
#include <iostream>

int main() {
    std::vector<int> v{10, 20, 30, 40, 50};

    auto it = std::for_each_n(v.begin(), 3, [](int& x) { x *= 2; });
    // v == {20, 40, 60, 40, 50}
    // it points at first unmodified element (40)

    std::cout << "distance: " << (it - v.begin()) << '\n';  // 3
}
```

### Overload with execution policy

```cpp
std::for_each_n(std::execution::par, v.begin(), n, [](auto& x) { ... });
```

Parallel semantics follow Chapter 22: **no races** on shared state.

---

## 23.2 New Numeric Algorithms (p. 281)

Header: `<numeric>` (same as `accumulate`, `iota`, etc.).

---

## 23.2.1 `std::reduce()` (p. 281)

**Purpose:** Generalized sum / fold with **unspecified** evaluation order when not using `std::execution::seq`, which enables efficient parallel tree reduction.

### Typical uses

```cpp
#include <numeric>
#include <vector>
#include <execution>

std::vector<int> v{1, 2, 3, 4, 5};

// Default: uses std::plus and value-initialized T (0 for int)
int s1 = std::reduce(v.begin(), v.end());

// With initial value
long s2 = std::reduce(v.begin(), v.end(), 10L);

// Custom op (should be associative/commutative for parallel)
int s3 = std::reduce(v.begin(), v.end(), 1, [](int a, int b) { return a * b; });

// Parallel
int s4 = std::reduce(std::execution::par, v.begin(), v.end(), 0);
```

### Reduce vs `accumulate` (execution order)

`accumulate` folds **left to right** deterministically. `reduce` may regroup terms.

---

### ASCII: `accumulate` (left fold order)

```
elements:  e0   e1   e2   e3
           │    │    │    │
           ▼    │    │    │
           +◄───+    │    │
                │    │    │
                ▼    │    │
                +◄───+    │
                     │    │
                     ▼    │
                     +◄───+
                          │
                          ▼
                        result
```

### ASCII: `reduce` (allowed to associate differently)

```
        e0   e1   e2   e3
         ╲ ╱     ╲ ╱
          +       +
           ╲     ╱
            +---+----> result  (actual tree is implementation-defined)
```

**Floating-point:** reordering changes rounding -> **non-bit-identical** results vs `accumulate`.

---

## 23.2.2 `std::transform_reduce()` (p. 283)

**Purpose:** Combine **transform** + **reduce** in one pass. **[OK]** There are two essential forms: a **binary / two-range** overload (generalized **inner product** style: combine pairwise `op2` then reduce with `op1`) and a **unary** overload (**map then reduce**: apply unary `op` to each element, then fold with `binary_op`).

### Binary form (two-range inner-product style)

```cpp
#include <numeric>
#include <vector>

std::vector<int> a{1, 2, 3};
std::vector<int> b{4, 5, 6};

// Like inner_product with std::plus and std::multiplies defaults
int ip = std::transform_reduce(a.begin(), a.end(), b.begin(), 0);
// 1*4 + 2*5 + 3*6 = 32
```

### Unary form

```cpp
#include <numeric>
#include <vector>
#include <functional>

std::vector<int> v{1, 2, 3, 4};
long sqsum = std::transform_reduce(
    v.begin(), v.end(),
    0L,
    std::plus<>{},
    [](int x) { return static_cast<long>(x) * x; });
// 1 + 4 + 9 + 16 = 30
```

### With execution policy

```cpp
double sumsq = std::transform_reduce(
    std::execution::par,
    v.begin(), v.end(),
    0.0,
    std::plus<>{},
    [](double x) { return x * x; });
```

**Notes:**

- Iterator requirements differ between overloads; parallel overloads need forward iterators at minimum (see exact constraints).
- Custom reduction ops should behave sensibly under reordering when using `par`.

---

## 23.2.3 `std::inclusive_scan()` and `std::exclusive_scan()` (p. 287)

**Prefix sums** generalize to arbitrary **associative** binary operations (default `std::plus`).

### Inclusive scan (includes current element)

For input `[e0, e1, e2, ...]` output `o[i] = e0 OP e1 OP ... OP ei`.

```
Input:   3   1   4   1   5  (OP = +)
Output:  3   4   8   9  14
```

### Exclusive scan (excludes current; needs initial `init`)

`o[0] = init` (conceptually), `o[i+1] = init OP e0 OP ... OP ei` (standard indexing details: output has same length as input; see signature).

```
Input:   3   1   4   1   5  (OP = +, init = 0)
Output:  0   3   4   8   9
```

### Example code

```cpp
#include <vector>
#include <numeric>
#include <iostream>

int main() {
    std::vector<int> v{3, 1, 4, 1, 5};

    std::vector<int> inc(v.size());
    std::inclusive_scan(v.begin(), v.end(), inc.begin());
    // inc: 3 4 8 9 14

    std::vector<int> exc(v.size());
    std::exclusive_scan(v.begin(), v.end(), exc.begin(), 0);
    // exc: 0 3 4 8 9
}
```

### Step-by-step ASCII (inclusive vs exclusive, `OP = +`, `init = 0`)

**Inclusive scan** — each output includes the current element:

```
i:         0   1   2   3   4
v[i]:      3   1   4   1   5
               │   │   │   │
inc[i]:      3───4───8───9──14
              └+─┘ └+──┘ └+──┘ ...
  step:     3   3+1  prev+4 prev+1 prev+5
```

**Exclusive scan** — output at index `i` is the sum of inputs *before* `v[i]`; first slot is `init`:

```
i:         0   1   2   3   4
v[i]:      3   1   4   1   5
init=0     │   │   │   │   │
exc[i]:    0───3───4───8───9
            │   └+3─┘ └+4─┘ ...
  step:     0   0+3 0+3+1 ...  (each exc[i] = fold of v[0]..v[i-1] with init)
```

[OK] Sanity: `exclusive_scan(..., init)[i] + v[i]` matches `inclusive_scan` at `i` when the op is `+` and the ranges align (informal check while reading the standard’s indexing).

### Parallel scans

```cpp
std::inclusive_scan(std::execution::par, in.begin(), in.end(), out.begin());
```

Scans can be parallelized with work-efficient algorithms; requirements on `OP` and iterators apply.

---

## 23.2.4 `transform_inclusive_scan` / `transform_exclusive_scan` (p. 289)

Apply unary `transform` to each element, then scan with `binary_op`.

```cpp
#include <vector>
#include <numeric>
#include <functional>

std::vector<int> v{1, 2, 3, 4};
std::vector<int> out(v.size());

// First square, then inclusive plus-scan
std::transform_inclusive_scan(
    v.begin(), v.end(),
    out.begin(),
    std::plus<>{},
    [](int x) { return x * x; });
// v^2: 1 4 9 16 -> prefix: 1 5 14 30
```

**`transform_exclusive_scan`** — transform each element, then exclusive scan (needs `init`):

```cpp
#include <vector>
#include <numeric>
#include <functional>

std::vector<int> v{1, 2, 3, 4};
std::vector<int> out(v.size());

// Square each element, then exclusive plus-scan with init 0 -> 0,1,5,14
std::transform_exclusive_scan(
    v.begin(), v.end(),
    out.begin(),
    0,
    std::plus<>{},
    [](int x) { return x * x; });
// transformed: 1 4 9 16 -> out: 0 1 5 14
```

---

## End-to-end: histogram normalization sketch

```cpp
#include <vector>
#include <numeric>
#include <algorithm>

std::vector<double> cumulative_weights(const std::vector<int>& counts) {
    std::vector<double> w(counts.size());
    std::transform_exclusive_scan(
        counts.begin(), counts.end(),
        w.begin(),
        0.0,
        std::plus<>{},
        [](int c) { return static_cast<double>(c); });
    double total = std::reduce(std::execution::seq, counts.begin(), counts.end(), 0);
    std::for_each(w.begin(), w.end(), [total](double& x) { x /= total; });
    return w;
}
```

---

## Cross-reference

- **Chapter 22:** execution policies, parallel pitfalls (pp. 259-277).
- Josuttis discusses numeric algorithm motivation adjacent to **Section 23.2 (pp. 281-291)**.
