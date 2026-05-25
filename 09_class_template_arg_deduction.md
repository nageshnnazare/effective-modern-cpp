# Chapter 9: Class Template Argument Deduction (CTAD)

**Book:** *C++17 - The Complete Guide*, Nicolai M. Josuttis  
**Chapter:** 9 - Class Template Argument Deduction  
**Book page span:** **pp. 77-93** (section numbers below align with the book: **9.1**, **9.2**, and so on)

This tutorial is **self-contained** for study, but **page references** (`p.xx`) point into the book for additional nuance, standard citations, and library evolution notes.

---

## Overview

Before C++17, every use of a class template required either **explicit template arguments** or a **factory function** (`std::make_pair`, `std::make_tuple`, ...). C++17 adds **class template argument deduction**: when you name a class template in a context where an object is created, the compiler **deduces** the template arguments from the constructor arguments (and sometimes from **deduction guides**).

```
C++14 style                          C++17 style (CTAD)
───────────────────────────────────  ───────────────────────────────────
std::complex<double> c{5.1, 3.3};    std::complex c{5.1, 3.3};
std::lock_guard<std::mutex> lg{mx};  std::lock_guard lg{mx};
std::vector<int> v{1, 2, 3};         std::vector v{1, 2, 3};
```

---

## 9.1 Use of Class Template Argument Deduction (p.77)

### What gets deduced (book examples)

Typical library patterns from Josuttis:

| Code snippet | Deduced specialization (conceptual) |
|--------------|-------------------------------------|
| `std::complex c{5.1, 3.3};` | `std::complex<double>` (floating literals promote inference to `double`) |
| `std::lock_guard lg{mx};` | `std::lock_guard<std::mutex>` when `mx` is a `std::mutex` |
| `std::vector v1{1, 2, 3};` | `std::vector<int>` |

**Initialization forms:** CTAD applies across the usual initialization syntax families that create an object of class type, including **braced initialization**, **parenthesized direct initialization**, and **copy initialization** where the syntax leads to constructor resolution for the class template. The book walks through that “all init methods” angle because users often mentally associate CTAD only with `{}`.

Example sketch (verify with your standard library and warning levels):

```cpp
#include <complex>
#include <vector>
#include <mutex>

void demo_init_forms() {
  std::mutex mx;

  // Braces (direct-list)
  std::complex ca{5.1, 3.3};

  // Parens (direct init) — generally OK for these types
  std::complex cb(5.1, 3.3);

  // Copy initialization — also participates where it calls a constructor
  std::complex cc = {5.1, 3.3};

  std::lock_guard lg{mx};
  std::vector v1{1, 2, 3};
}
```

### Non-type template parameters (p.77-78)

When a class template has both **type** and **non-type** parameters, deduction can fill **both** from a constructor call:

```cpp
template<typename T, int SZ>
class MyClass {
 public:
  MyClass(const T (&)[SZ]) {}
};

void demo() {
  MyClass mc("hello");  // deduces T as const char, SZ as 6 (includes null terminator array extent)
}
```

**Reading guide:** The array parameter `const T (&)[SZ]` ties `SZ` to the bound of the array passed. String literals have type `const char[N]` including the terminating `'\0'`, which is why `SZ` becomes `6` for `"hello"`.

### Variadic class templates (p.78)

```cpp
#include <tuple>

void demo_tuple() {
  std::tuple t{42, 'x', nullptr};
  // Deduced roughly as std::tuple<int, char, std::nullptr_t>
}
```

The book uses this to illustrate that deduction propagates through **multiple** constructor parameters into **multiple** class template parameters at once.

---

## 9.1.1 Copying by Default (p.79)

### The core pitfall: `vector` of vectors vs copy

```cpp
#include <vector>

void demo_vector_copy_vs_nested() {
  std::vector v1{1, 2, 3};        // std::vector<int>

  std::vector v2{v1};             // COPY: std::vector<int> (same element type as v1)
                                  // NOT std::vector<std::vector<int>>

  std::vector vv{v1, v2};         // Multiple vectors -> elements ARE vectors
                                // std::vector<std::vector<int>>
}
```

**ASCII diagram: “one vector arg vs many vector args”**

```
Case A: std::vector v2 { v1 }
─────────────────────────────

    v1  ──►  [1, 2, 3]     (element type: int)
                      │
                      │  constructor sees:
                      │  ONE argument of type vector<int>
                      │
                      ▼
               ┌──────────────────────────────┐
               │ Target: std::vector<???>     │
               │ Element type deduced as int  │
               │ (copy construction from v1)  │
               └──────────────────────────────┘


Case B: std::vector vv { v1, v2 }
─────────────────────────────────

    v1  ──►  vector<int> ─┐
    v2  ──►  vector<int> ─┼──► TWO arguments, same type: vector<int>
                          │
                          ▼
               ┌──────────────────────────────┐
               │ Target: std::vector<???>     │
               │ Element type deduced as      │
               │   std::vector<int>           │
               └──────────────────────────────┘
```

**Takeaway (book):** When you pass **one** `std::vector<T>` where a `std::vector` is being constructed, do not assume a nested vector; the default story is “**container of `T`**”, and a lone container argument often means “**copy**”.

---

## 9.1.2 Deducing the Type of Lambdas (p.80)

Lambdas produce **unique closure types**. CTAD can capture that unique type as a template argument of a wrapper class.

### CountCalls pattern

The book presents a class that wraps a callable and counts invocations (see **p.80** for the full version). Conceptually:

```cpp
#include <utility>

template<typename Callable>
class CountCalls {
 private:
  Callable c;
  long calls = 0;
 public:
  CountCalls(Callable c) : c(std::move(c)) {}

  template<typename... Args>
  decltype(auto) operator()(Args&&... args) {
    ++calls;
    return c(std::forward<Args>(args)...);
  }

  long count() const { return calls; }
};

#include <vector>
#include <algorithm>

void demo_lambda_sort() {
  std::vector<int> v{5, 42, 1};

  CountCalls sc{[](auto x, auto y) { return x > y; }};

  // Important: std::sort takes the comparator by UNQUALIFIED value type;
  // passing the wrapper BY REFERENCE avoids copying the lambda-like callable
  std::sort(v.begin(), v.end(), std::ref(sc));

  // std::for_each may return a copy of the function object depending on signature;
  // the book contrasts “pass by ref for sort” vs “for_each returns copy” nuances.
  std::for_each(v.begin(), v.end(), sc);
}
```

**Practical reading:**

- For algorithms that take comparators **by value** internally, wrapping a lambda in heavy objects may imply **copies** unless you use `std::ref` / design choices discussed in the book.
- The **deduced** type of `sc` includes the **unique lambda type** `Callable`.

---

## 9.1.3 No Partial Class Template Argument Deduction (p.81)

### “All or nothing” for template arguments

You **cannot** deduce **only some** class template parameters while explicitly specifying others in the same place as CTAD, the way you might intuitively want for function templates.

```cpp
template<typename T, typename U>
class C {
 public:
  C(T, U) {}
};

void errors() {
  // C<std::string> c4("hi", "my");  // [ERROR]: partial explicit args + deduction
  // C<> c5(22, 44.3);                // [ERROR]: even empty <> does not enable partial CTAD
}
```

### Why: ambiguities with variadic templates (p.81)

The book explains ambiguity scenarios such as `std::tuple`-like templates where specifying some parameters while deducing others collides with **variadic constructor** shapes.

Sketch:

```cpp
// Conceptual problem sketch (tuple-like):
// std::tuple<int> t(42, 43); // how many elements? what is deduced vs fixed?
```

### Lambdas and `std::set` (p.81)

You still typically **cannot** plug a **lambda** directly as a template parameter for `std::set<T, Compare>` because the lambda type is not named unless you lift it via `decltype` / template parameters / wrapper patterns; **CTAD does not magically solve** the “anonymous closure type as comparator” ergonomics issue in the same way as other templates.

---

## 9.1.4 CTAD Instead of Convenience Functions (p.83)

### `std::pair` without `make_pair`

```cpp
#include <vector>
#include <utility>

void demo_pair_ctad(std::vector<int>& v) {
  std::pair p(v.begin(), v.end());   // Often replaces std::make_pair for iterator pairs
}
```

### Decay vs reference parameters: why `make_pair` still matters (p.83)

`make_pair` **decays** arguments (arrays to pointers, function types to function pointers, stripping top-level const in the value category ways described in the book). CTAD follows template deduction rules for constructors: **if the constructor takes references**, you may deduce **array types** instead of pointers, which can be surprising or **ill-formed** if the resulting `pair` type would store references to incompatible temporaries or mismatched cv extents.

Illustrative pattern from the narrative:

```cpp
template<typename T, typename U>
struct Pair1 {
  Pair1(const T&, const U&);
};

template<typename T, typename U>
struct Pair2 {
  Pair2(T, U);
};

void demo_decay_stories() {
  // Pair1 p1{"hi", "world"}; // Often [ERROR] / undesirable: char[N] vs char[M] mismatch story
  Pair2 p2{"hi", "world"};    // [OK] typically: decay to const char* / pointer-ish types
}
```

**Reading guide:** This is the “CTAD is not a drop-in semantic replacement for every factory” lesson.

---

## 9.2 Deduction Guides (p.84)

Guides are **explicit instructions** that tell the compiler how to map constructor parameter patterns to **class template specializations**.

Syntax pattern:

```cpp
template<typename T1, typename T2>
class Pair3 {
 public:
  Pair3(T1, T2) {}
};

// Deduction guide: left side pattern -> right side invented type
template<typename T1, typename T2>
Pair3(T1, T2) -> Pair3<T1, T2>;
```

### Resolution flow (ASCII)

```
User writes:  Pair3 p(a, b);
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│ 1) Collect candidate guides + constructors                 │
│    - guides: template patterns with "-> ResultType"        │
│    - ctor templates may also deduce without a guide        │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ? 2) Build candidates / reconcile matches ()
                            │
                            ▼
┌────────────────────────────────────────────────────────────┐
│ 3) Pick best via overload resolution-like rules            │
│    If guide ties: see 9.2.3 (guide preferences)            │
└───────────────────────────┬────────────────────────────────┘
                            │
                            ▼
                     Deduced: Pair3<...>
```

**Book emphasis:** The **right-hand side** of `->` is the **synthesized class template id** with concrete arguments.

---

## 9.2.1 Force Decay (p.85)

If a constructor template would deduce “ awkward” reference/array types, a guide taking parameters **by value** can force **decay**, matching the intuition of factory functions.

```cpp
template<typename T>
class C {
 public:
  C(T) {}
};

template<typename T>
C(T) -> C<T>;  // Taking T by value in the guide/ctor pairing can decay similarly to ctor(T)
```

(Read **p.85** for the precise mechanics and what happens if ctor signature differs.)

---

## 9.2.2 Non-Template Deduction Guides (p.86)

A deduction guide need not introduce a new template: non-template guides apply to specific constructor shapes.

Typical library narrative example: a class template plus a **non-template** guide that maps `const char*` construction to `std::string` as the stored type parameter.

```cpp
#include <string>

template<typename T>
class S {
 public:
  explicit S(T v) : value(std::move(v)) {}
  const T& get() const { return value; }

 private:
  T value;
};

// Non-template deduction guide (p.86): note the arrow to a concrete specialization
S(const char*) -> S<std::string>;

void demoNonTemplateGuide() {
  S s{"hello"};  // Deduces S<std::string> via the guide + decay story on p.86
  (void)s.get();
}
```

---

## 9.2.3 Deduction Guides vs Constructors (p.86)

When **both** a constructor template and a deduction guide could deduce the same pattern with equal “match quality”, the standard prioritizes **deduction guides** in the tie-break sense described on **p.86**.

---

## 9.2.4 Explicit Deduction Guides (p.87)

Guides can be marked `explicit`, mirroring `explicit` constructor effects: **copy initialization** contexts may reject implicit guide-based deductions.

```cpp
// Sketch (see book for compile-fail examples with '=' initialization)
template<typename T>
struct E {
  explicit E(const T&) {}
};

// explicit E(const char*) -> E<std::string>;  // example narrative on p.87
```

---

## 9.2.5 Deduction Guides for Aggregates (p.88)

Aggregates may not have “real” constructors, but you can still define guides to teach CTAD how to build the type.

```cpp
template<typename T>
struct A {
  T name;
};

A(const char*) -> A<std::string>;  // Aggregate + guide story (p.88)
```

---

## 9.2.6 Standard Deduction Guides (p.89)

### Pairs and tuples

Standard guides generally incorporate **decay** behaviors analogous to historical `make_pair` / tuple factories.

### Iterators and `std::vector` (p.89): the initializer_list trap

There is a guide that maps an iterator pair to `vector` of `std::iterator_traits<Iterator>::value_type`.

**BEWARE (book warning):**

```cpp
#include <string>
#include <vector>

void trap(std::string s) {
  // Many expect iterator CTAD here:
  std::vector v2{s.begin(), s.end()};
  // But this often hits std::initializer_list<typename> deduction instead
  // (not the iterator-pair guide), producing a surprising vector<T> type.
}
```

ASCII warning:

```
You wrote:  std::vector v { s.begin(), s.end() };
                │
                ├─ (expected) iterator guide -> vector<char> (for std::string)
                │
                └─ (often actual) -> initializer_list<iterator>
                                    vector of iterators (unexpected!)
```

### `std::array` (p.89)

`std::array` deduction uses variadic tricks (fold / `enable_if` patterns in the standard) to ensure **homogeneous** element types.

### `std::map` / `std::multimap` deduction (p.89)

Early C++17 had deduction pitfalls around **const key types**; **LWG 3025** is cited in the book as the library fix trajectory.

### Smart pointers (p.89)

No CTAD guides for `std::unique_ptr` / `std::shared_ptr` from raw pointers in the standard due to **array allocation ambiguity** (`new T` vs `new T[n]`).

---

## Quick “mental checklist” (chapter summary)

1. CTAD deduces from **constructors**; understand **recursive copy** vs **nested vectors** (**p.79**).
2. **Partial** explicit template arguments do not compose with CTAD the way you might want (**p.81**).
3. Prefer guides when you need `make_*`-style **decay** semantics (**p.83-85**).
4. Library guides have sharp edges (`vector` + `{it,it}`, `map` key const, no smart-pointer guides) (**p.89**).

---

## Suggested exercises (optional)

1. Reproduce the `std::vector{s.begin(), s.end()}` trap with `static_assert` or compiler error messages; compare to `std::vector<char>(s.begin(), s.end())`.
2. Implement `Pair3` with and without a deduction guide; compare `Pair3{"a","bc"}` deduced types when constructors take `(T,U)` vs `(const T&, const U&)`.
3. Write `CountCalls` and verify `std::sort(..., std::ref(sc))` vs passing `sc` by value.

---

**Primary reference:** Josuttis, *C++17 - The Complete Guide*, Chapter 9, **pp. 77-93**.
