# Chapter 17: `std::any` (The Complete Guide, pp. 171-177)

This chapter follows *C++17 - The Complete Guide* by Nicolai M. Josuttis. Page references use **p.**

---

## 17.1 Using `std::any` (p. 171)

### Empty `any`

```cpp
#include <any>
#include <string>

std::any a;           // empty: no contained object
assert(!a.has_value());
```

### Hold any (copyable) type

```cpp
std::any a = 42;
a = std::string{"hello"};  // replaces stored value
```

`std::any` owns **one** object whose type is erased at compile time but recorded at run time (via `type()` -> `std::type_info`).

### `any_cast<T>(a)` throws on mismatch (p. 172)

```cpp
std::any a = 42;
int n = std::any_cast<int>(a);  // [OK]

try {
    std::any_cast<std::string>(a);  // [ERROR] bad_any_cast
} catch (const std::bad_any_cast&) {
    // wrong type
}
```

### Pointer overload: `nullptr` if wrong type (p. 172)

```cpp
std::any a = 42;
if (int* p = std::any_cast<int>(&a)) {
    *p = 100;  // modify inside the any
}
if (std::any_cast<std::string>(&a) == nullptr) {
    // not a string
}
```

Non-throwing introspection + optional in-place update via pointer.

### `type()`, `has_value()`, `reset()` (p. 173)

```cpp
const std::any& ca = a;
assert(ca.type() == typeid(int));
a.reset();
assert(!a.has_value());
```

### `make_any` (p. 173)

```cpp
auto s = std::make_any<std::string>(5u, 'z');  // "zzzzz"
```

### `any_cast` with references to modify in place (p. 173)

```cpp
std::any a = std::string{"hi"};
std::any_cast<std::string&>(a) += " there";
```

Use the **reference** form when you need to mutate the contained object without replacing the whole `any`.

### Decay behavior when storing (p. 173)

Values are stored using the **decayed** type of the initializer. String literals decay to `const char*`, not `std::string`.

```cpp
std::any a = "hello";
// [OK] a.type() is typeid(const char*), NOT typeid(std::string)
if (a.type() == typeid(const char*)) { /* [OK] true */ }

try {
    (void)std::any_cast<std::string>(a);  // [ERROR] bad_any_cast: wrong type
} catch (const std::bad_any_cast&) { }
```

To hold a `std::string`, construct or assign one explicitly (e.g. `std::string{"hello"}` or `emplace<std::string>(...)`).

### `in_place_type` when the constructed type differs from the initializer (p. 175)

Josuttis shows the **in-place constructor** so the object inside the `any` is built as **`long`** or **`std::string`** even when the initializer alone would not select that type. The first argument is the tag `std::in_place_type_t<T>{}` (C++17); **since C++20** you can write **`std::in_place_type<T>`** as a shorthand `constexpr` tag object.

```cpp
#include <utility>  // std::in_place_type_t

std::any a4{std::in_place_type_t<long>{}, 42};
std::any a5{std::in_place_type_t<std::string>{}, "hello"};  // [OK] std::string inside

// [OOPS] Even in_place_type does not bypass decay for array-to-pointer:
std::any a5b{std::in_place_type_t<const char[6]>{}, "hello"};
// Still stores const char* (array decays); not a char array inside the any
```

### What `std::any` does **not** provide (p. 173)

- **No** `operator==` / `operator!=` (or other comparisons) for `std::any` values.
- **No** `std::hash<std::any>` in the standard library.
- **No** `.value()` member (unlike `optional` / some other wrappers); use `any_cast` after `type()` checks.

---

## 17.2 Types and operations (p. 174)

### Heap allocation and small-buffer optimization (p. 174)

Unlike `optional`/`variant`, `any` may **allocate** on the heap for large or alignment-heavy types, but implementations often apply **small buffer optimization (SBO)**: small types stored inline, big types on the heap. Josuttis explains that this tradeoff makes `any` flexible but less transparent than `variant` for performance reasoning.

### Value semantics: deep copy (p. 174)

Copying `std::any` **copies the contained object** if the held type is copyable. If the held type is not copyable, copying the `any` may be deleted or ill-formed depending on how the `any` was populated (Josuttis notes limitations around non-copyable types).

### `emplace<T>(args...)` (p. 175)

```cpp
std::any a;
a.emplace<std::vector<int>>(5, 0);  // vector of five zeros
```

Destroys previous content (if any) and constructs `T` directly inside the `any`.

### Comparison with `void*` and Boost.Any (p. 175-176)

| Feature           | `void*`                     | `std::any`                    |
|-------------------|-----------------------------|-------------------------------|
| Type safety       | None (manual casts)         | Checked `any_cast`            |
| Ownership         | Manual                      | `any` owns its object         |
| Copy semantics    | Shallow (pointer copy)      | Deep copy of contained object |

Boost.Any inspired `std::any`; migration is usually straightforward for basic usage (Josuttis positions `std::any` as the standard facility).

### Heterogeneous sequence: `std::vector<std::any>` (p. 173)

```cpp
#include <any>
#include <string>
#include <vector>

std::vector<std::any> v;
v.push_back(42);
std::string s = "hi";
v.push_back(s);

for (auto& e : v) {
    if (int* p = std::any_cast<int>(&e)) {
        // [OK] use *p
    } else if (std::string* ps = std::any_cast<std::string>(&e)) {
        // [OK] use *ps
    }
}
```

There is **no** `switch` on the dynamic type and **no** `std::visit` for `any`; dispatch is typically an **if–else** chain (or a map from `type_index` to handlers).

---

## Move semantics with `std::any` (pp. 176-177)

### Move-only types are not supported

`std::any` requires the contained type to be **copyable** (CopyConstructible). **Move-only** types cannot be stored in `std::any` in the usual sense—Josuttis stresses this limitation relative to type-erased containers that only move.

### Moving in and out

```cpp
std::any a;
std::string s = "data";
a = std::move(s);  // [OK] move string into the any (s moved-from)

// Move out (idiomatic): modify through reference, then move from lvalue
std::string out = std::move(std::any_cast<std::string&>(a));  // [OK] preferred idiom

// Alternative: extra move from the temporary returned by value overload
std::string out2 = std::any_cast<std::string>(std::move(a));  // [OK] but one extra move

// std::string&& r = std::any_cast<std::string&&>(a);  // [ERROR] does not compile
```

After moving **out**, `a` typically **`has_value()` remains true**; the contained object is a **moved-from** `std::string` (valid but unspecified state), not an empty `any`. To clear, call `reset()` or assign a new value.

```
 Move-out via any_cast<string&>(a)
 ┌────────────────────────────────────────┐
 │ any still holds type string            │
 │ contained object = moved-from string   │
 └────────────────────────────────────────┘
 Reset explicitly if you need "empty any"
```

---

## ASCII: `optional` vs `variant` vs `any`

```
 Problem: "maybe no value"
 optional<T>
 ┌─────────────────────┐
 │ empty OR exactly T  │
 └─────────────────────┘
  T fixed at compile time

 Problem: "one of several known types"
 variant<A,B,C>
 ┌─────────────────────┐
 │ exactly one of A|B|C│
 └─────────────────────┘
  Closed set, tag is index

 Problem: "any copyable value, type unknown until runtime"
 any
 ┌─────────────────────┐
 │ type-erased object  │
 └─────────────────────┘
  Open (for copyable types), type in runtime type_info
```

**When to choose** (Josuttis-style guidance):

- **`optional`**: absence/presence of a **single** known type.
- **`variant`**: **fixed** alternation between specific types (`visit`-friendly).
- **`any`**: **heterogeneous** maps, plugin-like prototypes, or transitional APIs when you cannot template on all types—accept runtime typing cost.

---

## Pitfalls and style notes

1. **Performance opacity**: every access through `any_cast`; harder for optimizers than `variant` visitation.
2. **Testing types**: prefer pointer `any_cast` when failure is expected.
3. **Large interfaces**: exposing `std::any` widely can obscure contracts; prefer `variant` or templates when callers know types.

---

## Minimal worked example: heterogeneous map (idiom)

```cpp
#include <any>
#include <iostream>
#include <string>
#include <unordered_map>

int main() {
    std::unordered_map<std::string, std::any> bag;
    bag["count"] = 7;
    bag["name"]  = std::string{"Scot"};

    int n = std::any_cast<int>(bag["count"]);
    const std::string& s = std::any_cast<const std::string&>(bag["name"]);
    std::cout << n << ' ' << s << '\n';
}
```

(Production code often still prefers typed structs once the schema stabilizes; Josuttis presents `any` as a tool, not a default design.)

---

## See also in Josuttis

- **§17.1** (p. 171): usage, casts, mutation, decay, no comparison/hash/`value()`.
- **§17.2** (p. 174): mechanics, SBO, `emplace`, `in_place_type_t`, heterogeneous containers.
- **Move semantics** (pp. 176-177): move-in/move-out idioms, moved-from state, no `any_cast` rvalue reference overload.

This file is a tutorial expansion keyed to the book's page ranges; consult the printed chapter for Josuttis's original prose and additional edge cases.
