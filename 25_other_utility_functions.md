# Chapter 25: Other Utility Functions and Algorithms

*Reference: Nicolai M. Josuttis, **C++17 - Part IV - The Complete Guide**, Chapter 25, pp. 299-307.*

This chapter covers small but broadly useful C++17 additions: **`size`**, **`empty`**, **`data`** free functions; **`as_const`**; **`clamp`**; and **`sample`**.

---

## 25.1 `size()`, `empty()`, `data()` (p. 299)

These **function templates** unify access to `.size()`, `.empty()`, `.data()` for containers **and** raw C arrays.

### `std::size`

```cpp
#include <iterator>
#include <vector>

void demo_vector() {
    std::vector<int> v{1, 2, 3};
    static_assert(std::size(v) == 3);
}

void demo_c_array() {
    int arr[] = {10, 20, 30, 40};
    constexpr std::size_t n = std::size(arr);  // 4
    (void)n;

    int three[] = {1, 2, 3};
    static_assert(std::size(three) == 3);  // [OK] std::size on bound C array yields element count
}
```

**Why it matters:** templates can write `std::size(x)` without special-casing pointers vs. containers. **Note:** Using `std::size` on a pointer is ill-formed unless you have a `std::extent` array type---raw pointers are not arrays.

### `std::empty`

```cpp
#include <iterator>
#include <vector>

bool demo() {
    std::vector<int> v;
    int arr[] = {1};
    return std::empty(v) && !std::empty(arr);
}
```

Works for containers with `empty()`, for `initializer_list`, for C arrays (zero elements -> empty), and strings.

### `std::data`

```cpp
#include <iterator>
#include <vector>
#include <cstring>

void fill(std::vector<int>& v) {
    int* p = std::data(v);  // [OK] pointer to underlying contiguous element storage
    std::memset(p, 0, std::size(v) * sizeof(int));  // example only
}

void c_array_ptr() {
    int arr[] = {1, 2, 3};
    int* p = std::data(arr);  // points to arr[0]
    (void)p;
}
```

**Use case:** generic code interoperating with C APIs expecting pointers + lengths.

---

## 25.2 `as_const()` (p. 302)

```cpp
template<typename T>
constexpr const T& as_const(T& t) noexcept { return t; }
```

**Effect:** obtains a **const lvalue reference** even if the original object is non-const.

### Practical uses

**[OK]** Force const overloads:

```cpp
#include <utility>

struct Widget {
    void work() &       { /* mutable path */ }
    void work() const&  { /* const path */ }
};

void call_const_path(Widget& w) {
    w.work();               // calls non-const &
    std::as_const(w).work(); // calls const &
}
```

**[OK]** Lambda capture holding const reference without copying:

```cpp
#include <utility>

template<typename T>
auto make_logger(T& value) {
    // capture as const ref to accidental mutation in logger
    return [val = std::as_const(value)] { /* read val */ };
}
```

**Limitation:** `as_const` does not help with **deep const** for pointers (`const T*` vs `T*`).

---

## 25.3 `clamp()` (p. 303)

```cpp
#include <algorithm>

template<class T>
constexpr const T& clamp(const T& v, const T& lo, const T& hi);

template<class T, class Compare>
constexpr const T& clamp(const T& v, const T& lo, const T& hi, Compare comp);
```

**Semantics (without custom comparator):**

- If `v < lo` -> return `lo`
- Else if `hi < v` -> return `hi`
- Else return `v`

### Examples

```cpp
#include <algorithm>
#include <iostream>

int main() {
    std::cout << std::clamp(42, 0, 100) << '\n';   // 42
    std::cout << std::clamp(-5, 0, 100) << '\n';   // 0
    std::cout << std::clamp(150, 0, 100) << '\n';  // 100

    // Custom comparator (e.g., absolute tolerance domain)
    auto abs_less = [](double a, double b) { return a < b; };
    double x = std::clamp(1.7, 0.0, 1.0, abs_less);
    (void)x;
    return 0;
}
```

**[ERROR] Undefined behavior** if `lo > hi` (precondition).

### Use cases

- GUI sliders mapping input to bounds.
- Physics or game clamps.
- Normalizing user configuration values.

---

## 25.4 `sample()` (p. 304)

**Signature shape (informal):** `std::sample(first, last, out, n, rng)` — **[OK]** the sample **preserves relative order** of the chosen elements as they appeared in `[first, last)`.

```cpp
#include <random>
#include <algorithm>
#include <numeric>
#include <vector>
#include <iterator>
#include <iostream>

int main() {
    std::vector<int> population(100);
    std::iota(population.begin(), population.end(), 0);

    std::vector<int> out;
    std::sample(
        population.begin(), population.end(),
        std::back_inserter(out),
        10,
        std::mt19937{std::random_device{}()});

    // out has 10 distinct picks (without replacement)
    // relative order among sampled elements follows their order in population

    for (int x : out) std::cout << x << ' ';
    std::cout << '\n';
}
```

### Properties (informal)

- **Without replacement** sampling: each element appears at most once in output (for the random-access iterator overloads / as specified---consult signature used).
- **Preserves relative order** of selected elements as in the input range (stable selection property).
- **Reservoir / selection** algorithms are used internally; complexity and behavior are defined in the standard.

### Practical scenario

- Audit logging: pick random subset of requests for deep inspection.
- A/B testing cohort selection from a known ID list.
- Testing: shrink large fixtures to reproducible subsets **if you seed RNG**.

### Reproducible sampling

```cpp
std::mt19937 rng(12345);  // fixed seed for tests
std::sample(first, last, out, n, rng);
```

---

## Summary table

| Facility | Header | Role |
|----------|--------|------|
| `std::size` | `<iterator>` | Element count for containers & arrays |
| `std::empty` | `<iterator>` | Generic emptiness |
| `std::data` | `<iterator>` | Pointer to contiguous elements |
| `std::as_const` | `<utility>` | Const view of object |
| `std::clamp` | `<algorithm>` | Bound a value |
| `std::sample` | `<algorithm>` | Random subset selection |

---

## Cross-reference

- Random engines: `<random>` (Josuttis covers alongside algorithmic additions around pp. 299-307).
- **Chapter 28:** other small utilities (e.g. chrono rounding) complement these patterns.
