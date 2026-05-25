# Chapter 15: `std::optional<>` (The Complete Guide, pp. 135-147)

This chapter follows *C++17 - The Complete Guide* by Nicolai M. Josuttis (O'Reilly). Page references use **p.** for a single page.

---

## 15.1 Using `std::optional<>` (p. 135)

### Nullable instance of arbitrary type

`std::optional<T>` models an object that **either** holds a value of type `T` **or** holds **no value**. It is the standard vocabulary type for "maybe absent" without sentinel values, raw pointers, or special return conventions like error codes paired with output parameters.

### Size and storage (p. 135)

- **Contained object**: when engaged, the `optional` stores a full `T` in place (like a small buffer inside the optional).
- **Discriminator**: typically **one extra byte** (or minimal storage) records whether a value is present, subject to **alignment** padding so the whole `optional<T>` remains suitably aligned for `T`.
- **No heap allocation**: unlike `std::unique_ptr<T>`, `optional` does not allocate; everything is inline in the `optional` object.

Josuttis emphasizes that you pay for the tag byte (plus possible padding), not for indirection.

### Deep copy semantics (p. 135)

Copying an `optional<T>` copies the contained `T` when engaged, or copies an empty state when disengaged. Move semantics propagate similarly: moving from an engaged optional moves the inner `T` and leaves the source in a valid but unspecified state consistent with moved-from `T` rules.

---

## 15.1.1 Optional return values (p. 135)

### Parsing example: `asInt`

A classic pattern is returning an optional when a conversion might fail, without exceptions or out-parameters:

```cpp
#include <optional>
#include <string>

std::optional<int> asInt(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return std::nullopt;
    }
}
```

Note: `std::stoi` can throw `std::invalid_argument` or `std::out_of_range`; catching and returning `nullopt` maps all failures to "no value." For production code you may prefer narrower exception handling or a non-throwing conversion strategy; the **idea** from Josuttis is: **optional expresses failure without a separate channel**.

### Checking and accessing

```cpp
auto oi = asInt("42");
if (oi) {
    int n = *oi;  // dereference: defined only if engaged
}
```

- **`if (oi)`** / **`oi.has_value()`**: test presence.
- **`*oi`** / **`oi->`**: access the value when you know it exists (undefined behavior if empty on some operations; prefer safe access below).

### `value()` throws (p. 135-136)

```cpp
try {
    int x = oi.value();  // std::bad_optional_access if empty
} catch (const std::bad_optional_access&) {
    // handle
}
```

### `value_or` (p. 136)

```cpp
int n = oi.value_or(0);  // 0 if no value
```

Useful when a default is always acceptable.

---

## 15.1.2 Optional arguments and data members (p. 137)

### Function parameters

Optional parameters let callers omit information:

```cpp
#include <optional>
#include <iostream>

void printName(const std::string& first,
               const std::optional<std::string>& middle,
               const std::string& last) {
    std::cout << first;
    if (middle)
        std::cout << ' ' << *middle;
    std::cout << ' ' << last << '\n';
}

// Calls:
// printName("John", std::nullopt, "Doe");
// printName("Mary", std::string{"Q"}, "Smith");
```

### Class members

```cpp
struct Person {
    std::string first_name;
    std::optional<std::string> middle_name;  // may be absent
    std::string last_name;
};
```

This avoids sentinel strings like `""` or `"N/A"` that collide with legitimate data.

---

## 15.2 Types and operations (p. 139)

### Complete operations overview (p. 139)

The standard provides the following facilities on `std::optional<T>` (names summarize the intent; exact signatures are in `[optional.optional]`). Use this as a checklist while reading the rest of the chapter.

```
+------------------------------------------------------------------+
| std::optional<T>  --  main operations (Ch. 15 / p. 139)          |
+-------------------+----------------------------------------------+
| Constructors      | default (empty); from nullopt (empty);     |
|                   | from U / T (engaged); copy/move;             |
|                   | in_place + ctor args; in_place + init-list   |
+-------------------+----------------------------------------------+
| std::make_optional| Factory; avoid spelling T twice              |
+-------------------+----------------------------------------------+
| Destructor        | Destroys contained T if engaged              |
+-------------------+----------------------------------------------+
| operator=         | From optional, T, nullopt, in_place args     |
+-------------------+----------------------------------------------+
| emplace           | Destroy old (if any); construct T in place   |
+-------------------+----------------------------------------------+
| reset             | Destroy contained value; become empty        |
+-------------------+----------------------------------------------+
| has_value         | [OK] true iff engaged                        |
+-------------------+----------------------------------------------+
| explicit bool /   | Conversion: [OK] true iff engaged            |
| operator bool     | (use explicit test; do not rely on "truth")  |
+-------------------+----------------------------------------------+
| operator*         | Lvalue/rvalue; [ERROR] if empty: UB          |
+-------------------+----------------------------------------------+
| operator->        | Member access; [ERROR] if empty: UB          |
+-------------------+----------------------------------------------+
| value()           | Throws bad_optional_access if empty; else    |
|                   | returns lvalue/rvalue reference to T         |
+-------------------+----------------------------------------------+
| value_or(U&&)     | Returns contained T or decay-copy of default   |
+-------------------+----------------------------------------------+
| swap              | noexcept if T is swappable noexcept          |
+-------------------+----------------------------------------------+
| Comparisons       | optional vs optional, vs T, vs nullopt;      |
|                   | mixed T / U with converted values            |
+-------------------+----------------------------------------------+
| std::hash         | If hash<T> exists: engaged uses hash<T>;    |
|                   | empty: hash unspecified (p. 146)             |
+------------------------------------------------------------------+
```

---

### Construction details (p. 140)

**Default and explicit empty**

Both of these create an empty optional:

```cpp
std::optional<int> o1;                        // default: empty
std::optional<int> o2{std::nullopt};          // explicitly empty
```

**Class template argument deduction (CTAD, C++17)**

When the initializer fixes `T`, you can write `std::optional` without `<T>`:

```cpp
std::optional o3{42};           // deduces std::optional<int>
std::optional o4{"hello"};      // deduces std::optional<const char*>  [OOPS] not std::string!
```

[OOPS] `o4` holds a pointer to the string literal’s array, not a `std::string`. If you want a string, spell it: `std::optional<std::string>{"hello"}` or `std::optional o4{std::string{"hello"}};`.

**`std::in_place` with constructor arguments**

Construct the contained object directly inside the optional:

```cpp
#include <complex>
#include <optional>

std::optional<std::complex<double>> o7{
    std::in_place, 3.0, 4.0};   // complex(3.0, 4.0) in place
```

**`std::in_place` with `initializer_list` plus extra arguments**

Useful for containers whose constructors take an init-list first, then more parameters (here, a comparator):

```cpp
#include <cstdlib>  // std::abs
#include <functional>
#include <optional>
#include <set>

auto sc = [](int a, int b) {
    return std::abs(a) < std::abs(b);
};

// Brace form as in the book; forwarded to set's (initializer_list, Compare) ctor:
std::optional<std::set<int, decltype(sc)>> o8{
    std::in_place, {4, 8, -7, -2, 0, 5}, sc};
```

**`std::make_optional`**

Lets the type be inferred or supplies explicit template arguments for the contained object’s constructor:

```cpp
auto o13 = std::make_optional(3.0);   // std::optional<double>, value 3.0

auto o15 = std::make_optional<std::complex<double>>(3.0, 4.0);
```

**Ternary with `optional` and `nullopt`**

When one branch has a value and the other is empty, both operands should be `optional` types (or convertible so the expression has a common `optional` type):

```cpp
#include <map>
#include <optional>
#include <string>

std::map<std::string, int> m;
auto pos = m.find("key");
auto end = m.end();
auto o16 = (pos != end) ? std::optional{pos->second} : std::nullopt;
```

[OK] The ternary keeps the “maybe absent” story in one expression without a dummy value.

**Copy / convert between optionals of related types**

Copying from `optional<const char*>` to `optional<std::string>` uses `std::string`’s converting constructor:

```cpp
std::optional<const char*> o9{"hello"};
std::optional<std::string> o10{o9};   // engaged std::string("hello")
```

**`emplace` after construction (same subsection as in_place in usage)**

```cpp
#include <optional>
#include <complex>

std::optional<std::complex<double>> oc;
oc.emplace(1.0, 2.0);  // destroys nothing (was empty), constructs in place
```

---

### Value access: references, temporaries, and `value_or` (p. 142)

**`value()` returns a reference**

`value()` yields `T&`, `const T&`, `T&&`, or `const T&&` depending on the value category of the optional (see **Move semantics** below). That means a reference can outlive the temporary optional or temporary contained object.

[ERROR] Dangling reference from a temporary:

```cpp
#include <optional>
#include <string>

std::optional<std::string> getString();

void bad() {
    const std::string& r1 = getString().value();  // [ERROR] r1 dangles after full-expression
}
```

[ERROR] Iterating a container accessed through a temporary optional:

```cpp
#include <optional>
#include <vector>

std::optional<std::vector<int>> getVector();

void bad_loop() {
    for (int i : getVector().value()) {  // [ERROR] vector destroyed; loop undefined
        (void)i;
    }
}
```

Fix pattern: name the optional (or the vector) in a variable whose lifetime covers all uses.

**`value()` vs `value_or()`**

- `value()` returns **by reference** (no allocation for `T`; may propagate references into danger with temporaries as above).
- `value_or(default)` returns **by value** (returns a copy of the contained value, or a copy of `default`). For expensive `T` (e.g. `std::string`), that may allocate or copy more than you need.

**Efficiency idiom when you only need a “view”**

For `optional<string>`, printing without an extra string copy often looks like:

```cpp
#include <iostream>
#include <optional>
#include <string>

void print(std::optional<std::string> const& o) {
    std::cout << (o ? o->c_str() : "fallback");   // [OK] no value_or copy
}
```

[OOPS] Do not write `std::cout << o ? o->c_str() : "fallback";`—without parentheses, `<<` binds tighter than `?:`, so the condition is wrong.

Use `value_or` when an actual `T` object (by value) is what you need.

**`*` and `->`**

[ERROR] Using `*o` or `o->` on an empty optional is **undefined behavior**. `value()` is how you get a checked reference (or an exception).

---

### Detailed comparison rules (p. 143-144)

**Both empty**

Two empty optionals compare **equal** (`==` true, neither less than the other for a total order consistent with the standard).

**Exactly one empty**

In relational comparisons, the empty optional is **less than** any engaged optional.

**Compared with a bare value (or `nullopt`)**

Rough picture: empty optional behaves like “no value” and sorts **less than** any engaged value. Effects on mixed `optional` / `T` comparisons match that story.

[OOPS] Surprising but intended:

```cpp
#include <cassert>
#include <optional>

void ordering_surprises() {
    std::optional<unsigned> uo;   // empty
    assert((uo < 0U) == true);      // [OK] empty < any engaged unsigned (here: 0)
    // Same idea with plain 0: uo < 0 is also true (usual arithmetic conversions).

    std::optional<bool> bo;        // empty
    assert((bo < false) == true);  // [OK] empty < engaged false — not “bool meaning”
}
```

Do not read `<` as “numeric less” when the left-hand side might be empty; read it as **optional three-way ordering**.

**Mixed `optional` types with comparable values**

If the contained values compare equal after usual conversions, the optionals compare equal:

```cpp
#include <cassert>
#include <optional>

void mixed_types() {
    std::optional<int> o1{42};
    std::optional<double> o2{42.0};
    assert(o1 == o2);   // [OK] values equal as double/int
}
```

`optional` also compares to `nullopt`, to `T`, and (where supported) between `optional<T>` and `optional<U>` using the same “both empty / one empty / both engaged” decomposition.

---

### Changing values (p. 144-145)

Ways to replace or clear the contained object (Josuttis: `emplace`, assignment, `reset`, empty state, `nullopt`):

```cpp
#include <optional>
#include <string>

std::optional<std::string> os{"hi"};

os.emplace(3, 'y');           // destroy old value (if any); construct new std::string in place
os = std::string{"again"};    // assign engaged value (replaces contents)
os = std::nullopt;           // [OK] disengage (no value)
os.reset();                  // [OK] destroy contained object, optional becomes empty
```

**`operator=` and `{}`**

- `os = std::nullopt` clears the optional ([OK]).
- `os = some_T` engages or replaces the stored `T`.
- [OOPS] Plain `os = {}` is **not** “clear the optional” in general: it typically uses `optional<T>::operator=(U&&)` with a **value-initialized** `T` (for `std::string`, that is an empty string—you stay **engaged**). For `std::optional<int>`, `o = {}` sets the value to `0`, still engaged.
- To assign “empty optional” from another optional, you can write `os = std::optional<std::string>{};`.

For `optional<T>`, assignment from `nullopt` clears. Assigning a `T` engages (or replaces) the value.

[ERROR] Assigning through dereference on an empty optional:

```cpp
std::optional<int> o;
*o = 42;   // [ERROR] undefined behavior: no value object to assign into
```

Always ensure engagement (`if (o)` / `has_value()`) before `*o = ...`, or use `emplace` / `operator=` / `value()` on an engaged object.

---

### Move semantics (p. 145-146)

**Moving the whole `optional`**

When you move from `std::optional<T>`, the **empty/engaged flag** is copied as appropriate and, if engaged, the contained `T` is **moved-from** (left in a valid but unspecified state per `T`).

**Move-assign into optional**

```cpp
#include <optional>
#include <string>

std::string s{"data"};
std::optional<std::string> os;
os = std::move(s);   // engaged; s moved-from
```

**Move value out**

```cpp
std::optional<std::string> os{"data"};
std::string s3 = std::move(*os);   // move from contained; optional still engaged [OOPS] but value unspecified/empty string typical
```

[OOPS] After moving `*os` out, the optional may still be “engaged” but its `string` is hollowed out; clear or assign if you need a known state.

**Ref-qualified `value()` and `operator*`**

`value()` and `*` have overloads for `&`, `const&`, `&&`, and `const&&` on the optional, so you can move out of rvalue optionals:

```cpp
#include <optional>
#include <string>

std::optional<std::string> func();

void take() {
    std::string s4 = func().value();   // [OK] moves from temporary optional’s value
}
```

---

### Hashing (p. 146)

If `std::hash<T>` is available, `std::hash<std::optional<T>>` is defined such that:

- **Engaged**: the hash is computed as for the contained `T` ([OK] same as hashing the value).
- **Empty**: the hash is **unspecified** by the standard (do not rely on a particular value across implementations or runs).

---

### `std::nullopt` reminder (p. 140)

```cpp
std::optional<int> o = std::nullopt;  // explicitly empty
```

`nullopt` is a tag meaning “disengaged optional” and participates in comparisons and assignments.

### `emplace`, `reset`, `has_value` (quick recap, p. 142-143)

```cpp
std::optional<std::string> s;
s.emplace(5, 'x');      // in-place std::string(5, 'x')
s.reset();              // destroy value; optional is empty
assert(!s.has_value());
```

### Monadic operations (overview, p. 143-145 context)

Josuttis discusses thinking of `optional` as supporting **monadic composition**: each step either produces a value or stops the chain. The **C++17 standard library** does not yet provide `and_then` / `transform` / `or_else` on `std::optional` (those arrive in C++23), but you express the same logic explicitly.

**Manual "bind" in C++17** (flatten nested optionals after a step):

```cpp
#include <cmath>
#include <optional>
#include <string>
#include <type_traits>

template <class T, class F>
auto flat_map(std::optional<T> o, F&& f)
    -> std::optional<std::invoke_result_t<F, T>> {
    if (!o) return std::nullopt;
    return f(*o);
}

std::optional<int> asInt(const std::string&);  // e.g. stoi wrapped as in 15.1.1
std::optional<double> sqrt_pos(int x) {
    if (x < 0) return std::nullopt;
    return std::optional<double>{std::sqrt(static_cast<double>(x))};
}

std::optional<double> parsed_root(const std::string& s) {
    return flat_map(asInt(s), [](int n) { return sqrt_pos(n); });
}
```

**`or_else` style** in C++17: use `value_or`, or `if (!o) o = fallback();`, or a small helper that returns the first engaged optional among alternatives. The **idea** (per Josuttis): keep failure implicit in the type instead of mixing sentinel values and error flags.

---

## 15.3 Special cases (p. 146)

### `optional<bool>`: presence vs value

```cpp
std::optional<bool> ob{false};  // HAS value, value is false
if (ob) { /* true: optional is engaged */ }  // [OK] tests ENGAGEMENT
```

**Gotcha**: `if (ob)` answers "is there a bool?" not "is the bool true?" Use:

```cpp
if (ob && *ob) { /* value is true */ }
```

### `optional<T*>`: presence vs null pointer

```cpp
int x = 1;
std::optional<int*> op{&x};
if (op) { /* engaged: we have a pointer object */ }

std::optional<int*> emptyPtr{nullptr};
// Still "engaged" if you constructed with null pointer!
```

Actually: if you **default-construct** or assign `nullopt`, it's empty. If you store `nullptr` as the **value** of an **engaged** optional, `if (op)` is still true. So:

- **`if (op)`** -> is the optional engaged?
- **`if (op && *op)`** -> is there a non-null pointer **value**?

Josuttis warns this double indirection is easy to misuse.

### Nested optionals and “optional of optional” (p. 146-147)

A value can be wrapped twice: the **outer** `optional` says whether the whole cell exists; the **inner** `optional` can represent another level of absence.

**Construction from a string literal (book example)**

Thanks to converting constructors, you can initialize nested optionals from a string literal; the result is **outer engaged**, **inner engaged**, holding a `std::string`:

```cpp
#include <optional>
#include <string>

std::optional<std::optional<std::string>> oos2 = "hello";   // outer + inner both engaged
```

**Two different ways to get “no value”**

```cpp
std::optional<std::optional<std::string>> oos1{std::in_place, "inner"};  // engaged / engaged

// Inner becomes empty; outer is still engaged (you have an “empty inner optional” object):
*oos1 = std::nullopt;

// Outer becomes empty (the whole thing is absent):
oos1 = std::nullopt;
```

Diagram (states):

```
 Outer optional        Inner optional        Meaning
 ─────────────────────────────────────────────────────────────
 engaged               engaged             Value is present (**oos gives string)
 engaged               empty               “Slot exists” but inner has no string
 empty                 (inactive)          Whole thing absent
```

**Accessing the inner string**

You need two successful levels before using the contained `std::string`:

```cpp
#include <iostream>
#include <optional>
#include <string>

void print_nested(std::optional<std::optional<std::string>> const& oos1) {
    if (oos1 && *oos1) {
        std::cout << **oos1 << '\n';   // [OK] outer engaged and inner engaged
    }
}
```

[ERROR] Using `**oos1` without checking both levels is unsafe (undefined behavior if either level is empty), just like `*` on a single empty optional.

**Another illustration**

```cpp
std::optional<std::optional<int>> oo{std::optional<int>{42}};
// Outer engaged, inner engaged, value 42.

std::optional<std::optional<int>> oo_empty_inner{
    std::optional<int>{}};  // outer engaged, inner empty — distinct from outer empty
```

Outer optional might be empty while inner encodes another layer of “maybe.” Useful rarely but **meaningful**: for example, outer = “field was supplied in JSON”, inner = “parsed int or null in JSON.”

---

## Memory layout (ASCII)

Conceptual layout (implementation-defined sizes; diagram is pedagogical):

```
 optional<T> object (inline storage)
 ┌──────────────────────────────────────────────┐
 │  engaged flag / discriminator (often 1 byte) │
 │  [padding for alignment of T]                │
 │  ┌────────────────────────────────────────┐  │
 │  │  storage for T (sizeof(T), aligned)    │  │
 │  └────────────────────────────────────────┘  │
 └──────────────────────────────────────────────┘
 * No pointer to heap for the T itself
```

Comparison semantics (schematic):

```
 State           Compares as
 ─────────────────────────────────────
 nullopt         "less than" any engaged value (when ordering)
 engaged a, b    lexicographic / type rules as for T
 empty vs empty  equal
```

---

## Quick reference table

### Book-style checklist (p. 139)
|-------------------|--------------------|
| Constructors      | default (`empty`), `nullopt`, copy/move, converting, `in_place` + args |
| `make_optional` | Factory function(s); deduction / explicit `T` |
| Destructor      | Destroys contained `T` if engaged |
| `operator=`     | Assign from optional, from `T`, from `nullopt`, etc. |
| `emplace`       | Destroy old value (if any); construct `T` in storage |
| `reset`         | Destroy value; optional empty |
| `has_value`     | [OK] `true` iff engaged |
| `operator bool` | [OK] explicit test for engagement |
| `operator*`     | Lvalue/rvalue; [ERROR] if empty: UB |
| `operator->`    | Member access; [ERROR] if empty: UB |
| `value()`       | Throws if empty; else reference to `T` (ref-qualified: see move section) |
| `value_or`      | Value or fallback (by value) |
| `swap`          | Exchange states/values (`noexcept` when `T` supports it) |
| Comparisons     | `optional`/`optional`, `optional`/`T`, `optional`/`nullopt`, mixed types |
| `std::hash`     | Engaged: hash of `T`; empty: unspecified (p. 146) |

---

### Empty vs engaged (symptoms)

| Operation       | Empty optional   | Engaged optional   |
|-----------------|------------------|--------------------|
| `if (o)`        | false            | true               |
| `*o` / `o->`    | UB / invalid use | access `T`         |
| `o.value()`     | throws           | returns `T&`       |
| `o.value_or(d)` | returns `d`      | returns value      |
| `o.reset()`     | no-op / stays    | destroys `T`       |

---

## Further reading in Josuttis

- **§15.1** (p. 135): usage patterns and motivation.
- **§15.2** (p. 139): complete operation set and edge cases.
- **§15.3** (p. 146): `bool`, pointers, nesting.

This tutorial expands the book's examples with self-contained snippets; align your coding style and error-handling discipline with your project's rules.
