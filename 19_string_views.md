# Chapter 19: String Views (The Complete Guide, pp. 185-198)

This chapter follows *C++17 - The Complete Guide* by Nicolai M. Josuttis. Page references use **p.**

---

## 19.1 Differences vs `std::string` (p. 185)

`std::string` **owns** a character buffer (typically heap-allocated beyond SSO threshold). `std::string_view` is a **non-owning** reference to a contiguous character sequence, represented as **`const char*` + `size`**.

Key contrasts Josuttis emphasizes:

| Aspect              | `std::string`        | `std::string_view`      |
|---------------------|----------------------|-------------------------|
| Ownership           | Owns buffer          | Does not own            |
| Allocation          | May allocate         | No allocation for view  |
| Null termination    | Often NTBS-compatible| **Not guaranteed**      |
| Lifetime            | Self-contained       | **Depends on source**   |

---

## 19.2 Using string views (p. 186)

### Construction details (pp. 193-195)

**Default-constructed view:**

```cpp
std::string_view sv0;
// sv0.data() == nullptr; sv0.size() == 0
// sv0[0] is invalid / undefined; do not index without checking size
```

**From a null-terminated string literal** (length excludes the terminating `\0`):

```cpp
std::string_view sv{"hello"};
// size() == 5; sv[0]..sv[4] valid; sv[5] is [ERROR] / UB (not part of the view)
```

**Including the null terminator in the span** (length counts the extra byte):

```cpp
std::string_view sv_nt{"hello", 6};  // [OK] size() == 6, last char is '\0'
```

**From `std::string`:** implicit conversion yields a view into the string’s contiguous storage. The view is **not** guaranteed to be null-terminated—only `size()` bytes are valid.

**`constexpr` views:**

```cpp
constexpr std::string_view hello = "Hello World!";  // [OK]
```

### Construction from literal (quick form)

```cpp
#include <string_view>

std::string_view sv = "hello";
```

The view points at static string literal storage with known length (implementation forms the view without allocating).

### Substrings and trimming (p. 186)

```cpp
std::string_view sv = "  hello world  ";
sv.remove_prefix(2);   // "hello world  "
sv.remove_suffix(2);   // "hello world"
auto tok = sv.substr(0, 5);  // view to first 5 chars: "hello"
```

**Josuttis-style trim example** (`remove_prefix` / `remove_suffix`, p. 195):

```cpp
#include <iostream>
#include <string_view>

std::string_view sv = "I like my kindergarten";
sv.remove_prefix(2);   // drop "I "
sv.remove_suffix(8);   // drop "garten" (8 chars from end)
std::cout << sv;       // [OK] prints: like my kind
```

**Note**: `substr` on a `string_view` returns another **view**; still no allocation.

### No `operator+` for `string_view` (pp. 195-196)

`string_view` does not concatenate with `+` like strings:

```cpp
// auto s1 = sv1 + sv2;              // [ERROR] no operator+
auto s2 = std::string(sv1) + sv2;    // [OK] at least one std::string participates
```

### Cheap to copy (p. 186)

Copying a `string_view` copies **pointer + length** (two word-like members typically), not the characters.

---

## 19.3 Using as parameters (p. 186)

### Pass by value (p. 187)

```cpp
void process(std::string_view sv);  // cheap by-value
```

Josuttis argues this is often superior to `const std::string&` when the callee only needs **read-only** access, because call sites can pass **literals and `const char*`** without constructing a temporary `std::string`.

### When to use (p. 187)

- Parsing, scanning, logging prefixes/suffixes.
- APIs that only **read** characters and do not need to extend lifetime.

---

## 19.3.1 String view considered harmful (p. 188)

### Dangling views

A `string_view` does not keep the underlying characters alive.

**Bad pattern** (conceptually what Josuttis warns against):

```cpp
std::string_view bad() {
    std::string s = "temporary";
    return s;  // [ERROR] s destroyed; view dangles
}
```

After the function returns, the pointer inside the view is invalid.

### Do not return `string_view` from factories Local strings

If a function **creates** a `std::string` and the caller needs to observe it, return `std::string` (or move it), not `string_view`.

### Do not store `string_view` without lifetime proof

```cpp
struct Cache {
    std::string_view ref;  // dangerous if source may disappear
};
```

Prefer `std::string` for stored text unless you **prove** the backing buffer outlives `Cache` (e.g., points into static storage or a guaranteed parent string).

### No implicit conversion to `std::string`

```cpp
std::string_view sv = "hi";
std::string s{sv};  // explicit construction
```

Implicit conversion would hide allocations; the design forces clarity.

### Not null-terminated (p. 189-190)

You **cannot** safely pass `sv.data()` to `strlen` or legacy APIs that require a null-terminated string unless you have a separate guarantee. Josuttis recommends using the **size** (`sv.size()`) with modern APIs, or materialize an `std::string` when calling C APIs that need NTBS.

### Function templates should use return type `auto` (p. 190)

If a generic function returns `x + y` for string-like `T`, **naming** the return type as `T` can yield a **`string_view`** that **dangles** when temporaries participate in `+`. Returning **`auto`** lets `operator+` deduce `std::string` when that is the natural result.

```cpp
template <typename T>
auto concat(const T& x, const T& y) { return x + y; }  // [OK] e.g. std::string when x,y are string + literal chain

// template <typename T>
// T concat_bad(const T& x, const T& y) { return x + y; }
// [OOPS] If T is std::string_view, the result can be string_view to dead temporaries -> dangling
```

**Rule of thumb:** with `auto`, `xy` can stay **`std::string`** when that is what `operator+` produces; if the return type is forced to **`T`**, **`xy` may become `string_view`** even when the expression built temporaries—**[ERROR] lifetime trap**.

### Do not use `string_view` in call chains that build ownership (pp. 190-191)

**Anti-pattern:** constructor takes `string_view` but **stores** a `std::string` member initialized from that view—callers can pass temporaries; the view can dangle before the member string is fully established in larger refactorings, and **moves** through the API degrade.

```cpp
// void Person(std::string_view n) : name{n} {}  // [OOPS] brittle for temporaries / chains

std::string s = "Ada";
// Person p3{std::move(s)};  // [OOPS] broken move: may convert to view first, then new string
```

**Better:** take owning `std::string` and move into the member:

```cpp
// Person(std::string n) : name{std::move(n)} {}  // [OK]
```

### Overloading ambiguity: `const std::string&` vs `string_view` (p. 197)

```cpp
void foo(const std::string&);
void foo(std::string_view);

// foo("hello");  // [ERROR] ambiguous: both overloads viable
```

Resolve with an explicit cast, `std::string{"hello"}`, or `""sv`, or remove/rename one overload.

### Summary rules (pp. 191-192)

- Do **not** use `string_view` in APIs that **always** sink into `std::string` anyway—take `std::string` (or templates) and **move**.
- Do **not** initialize **`std::string` data members** directly from **`string_view` parameters** without proven lifetime.
- Do **not** **return** `string_view` **unless** you are clearly forwarding storage the caller already owns (static, parent string, etc.).
- Function templates should **not** blindly return `T` when `T` might be `string_view` and the value is computed from temporaries.
- **Never** use a **returned** value to initialize a **`string_view`** unless lifetime is proven.
- **AAA** (“Almost Always `auto`”) interacts badly with `string_view`: `auto v = expr` can silently become a **view** where you meant an **owning** `std::string`—be explicit when ownership matters.

---

## 19.4 Types and operations (p. 192)

### Type family

- `std::string_view` (`char`)
- `std::wstring_view`
- `std::u16string_view`
- `std::u32string_view`

### Operations (p. 192-194)

Essentially the **const** part of the `std::string` interface: `find`, `starts_with` (C++20 additions aside), iteration, indexing with bounds discipline, `compare`, etc. (Exact method availability follows the standard for each revision; Josuttis maps `string` familiarity to `string_view`.)

### User-defined literal `operator ""sv` (p. 195)

```cpp
using namespace std::literals;
auto sv = "hello"sv;  // std::string_view
```

### Hashing (p. 195)

`std::hash<std::string_view>` allows `unordered_map` with `string_view` keys **if** lifetimes are managed carefully (another dangling trap if keys view freed buffers).

---

## 19.5 Using in APIs (p. 196)

### Replacing `const std::string&`

```cpp
void logMessage(std::string_view msg);
```

**Call sites:**

```cpp
logMessage("literal");
std::string s = "owned";
logMessage(s);
```

Implicit conversions produce a **temporary view** pointing into existing storage (`string` has contiguous data; Josuttis discusses temporary validity for the duration of the call).

### Implicit conversion from `string` and literal (p. 196)

This is the ergonomic win: fewer redundant `std::string` temporaries at boundaries.

---

## ASCII: `string_view` internals

```
 std::string_view (conceptual)
 ┌───────────────────────────────────────┐
 │ pointer to first char (const char*)   │
 │ size_t length                         │
 └───────────────────────────────────────┘
           │                    │
           └────>  h e l l o    (storage owned elsewhere)
```

---

## ASCII: lifetime pitfall

```
 Timeline
 ──────────────────────────────────────────────────────────>
 1) std::string s = make();     [====string buffer====]
 2) std::string_view v = s;            v ----> buffer
 3) s destroyed                      [buffer freed]
 4) use v                            [ERROR] dangling
```

Rule: **view must not outlive** the character storage.

---

## Practical checklist

- [OK] Use `string_view` for **read-only** parameters **when you do not extend lifetime** and do not force awkward ownership chains.
- [OK] Use `std::string` for **ownership** or **storage** in types.
- [OK] Prefer **`auto`** return from generic concatenation helpers so `+` can produce `std::string` when appropriate.
- [ERROR] Return `string_view` to **interior** of local `std::string`.
- [ERROR] Assume `data()` is null-terminated.
- [ERROR] Store `string_view` without documenting lifetime dependency.
- [ERROR] Chain APIs that turn `std::string` into `string_view` then re-wrap as `std::string` without a clear owning parameter.
- [ERROR] Overload `foo(const std::string&)` and `foo(std::string_view)` and call with string literals without disambiguation.

---

## See Josuttis

- **§19.1-19.3** (pp. 185-191): semantics, parameters, construction, `remove_prefix`/`remove_suffix`, no `operator+`, overload ambiguity.
- **§19.3.1 / rules** (pp. 188-192): lifetime, templates/`auto` return, owning APIs, AAA vs `string_view`.
- **§19.4-19.5** (pp. 192-198): API migration, hashing, `""sv`.

This tutorial expands examples around the book's guidance; for exhaustive trait tables and standard wording, use the O'Reilly text directly.
