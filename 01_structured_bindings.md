# Chapter 1: Structured Bindings

**Book:** *C++17 - The Complete Guide*, first edition, Nicolai M. Josuttis (published 2019-12-20, 452 pages)  
**Chapter:** 1 - Structured Bindings  
**Book page span:** approximately **pp. 3-19** (section numbering below mirrors the book: **1.1**, **1.2**, and so on)

This tutorial is **self-contained**: you can study it without the book, but **page references** point to the book for deeper commentary and additional edge cases.

---

## Why structured bindings exist

Before C++17, extracting multiple names from a composite value (struct, pair, tuple, array) required either verbose member access or `std::tie`. Structured bindings let you **declare multiple names** in one statement while keeping type deduction ergonomic and (often) const-correct.

```
Before (C++14 style with a pair)              After (C++17)
───────────────────────────────────          ─────────────────────────────
auto result = m.insert({"key", 42});         auto [pos, ok] = m.insert({"key", 42});
auto pos = result.first;                     // pos and ok are direct names
bool ok = result.second;
```

---

## 1.1 Structured Bindings in Detail (book p.4)

### 1.1.1 Hidden anonymous entity (core semantics)

A structured binding **always** introduces a **backing object** (an unnamed variable in the compiler's model) initialized from the right-hand side. The bindings (`u`, `v`, etc.) are **aliases** to members or elements **of that backing object** (or pseudo-members via the tuple-like protocol), not independent variables that directly rebind to members of some pre-existing object.

Conceptual model for:

```cpp
auto [u, v] = ms;
```

When `ms` is a struct with members `i` and `s`, the book explains that the construct behaves **as if** you first materialize a fresh object `e` initialized from `ms`, then introduce names that alias members accessed through that object.

ASCII picture:

```
Right-hand side    Backing object "e"      Bindings (names you use)
────────────────   ────────────────────     ────────────────────────

    ms      ──►    ┌─────────────────┐
                   │  e.i   e.s      │
                   └────────┬────────┘
                            │
                    u  ─────┼── alias view into e.i   (not a reference variable)
                    v  ──────── alias view into e.s
```

**Critical fine point (book):**  
`u` and `v` are **not** `decltype(ms.i)`-style references unless you wrote reference qualifiers on the **whole** structured binding (see below). The bindings introduce names with types corresponding to the **members' types** (possibly const-qualified through the anonymous entity's qualifiers). The standard wording is subtle; Josuttis emphasizes that imagining `aliasname u = e.i` captures the **name lookup / meaning**, but the **types** follow the structured binding rules (`decltype(u)` tracks the member's type, not a reference type, unless you forced references via `auto&` / `auto&&` on the binding declaration).

### 1.1.2 `decltype` on bindings

For:

```cpp
struct MS { int i; std::string s; };
MS ms{42, "hi"};
auto [u, v] = ms;
```

**Expectation from the book's discussion:**  
`decltype(u)` is `int`, not `int&`. The binding name refers to the `int` member **as accessed through** the structured binding mechanism, but it does not behave as if you had written `int& u = ms.i` in the general case.

### 1.1.3 Using qualifiers: `const auto&`, `auto&`, `auto&&`

Qualifiers apply to the **anonymous entity** initialization, not "each binding" in isolation.

Example patterns:

```cpp
const auto& [a, b] = expr; // backing object is const lvalue binding to whatever expr denotes
auto&       [a, b] = expr; // backing entity binds as an lvalue reference to the referent
auto&&      [a, b] = expr; // backing entity is a forwarding reference initialized from expr
```

ASCII:

```
Declaration              Effect on backing object          Typical effect on reads/writes
─────────────────────────────────────────────────────────────────────────────────────────
const auto& [x,y] = o;   const lvalue ref to o           x,y read-only through binding
auto& [x,y] = o;        lvalue ref to o                 x,y can modify members of o
auto&& [x,y] = std::move(o);  rvalue ref init           move-like access patterns possible
```

### 1.1.4 `alignas` interacts with the backing object

From the book's note: **`alignas(16) auto [u, v] = ms;` aligns the anonymous entity**, not "later" bindings like `v` individually. If you need member-level alignment, that belongs on the class definition.

### 1.1.5 No decay for arrays (contrast with plain `auto`)

Plain `auto` **decays** arrays to pointers. Structured bindings **do not** decay array bindings the same way:

```cpp
char a[6] = "hello";
auto x = a;            // x is char*
auto [b0, b1, b2, b3, b4, b5] = a; // six char bindings (copied from elements)
```

In the array case, the backing object behaves like a **copy** of the array (see 1.2).

### 1.1.6 Move semantics: `auto` vs `auto&&`

```cpp
struct MS { int i; std::string s; };
MS ms{42, "hello"};

auto [v, n] = std::move(ms);
// Commonly: n binds to a moved-from string inside the backing object copy/move;
// ms may be left moved-from depending on how bindings initialize.

auto&& [v2, n2] = std::move(ms2);
// Forwards/move semantics through the reference binding of the backing entity.
```

**Practical guidance:** If you need to **consume** expensive members from an rvalue, use `auto&&` with `std::move` on the source, mirroring forwarding-reference style. If you want a **copy** into a new anonymous object, use `auto`.

Mini diagram for value categories:

```
expr is lvalue                auto [a,b] = expr      copies / binds per rules
expr is prvalue               auto [a,b] = expr      materializes temporary T, binds to members/elements
std::move(lv) is xvalue       auto&& [a,b] = ...     treat like forwarding ref to xvalue
```

---

## 1.2 Where Structured Bindings Can Be Used (book p.7)

### 1.2.1 Structures and classes (public non-static data members)

Structured bindings decompose **non-static data members that are public** and visible. Private members **cannot** be bound unless you use the **tuple-like customization** (1.3).

### 1.2.2 Inheritance restriction (book example)

All bound members must come from **one class definition's direct members** (the book's point: you cannot mix a base subobject's members into the same binding list across an **extension** in the derived class in the broken way below).

**OK (derived adds no new members):**

```cpp
struct B {
  int a = 1;
  int b = 2;
};

struct D1 : B { };

void demo_ok() {
  auto [x, y] = D1{};
  // x binds to B::a, y binds to B::b
}
```

**ERROR (derived adds member `c`; binding three names is not allowed in this pattern):**

```cpp
struct D2 : B {
  int c = 3;
};

void demo_error() {
  // auto [i, j, k] = D2{}; // compile error (book illustration)
}
```

ASCII:

```
┌─────────────────────────────────────────────────────────────┐
│ Binding `[..]` to class type T                              │
│ Requires: decomposition follows ONE class's member list     │
│ (Josuttis: not a flat mix across derived-added layout)      │
└─────────────────────────────────────────────────────────────┘
```

If you need a flat view across hierarchy, use accessors or tuple-like `get`.

### 1.2.3 Raw arrays (copy by value)

```cpp
int arr[] = {47, 11};
auto [x, y] = arr; // copies the array into backing object (rare C++ array copy)
```

This is one of the few places C++ **copies** a raw array by value through initialization machinery.

### 1.2.4 `std::pair`, `std::tuple`, `std::array` (extensible mechanism)

**`std::array` by reference (modify original):**

```cpp
#include <array>
std::array<int, 4> stdarr{1, 2, 3, 4};
auto& [a, b, c, d] = stdarr;
b = 99; // mutates stdarr[1]
```

**Tuple from factory:**

```cpp
#include <tuple>
std::tuple<char, float, std::string> getTuple();

void use() {
  auto [a, b, c] = getTuple();
}
```

**`map::insert` result:**

```cpp
#include <map>
#include <string>
void example_insert(std::map<std::string, int>& coll) {
  auto [pos, ok] = coll.insert({"new", 42});
  if (!ok) {
    // insertion failed: element exists
    const auto& key = pos->first;
    (void)key;
  }
}
```

### 1.2.5 Iterating maps with structured bindings (realistic idiom)

```cpp
#include <iostream>
#include <map>
#include <string>

void print_map(const std::map<std::string, int>& m) {
  for (const auto& [key, val] : m) {
    std::cout << key << " -> " << val << '\n';
  }
}
```

**Lifetime note:** The range-for with `const auto& [key, val]` binds to the pair type produced by the map iterator's `operator*`. Keep iterations **local**; do not store pointers to `key`/`val` beyond the iterator validity unless you extend lifetime explicitly.

### 1.2.6 Assigning into existing objects with `std::tie`

Bindings declare **new names**; they do not reassign symbols later. To **write into existing variables**, use **`std::tie`**:

```cpp
#include <tuple>
void tie_example() {
  int a = 0, b = 0;
  std::tie(a, b) = std::make_pair(3, 4);
}
```

Use structured bindings for **readability on fresh extraction**; use **`tie` for out-parameters** into pre-existing locals.

---

## 1.3 Providing a Tuple-Like API for Structured Bindings (book p.11)

This section mirrors Josuttis's **`Customer`** example: **private data**, public tuple-like interface.

### 1.3.1 Goals

- Allow `auto [first, last, val] = c;` for `Customer c`.
- Support **read-only** access for `const Customer&`.
- Support **read/write** through `Customer&` and `Customer&&` as appropriate.

### 1.3.2 Required protocol pieces

1. `std::tuple_size<Customer>::value == 3`
2. `std::tuple_element<Idx, Customer>` for `Idx in {0,1,2}`
3. **`get<Idx>(...)` overload sets** for:
   - `const Customer&`
   - `Customer&`
   - `Customer&&`
4. Implement **`get`** with `if constexpr` on index for clarity.

### 1.3.3 Complete example (educational, book-style)

This compiles as C++17 and follows the book's pattern: **private members**, **`get` with `if constexpr`**, three overload sets for **`const&` / `&` / `&&`**, and `tuple_element` tied to what **`get`** returns.

```cpp
#include <tuple>
#include <string>
#include <utility>
#include <iostream>
#include <type_traits>

class Customer {
 private:
  std::string first;
  std::string last;
  long val;

 public:
  Customer(std::string f, std::string l, long v)
  : first{std::move(f)}, last{std::move(l)}, val{v} {}

  // Ordinary accessors remain useful for non-binding callers:
  const std::string& firstName() const { return first; }
  const std::string& lastName()  const { return last; }
  long               value()     const { return val; }

  template<std::size_t I>
  friend constexpr decltype(auto) get(const Customer& c) {
    if constexpr (I == 0) return (c.first);
    else if constexpr (I == 1) return (c.last);
    else return (c.val);
  }

  template<std::size_t I>
  friend constexpr decltype(auto) get(Customer& c) {
    if constexpr (I == 0) return (c.first);
    else if constexpr (I == 1) return (c.last);
    else return (c.val);
  }

  template<std::size_t I>
  friend constexpr decltype(auto) get(Customer&& c) {
    if constexpr (I == 0) return std::move(c.first);
    else if constexpr (I == 1) return std::move(c.last);
    else return std::move(c.val);
  }
};

template<>
struct std::tuple_size<Customer> : std::integral_constant<std::size_t, 3> {};

template<std::size_t I>
struct std::tuple_element<I, Customer> {
  // Strip references so tuple_element names the "element object type";
  // the bindings still bind according to the get overload and the structured-binding rules.
  using type = std::remove_reference_t<decltype(get<I>(std::declval<Customer&>()))>;
};

int main() {
  Customer c{"Tim", "SuperDict", 42};

  auto [f, l, v] = c;
  std::cout << "copy-bind: " << f << ' ' << l << ' ' << v << "\n";

  auto& [f2, l2, v2] = c;
  f2 = "Tom";
  v2 = 77;
  std::cout << "mutated: " << c.firstName() << ' ' << c.lastName() << ' ' << c.value() << "\n";
}
```

**Why `decltype(auto)` and `return (member)`:** Returning `(c.first)` yields an **lvalue reference** type in `decltype(auto)`; parenthesized `return (expr)` preserves a deliberate **lvalue** category in some idioms. **`std::move(c.first)`** in the `Customer&&` overload models **move-return** for rvalues (book: three overload sets for write access patterns).

### 1.3.4 Tuple-like API structure (diagram)

```
Customer
────────
 private:
   first, last, val
 public:
   normal methods
             │
             ▼
   Customization point in namespace std (specializations)
   ┌─────────────────────────────────────────────────────┐
   │ tuple_size<Customer>                                │
   │ tuple_element<I, Customer>                          │
   └───────────────┬─────────────────────────────────────┘
                   │
                   ▼
   Friend get<I>(Customer&) / const& / &&  (ADL finds get in Customer's scope)
   ┌─────────────────────────────────────────────────────┐
   │ if constexpr (I == 0) -> first                      │
   │ if constexpr (I == 1) -> last                       │
   │ if constexpr (I == 2) -> val                        │
   └─────────────────────────────────────────────────────┘
                   │
                   ▼
   Compiler lowers: auto [a,b,g] = customer;
                    into get<0>, get<1>, get<2> calls
```

---

## 1.4 Afternotes (book p.19)

Structured bindings originated from proposals associated with **Herb Sutter**, **Bjarne Stroustrup**, and **Gabriel Dos Reis**. As always, read the book's closing notes for historical context and committee rationale; standards evolve, but the C++17 behavior is stable.

---

## Quick pitfalls checklist

| Topic | Pitfall | Safer pattern |
|------|---------|----------------|
| Map iteration | Storing `&key` across mutations | Keep bindings inside loop body |
| `auto [..] = map_iter` | Confusion about pair type | Remember iterator `value_type` is `pair<const K, T>` |
| Private members | Cannot bind directly | Tuple-like API (1.3) |
| Arrays | Accidental copies | Bind by `auto&` if you need original elements |
| Inheritance | Multi-level flat binding | Use accessors / `get` |

---

## Further reading map

- Book **pp. 3-6**: precise standardese mapping and qualifier examples.  
- Book **pp. 7-10**: pairs, tuples, **insert**, and **tie**.  
- Book **pp. 11-18**: detailed `Customer` customization and commentary.  
- Book **p. 19**: proposal/authorship notes.

---

## See also

- `00_index.md` - master table of contents with page ranges for all parts.  
- `02_if_switch_init.md` - combining structured bindings with `if` init statements.  

Reference: Nicolai M. Josuttis, *C++17 - The Complete Guide* (first edition, 2019-12-20).
