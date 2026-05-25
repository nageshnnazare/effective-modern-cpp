# Chapter 13: Placeholder Types like `auto` as Template Parameters

**Book:** *C++17 - The Complete Guide*, Nicolai M. Josuttis  
**Chapter:** 13 - Placeholder Types like `auto` as Template Parameters  
**Book page span:** **pp. 121-127** (sections **13.1**-**13.3**)

C++17 lets template parameters be deduced “non-types” using **`auto`** (and refined with **`decltype(auto)`** in **13.3**), which unifies NTTP syntax and enables cleaner metaprogramming for constants of many scalar kinds.

---

## 13.1 Using `auto` for Template Parameters (p.121)

### Basic pattern

```cpp
template<auto N>
struct S {};

void demoBasic() {
  S<42> s1;   // N deduced as int (book examples)
  S<'a'> s2;  // N deduced as char

  // S<2.5> s3;  // [ERROR] typically: floating-point NTTP not allowed here (pre-C++20 float NTTP rules)
}
```

**Book note:** Floating literals as NTTP were not the C++17 story; `2.5` in `S<2.5>` is the textbook **ERROR** example on **p.121**.

### Partial specialization still names the underlying value

```cpp
template<auto N>
class S {};

template<int N>
class S<N> {  // partial specialization when N’s type is int
 public:
  static constexpr int kind = 1;
};
```

### CTAD interplay with class templates that have `auto` parameters (p.121)

If a class template mixes type and `auto` non-type parameters, constructor-driven CTAD can deduce both **types** and **values** in some designs (**p.121**, ties back to Chapter 9).

Sketch:

```cpp
template<typename T, auto N>
class A {
 public:
  A(T (&)[N]) {}  // illustrative: array extent + element type
};
```

### Qualified `auto` (p.121)

You can constrain placeholder kinds:

```cpp
template<const auto* P>
struct PtrParam {};
```

Meaning: the non-type parameter is a pointer-ish `auto` deduced placeholder with `const` on the deduced type side as appropriate (follow book wording).

### Heterogeneous value lists (p.121)

```cpp
template<auto... VS>
class HeteroValueList {};

void demoHetero() {
  HeteroValueList<1, 'x', true> hv;
  (void)hv;
}
```

### Homogeneous value lists pinning first type (p.121)

```cpp
template<auto V1, decltype(V1)... Vs>
class HomoValueList {};

void demoHomo() {
  HomoValueList<1, 2, 3> ok;
  // HomoValueList<1, 2, 3u> bad; // typically [ERROR]: 3u not same type as int V1 path
  (void)ok;
}
```

**Reading guide:** The second parameter pack **repeats** `decltype(V1)` forcing a single scalar “family”.

---

## 13.1.1 Characters and Strings (p.122)

### Default non-type parameter for formatting

```cpp
#include <iostream>

template<auto Sep = ' '>
void print() {}

// Overload set typically pairs Sep with variadic printing; book uses a pattern like:
template<auto Sep = ' ', typename T, typename... Rest>
void print(const T& x, const Rest&... rest) {
  std::cout << x;
  ((std::cout << Sep << rest), ...);
}

void demoSep() {
  std::string s = "world";
  print<'-'>(7.5, "hello", s);  // book-style narrative: 7.5-hello-world
}
```

### Using `static const char sep[]` as NTTP (p.122)

```cpp
static const char sep[] = ", ";

template<const char* S, typename T, typename... Rest>
void printSep(const T& x, const Rest&... rest) {
  std::cout << x;
  ((std::cout << S << rest), ...);
}
```

This illustrates lifting string data into named arrays so NTTP pointers refer to stable storage.

---

## 13.1.2 Metaprogramming Constants (p.123)

### The `constant` wrapper idiom

```cpp
template<auto v>
struct constant {
  static constexpr auto value = v;
};

using i = constant<42>;
using c = constant<'x'>;

static_assert(i::value == 42);
static_assert(c::value == 'x');
```

### Heterogeneous sequences

```cpp
template<auto... Vs>
struct sequence {};

using tupleLike = sequence<0, 'h', true>;
```

These become building blocks for compile-time data structures and algorithm dispatch.

---

## 13.2 Variable Template Parameter (p.124)

### `std::array` alias with `auto` extent

```cpp
#include <array>

template<typename T, auto N>
std::array<T, N> arr{};

void demoArrayAlias() {
  arr<int, 5>;
  arr<int, 5u>;  // book: distinguishes types of NTTP -> different objects/templates
}
```

**Book emphasis:** `5` vs `5u` may instantiate **distinct** variable templates even when extents “mean the same size” at runtime.

### Variable template shorthand (p.124)

```cpp
template<auto N>
constexpr auto val = N;

constexpr auto fortyTwo = val<42>;
```

Use this to propagate constants through generic code with less `typename` noise than type-only approaches.

---

## 13.3 `decltype(auto)` as Template Parameter (p.126)

This section explores that **`auto` vs `decltype(auto)`** distinction for **NTTP** matches the same subtleties as in variable declarations: **value vs reference** kinds matter.

### Setup: `const int` lvalue constant

```cpp
static const int c = 42;

template<decltype(auto) N>
struct S {
  static void printN();
};
```

### `S<c>` vs `S<(c)>` vs `S<(v)>`

**Book patterns:**

- `S<c>` deduces **by name idiom** leading to `const int` value semantics in the narrative.
- `S<(c)>` treats `(c)` as an **lvalue expression**, deducing **reference** kind: `const int&` referring to `c`.
- For a non-const `int v`, `S<(v)>` may deduce `int&`, and mutating `v` later affects what member functions observe.

Illustrative sketch:

```cpp
int v = 42;

template<decltype(auto) N>
struct S2 {
  static void printN() {
    // std::cout << N;
  }
};

void demoDecltypeAuto() {
  // S2<c>     ...
  // S2<(c)>   ...
  // S2<(v)>   ...
  v = 77;
  // S2<(v)>::printN(); // book narrative: prints 77 after mutation
}
```

**ASCII reference summary:**

```
┌─────────────────────┬──────────────────────────────┬────────────────────────────┐
│ Template arg        │ Typical deduction story      │ What mutating source does  │
├─────────────────────┼──────────────────────────────┼────────────────────────────┤
│ c   (const int obj) │ const int value NTTP         │ source immutable           │
│ (c) (lvalue expr)   │ const int& NTTP to c         │ still bound to c           │
│ (v) (mutable lvalue)│ int& NTTP to v               │ affects via reference      │
└─────────────────────┴──────────────────────────────┴────────────────────────────┘
```

**Why this matters:** metaprogramming sometimes wants **by-value copies** of constants; other times you intentionally want **references** into translation-unit objects for testing or advanced protocols — `decltype(auto)` exposes that choice explicitly.

---

## Practical guidance

1. Prefer `auto` NTTP for **integral/enum/pointer** constants; know the **float** limitations in C++17 (**p.121**).
2. Use **heterogeneous** packs for tuples-like lists; use **`decltype(V1)...`** to **pin** homogeneity (**p.121**).
3. Remember `5` vs `5u` can split instantiations for `template<auto N>` variable templates (**p.124**).
4. Treat `decltype(auto)` NTTP like ordinary `decltype(auto)` deduction: **parentheses create expressions** (**p.126**).

---

**Primary reference:** Josuttis, *C++17 - The Complete Guide*, Chapter 13, **pp. 121-127**.
