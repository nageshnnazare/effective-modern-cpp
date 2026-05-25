# Chapter 16: `std::variant<>` (The Complete Guide, pp. 149-168)

This chapter follows *C++17 - The Complete Guide* by Nicolai M. Josuttis. Page references use **p.**

---

## 16.1 Motivation (p. 149): type-safe union

A C `union` holds one of several alternatives but **does not track** which member is active; reading the wrong member is undefined behavior. `std::variant<Alternatives...>` is a **tagged union**: it stores **one** of the types and remembers **which** one (the **discriminator** / **index**).

**Goals** (as Josuttis presents them):

- Type-safe access (`std::get`, `std::get_if`, `std::visit`).
- Value semantics (stack storage, no mandatory heap).
- Suitable for **closed** sets of types known at compile time.

---

## 16.2 Using `std::variant<>` (p. 150)

### Construction and active alternative

```cpp
#include <variant>
#include <string>

std::variant<int, std::string> var{"hello"};
// Active alternative is std::string if overload resolution picks it.
```

### Construction ambiguities (p. 154-155)

Overload resolution for the converting constructor can pick exactly one best-matching alternative. When no unique best match exists, the program is ill-formed.

```cpp
std::variant<long, int> v2{42};
// [OK] index() == 1 — int is the better match than long for the int literal

std::variant<long, long> v3{42};
// [ERROR] ambiguous: both alternatives match equally well

std::variant<int, float> v4{42.3};
// [ERROR] ambiguous between int and float for the double literal

std::variant<int, double> v5{42.3};
// [OK] double literal unambiguously selects double

std::variant<std::string, std::string_view> v7{"hello"};
// [ERROR] ambiguous which string-like alternative is best

std::variant<std::string, std::string_view, const char*> v8{"hello"};
// [OK] const char* is the best match; index() == 2
```

Josuttis emphasizes that **conversion sequences** and **ranking** determine which alternative becomes active—not source order alone.

### `std::in_place_type` and `std::in_place_index` (p. 154)

When overload resolution is unclear—or you must construct a specific alternative explicitly—use **in-place tags** so the variant knows **which** member to construct and with **which** constructor arguments.

```cpp
#include <complex>
#include <vector>
#include <set>

std::variant<std::complex<double>> v11{
    std::in_place_type<std::complex<double>>, 3.0, 4.0
};
// Constructs complex from (3.0, 4.0) in the single alternative

std::variant<int, int> v13{std::in_place_index<1>, 77};
// [OK] Initializes the *second* int (index 1), not the first

// With initializer_list into a specific alternative (e.g. set with comparator):
struct SomeComp {
    bool operator()(int a, int b) const { return a < b; }
} sc;
std::variant<std::vector<int>, std::set<int, decltype(sc)>> v15{
    std::in_place_index<1>, {4, 8, -7}, sc
};
// Brace list + comparator forwarded to std::set's constructor
```

**No CTAD for `variant`**: class template argument deduction does not apply usefully here in the way it does for containers—the alternative list must be explicit.

**No `std::make_variant`**: unlike `make_pair` / `make_tuple`, a factory like `make_variant<>` does not exist in the library; the type set is the whole point of the object, so you always name the alternatives explicitly (possibly with `in_place_*` for construction).

### Index-based and type-based `get`

```cpp
std::string& s = std::get<1>(var);           // by index (0=int, 1=string)
std::string& s2 = std::get<std::string>(var);
```

### `index()` (p. 150)

```cpp
assert(var.index() == 1);  // 1 means second type (string) — 0-based
```

### `holds_alternative` (p. 151)

```cpp
if (std::holds_alternative<std::string>(var)) {
    // ...
}
```

### `std::get` throws `std::bad_variant_access` (p. 151)

```cpp
var = 42;  // now holds int
try {
    std::get<std::string>(var);  // [ERROR] wrong active type
} catch (const std::bad_variant_access&) {
    // handle
}
```

### `std::get_if`: pointer or `nullptr` (p. 151)

```cpp
if (auto* p = std::get_if<std::string>(&var)) {
    // use *p
}
```

Non-throwing inspection at the cost of pointer checks.

---

## 16.3 Types and operations (p. 152)

### Default construction: first alternative (p. 152)

Default-constructed `std::variant<T1, T2, ...>` default-constructs **`T1`**. Therefore **`T1` must be default-constructible** unless you use a workaround.

### `std::monostate` as first type (p. 153)

If the "real" first type is not default-constructible, prepend `std::monostate`:

```cpp
#include <variant>

struct NoDefault {
    NoDefault(int) {}
};

std::variant<std::monostate, NoDefault, int> v;
// v holds monostate initially
```

`std::monostate` is an empty type used purely as a placeholder.

### Deriving from `std::variant` (p. 152)

`std::variant` is not `final`. You may inherit publicly to add convenience or domain-specific APIs while reusing variant storage and dispatch.

```cpp
class Derived : public std::variant<int, std::string> {};

Derived d = {{"hello"}};   // brace form initializes the base variant
d.emplace<0>(77);          // still use index-based emplace on the variant interface
```

The usual caveats apply: slicing if you pass `Derived` where a base `variant` is expected by value; prefer composition unless inheritance genuinely simplifies your design.

### Duplicate alternative types (p. 155-156)

If the same type appears more than once, **index** distinguishes the slots; **`std::get<T>` by type is ambiguous**.

```cpp
std::variant<int, int, std::string> var;   // two int slots (indices 0 and 1)

// std::get<int>(var);
// [ERROR] ambiguous at compile time: two alternatives are int

std::get<0>(var);  // [OK] first int (throws std::bad_variant_access if not active)

std::get<1>(var);  // [OK] second int — throws if the *currently active* index is not 1
```

Use **`std::get<I>(var)`** with a constant index when duplicates exist. If you request the wrong index for the active member, you get **`std::bad_variant_access`** at runtime—not a silent read from the wrong slot.

### Assignment, `emplace`, comparisons (p. 153-156)

```cpp
std::variant<int, std::string> v = 7;
v = "abc";                       // assigns active alternative
v.emplace<int>(99);              // destroys old alt, constructs int
```

Comparison operators are defined when the alternatives support comparison. Josuttis gives these **ordering rules** (p. 156-157):

1. **Different active indices**: Compare **indices** first. The alternative with the **smaller index** (listed **earlier** in `variant<...>`) orders **before** the other for `<` / `>` / `<=` / `>=`. For `==` and `!=`, different active indices mean **not equal** (values are not compared).
2. **Same active index**: Compare the **held values** using that alternative’s **own** `operator==` / relational operators.
3. **Valueless by exception** (see below): **`valueless_by_exception() == valueless_by_exception()`** is `true` for both in that state; a valueless variant compares **less than** any non-valueless variant (and not equal to any value-holding state).
4. **Different `variant` specializations**: You cannot compare `std::variant<A,B>` with `std::variant<C,D>` if they are **different types**—that is a **compile-time error**, even if `A` and `C` are the same:

```cpp
std::variant<int, double> v1{1};
std::variant<int, long> v4{1};
// v1 == v4;
// [ERROR] different variant types; no comparison operator
```

### Move semantics (p. 156)

Moves move the **currently active** object; moving-from leaves the variant in a valid state with a moved-from active object (per moved-from type's rules).

---

## 16.3.3 Visitors (p. 157)

### `std::visit(visitor, var)`

The visitor must be callable with **every** alternative type (with the value category the visit uses). The compiler generates the dispatch (effectively a jump table / cascade).

### Function object visitor (p. 157)

A **class-type visitor** provides one `operator()` overload per alternative—clear and easy to optimize.

```cpp
#include <iostream>
#include <variant>
#include <string>

struct MyVisitor {
    void operator()(int i) const { std::cout << "int: " << i << '\n'; }
    void operator()(const std::string& s) const { std::cout << "str: " << s << '\n'; }
};

std::variant<int, std::string> v = 42;
std::visit(MyVisitor{}, v);
```

### Modifying visitors (p. 157-158)

Passing a non-const variant to `visit` yields lvalue references to the active member, so the visitor can **mutate** the stored value:

```cpp
struct Twice {
    void operator()(double& d) const { d *= 2; }
    void operator()(int& i) const { i *= 2; }
    void operator()(std::string& s) const { s += s; }
};

std::variant<double, int, std::string> vv{3.14};
std::visit(Twice{}, vv);   // doubles the active double in place
```

### Generic lambda visitor (p. 158)

```cpp
std::visit([](const auto& val) { std::cout << val; }, v);
```

`val` is deduced independently for each alternative. For **different behavior per category**, combine with **`if constexpr`** and a trait that matches the **value’s types as seen through references**:

```cpp
#include <type_traits>
// ...
std::visit([](const auto& val) {
    if constexpr (std::is_convertible_v<decltype(val), std::string>) {
        // treat as string-like
    } else {
        // numeric or other
    }
}, v);
```

**Note** (Josuttis): use **`std::is_convertible_v`** (or similar) rather than **`std::is_same`** with `decltype(val)`, because `val` is a **reference** type in the lambda; you care whether the **referred-to** type behaves like a string for your logic.

### Return values from `visit` (p. 158-159)

All `operator()` overloads (or lambda bodies) that `visit` might invoke must return **the same type** (after decay), or a **common type** the implementation can deduce. If deduction fails, specify the return type explicitly:

```cpp
double r = std::visit([](const auto& x) -> double {
    return static_cast<double>(x);   // explicit -> double unifies all branches
}, v);
```

Without a consistent return type, you get compile errors when one branch returns `void` and another returns a value, or when deduction is ambiguous.

### Overload pattern / overloaded lambdas (p. 158-159)

When behavior differs a lot per alternative, combine lambdas into one functor with overloaded `operator()` and pass it to `visit`:

```cpp
template<class... Ts>
struct overload : Ts... { using Ts::operator()...; };
template<class... Ts>
overload(Ts...) -> overload<Ts...>;

std::visit(overload{
    [](int i) { /*...*/ },
    [](const std::string& s) { /*...*/ }
}, v);
```

Josuttis presents this as practical sugar: **`std::visit(overload{ ... }, var);`** reads as “dispatch to the matching lambda.”

### Multi-variant `visit` (p. 159-160)

```cpp
std::variant<int, double> a{1};
std::variant<int, double> b{2.5};
std::visit([](auto x, auto y) {
    // Cartesian dispatch over alternatives
}, a, b);
```

**Return type rule**: every overload invoked (for every combination of active alternatives) must return **the same type** (or a type convertible to one deduced result type).

---

## 16.3.4 Valueless by exception (p. 161-162)

If an operation that **changes the active alternative** throws **during** the process, the variant can end up **holding no value**: the **valueless-by-exception** state.

```cpp
if (v.valueless_by_exception()) {
    // rare; indicates failed transition
}
```

In that state, **`v.index() == std::variant_npos`** (implementation-defined constant; no valid alternative index).

### Example: throwing conversion into `emplace` (p. 161-162)

```cpp
struct S {
    operator int() { throw "EXCEPTION"; }
};

std::variant<std::string, int> vv{"hello"};
try {
    vv.emplace<1>(S{});   // attempting to construct int from S; conversion throws
} catch (...) { }

// [OOPS] vv may now be valueless_by_exception()
```

### Guarantees (Josuttis, p. 161-162)

- **`emplace`**: If an exception escapes while the variant switches or constructs the new alternative, the variant may become **valueless**; any strong guarantee depends on the exception occurring **when** relative to destroying the old state and building the new one.
- **`operator=`**: Whether you end up valueless depends on **when** the exception is thrown (e.g. during destruction of the old member vs. construction of the new). The standard specifies the exceptional cases; in practice treat **valueless** as a rare but possible outcome after a failed type-changing assignment or `emplace`.

Non-state-changing operations on an already-correct alternative typically do not induce valuelessness; the risky transitions are those that **replace** the active type.

---

## 16.4 Polymorphism with `std::variant` (p. 162)

### Geometric objects: `GeoObj` example (p. 163-165)

Josuttis uses a **closed** set of geometric types—**no common base class** and **no virtual functions** required. Each concrete type implements its own **`draw()`** member; dispatch is via **`std::visit`** (or index), not vtables.

```
 Polymorphism via variant (closed set)
 ┌─────────────────────────────────────────┐
 │ GeoObj = variant<Line, Circle, Rectangle>│
 │   each type: concrete class + draw()     │
 └─────────────────────────────────────────┘
        │
        ▼ std::visit(lambda, geoObj) ──► obj.draw()
```

```cpp
#include <variant>
#include <vector>
#include <iostream>

struct Line {
    void draw() const { std::cout << "Line\n"; }
};
struct Circle {
    double r{};
    void draw() const { std::cout << "Circle r=" << r << '\n'; }
};
struct Rectangle {
    double w{}, h{};
    void draw() const { std::cout << "Rectangle " << w << 'x' << h << '\n'; }
};

using GeoObj = std::variant<Line, Circle, Rectangle>;

void testGeo() {
    std::vector<GeoObj> coll;
    coll.push_back(Line{});
    coll.push_back(Circle{1.5});
    coll.push_back(Rectangle{2.0, 3.0});

    for (const auto& obj : coll) {
        std::visit([](const auto& o) { o.draw(); }, obj);
    }
}
```

**Takeaway**: the **variant** is the sum type; **`visit` + generic lambda** (or a visitor struct) is the uniform call pattern. Adding a type means editing **`GeoObj`** and recompiling—by design for a fixed world.

### Closed set (variant) vs open set (virtuals) (p. 163-165)

| Aspect              | `std::variant` (closed)    | `virtual` + inheritance (open)  |
|---------------------|----------------------------|---------------------------------|
| Add new types       | Edit variant, recompile    | Derive new class, link          |
| Dispatch            | `visit`/index (static)     | vtable (dynamic)                |
| Storage             | Inline max size            | Often heap + pointer            |
| Performance         | Often faster (localized)   | Pointer chasing + virtual call  |

Josuttis notes `variant` is attractive when the set is **fixed** and performance predictability matters.

### Heterogeneous collections with compile-time dispatch (p. 165-167)

A `std::vector` of **`std::variant<int, double, std::string>`** (or similar) stores **rows** of mixed types. **Printing** can use **`std::visit`** so **strings** get **quotes** and **numbers** print plainly—using **`if constexpr`** inside a **generic lambda** (same idea as p. 158: **`is_convertible`** / string detection on `decltype(val)`).

```cpp
#include <vector>
#include <variant>
#include <string>
#include <iostream>
#include <type_traits>

using Var = std::variant<int, double, std::string>;

void printColl(const std::vector<Var>& coll) {
    for (const auto& elem : coll) {
        std::visit([](const auto& val) {
            if constexpr (std::is_convertible_v<decltype(val), std::string>) {
                std::cout << '"' << val << '"';
            } else {
                std::cout << val;
            }
            std::cout << ' ';
        }, elem);
    }
    std::cout << '\n';
}
```

**Visitor with overloaded alternatives** (same collection type, explicit lambda per alternative):

```cpp
template<class... Ts>
struct overload : Ts... { using Ts::operator()...; };
template<class... Ts>
overload(Ts...) -> overload<Ts...>;

void printCollOverloaded(const std::vector<Var>& coll) {
    for (const auto& elem : coll) {
        std::visit(overload{
            [](int i) { std::cout << i << ' '; },
            [](double d) { std::cout << d << ' '; },
            [](const std::string& s) { std::cout << '"' << s << "\" "; },
        }, elem);
    }
    std::cout << '\n';
}
```

Josuttis shows **generic lambdas** with **`if constexpr`** and **`overload{ ... }`** visitors as complementary styles for heterogeneous sequences.

---

## 16.5 Special cases (p. 168)

### `bool` and `std::string` [OOPS]

```cpp
std::variant<bool, std::string> v = "hi";  // may NOT do what you expect
```

**Problem**: unscoped overload resolution: a **string literal** can convert to `const char*`, and `const char*` converts to `bool` (non-null pointer -> `true`). So the literal may initialize the **`bool`** branch, not `std::string`.

**Fix**: qualify the alternative:

```cpp
v = std::string{"hi"};
```

or use different design (order alternatives carefully, prefer `std::string` before `bool`, but the implicit `const char*` -> `bool` pitfall remains for certain initializations). Josuttis highlights this as a **real footgun**.

---

## ASCII: variant vs union vs inheritance

```
 C union (untagged)
 ┌─────────────────────────┐
 │  raw storage (max size) │  programmer must remember active member
 └─────────────────────────┘

 std::variant<A,B,C> (tagged)
 ┌───────┬─────────────────────────┐
 │ index │ storage for active type │
 └───────┴─────────────────────────┘

 Inheritance + virtual
      Shape*
         │
    ┌────┴────┬────────┐
    v         v        v
  Circle   Rectangle  Line
  (usually separate allocations when heap-polymorphic)
```

---

## ASCII: variant memory layout (schematic)

```
 variant<int, std::string> (conceptual)
 ┌──────────────────────────────────────┐
 │ discriminator (which alternative)    │
 │ [possible padding]                   │
 │ union-like storage for max(sizeof...)│
 │   either int bits                    │
 │   or string object (SSO / heap ptr)  │
 └──────────────────────────────────────┘
```

(Exact layout is implementation-defined; string may use SSO so "heap ptr" is not always present.)

---

## ASCII: visitor dispatch

```
 visit(visitor, variant)
        │
        ├─ index == 0 ──> visitor(Alt0&)
        ├─ index == 1 ──> visitor(Alt1&)
        └─ index == 2 ──> visitor(Alt2&)

 * Single dynamic branch on index, then static call
 * Compared to virtual: no per-object vptr indirection for the variant object itself
```

---

## Summary

- **`variant`** is the standard **tagged union** for a **closed** type set (Josuttis pp. 149-168).
- **Construction** can be **[OK]** (unique best alternative), **[ERROR]** (ambiguous overload), or surprising **[OOPS]** (`bool` vs string literal). Use **`in_place_type` / `in_place_index`** when you must disambiguate; there is **no CTAD / `make_variant`**.
- **`std::get<T>`** is **[ERROR]** at compile time if **duplicate** `T` appears in the list; use **`get<I>`** by index.
- **Comparisons**: earlier index wins for ordering when indices differ; same index compares values; **valueless** compares specially; **[ERROR]** to compare **different** `variant<...>` specializations.
- Prefer **`get_if`** / **`holds_alternative`** when you want non-throwing checks.
- **`visit`**: function objects, **mutating** visitors, **generic lambdas**, **`if constexpr`**, unified **return types**, and **`overload{...}`** cover most dispatch patterns.
- Watch **valueless-by-exception** (e.g. throwing conversions in **`emplace`**) and the **`bool` / `const char*`** interaction (p. 168).
