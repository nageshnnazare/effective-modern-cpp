# Chapter 7: New Attributes and Attribute Features

**Book reference:** *C++17 - The Complete Guide* by Nicolai M. Josuttis (pages 55-59)

C++17 standardizes several attributes that compilers can use for **warnings**, **portable annotation**, and **API evolution**. This chapter focuses on `[[nodiscard]]`, `[[maybe_unused]]`, and `[[fallthrough]]`, plus **general attribute extensions** (namespaces, enumerators, attribute namespaces with `using`).

---

## 7.1 Attribute `[[nodiscard]]` (p. 55)

### Purpose

**`[[nodiscard]]`** encourages Diagnosing **discarded return values**. If you call a function and ignore its result, the compiler may emit a **warning**.

### Typical use cases (book)

* **Resource allocation** patterns where ignoring the return value leaks (e.g. C-style `malloc` wrappers -- though in modern C++ you prefer smart pointers).
* **Async** APIs where discarding the return value (e.g. from `std::async`) may mean you never observe results or exceptions.
* **Query functions** like `empty()` where ignoring the result is almost always a logic bug.

### Class member function example

```cpp
class MyContainer {
public:
    [[nodiscard]] bool empty() const noexcept;
};
```

Callers should use the result:

```cpp
void client(MyContainer& coll) {
    if (coll.empty()) { /* ... */ }
}
```

### Suppressing the warning intentionally

If you truly intend to discard:

```cpp
(void)coll.empty();
```

The cast-to-void idiom documents intent and typically silences `[[nodiscard]]` warnings.

### Inheritance

**`[[nodiscard]]` is *not* inherited** by overriding functions in the sense you might expect from `override` -- if you need the warning on **all** overrides, repeat the attribute in the derived class as appropriate (book warns about this pitfall).

### Placement

Attributes may appear:

* **Before** the declaration specifiers; or  
* **After** the function declarator (after the parameter list / function name part depending on syntax).

Consult your compiler for the exact position it prefers in tricky cases; both placements are standardized for functions.

### Standard library status in C++17

Many `[[nodiscard]]` additions people associate with `optional::value()`, `vector::empty()`, etc. landed in **C++20** for the standard library. Josuttis notes that in **C++17** the attribute exists in the language, but **standard library marking was largely deferred**.

---

## 7.2 Attribute `[[maybe_unused]]` (p. 57)

### Purpose

Silence **unused entity** warnings for **names** that must exist for API or debugging reasons but are not always referenced.

### Parameters

```cpp
void foo(int val, [[maybe_unused]] std::string msg) {
    // msg might only be used in logging builds
}
```

### Members, variables, functions, types, enumerators

Applicable to many declaration kinds, e.g.:

```cpp
[[maybe_unused]] static void helper() { }

enum class E {
    a = 1,
    b [[maybe_unused]] = 2,
};
```

### Common mistake: applying to a *statement*

```cpp
// [[maybe_unused]] foo();  // [ERROR] not a declaration
```

### But on a *variable* initialized from a call

```cpp
[[maybe_unused]] auto x = foo();  // [OK] names 'x'
```

The attribute applies to **`x`**, not to the statement alone.

---

## 7.3 Attribute `[[fallthrough]]` (p. 58)

### Purpose

In `switch` statements, compilers often warn on **implicit fall-through** between `case` labels. `[[fallthrough]]` documents **intentional** fall-through.

### Must appear as its own statement

It must be used in an **empty statement** ending with `;` (book emphasizes the syntax discipline).

### Example

```cpp
#include <cassert>

void dispatch(int x) {
    switch (x) {
        case 1:
            do_first_thing();
            [[fallthrough]];   // intentional
        case 2:
            do_second_thing();
            break;
        default:
            assert(false);
            break;
    }
}
```

Without `[[fallthrough]];`, `-Wimplicit-fallthrough` or similar may warn on `case 1`.

---

## 7.4 General Attribute Extensions (p. 58)

### Attributes on **namespaces**

```cpp
namespace [[deprecated]] DraftAPI {
    void experimental();
}
```

Using `DraftAPI::experimental()` may trigger deprecation diagnostics.

### Attributes on **enumerators**

```cpp
enum City {
    Mumbai = 2,
    Bombay [[deprecated]] = Mumbai,
};
```

### Attribute namespace with `using` (book syntax, C++17)

C++17 generalizes attribute syntax so a **prefix** can introduce a namespace for a comma-separated list of attributes. Josuttis gives forms along this line:

```cpp
[[using MyLib: WebService, RestService, doc("html")]] void foo();
```

Here `MyLib` is treated as an **attribute namespace**; `WebService`, `RestService`, and `doc("html")` are attributes living in that vendor or library convention. Real compilers map such namespaces to their supported attribute schemes; until your toolchain documents a namespace, treat this as **illustrative syntax** from the standard's general attribute grammar.

Without `using`, the equivalent might require repeating `MyLib::` on each token; the prefix form batches them. For everyday code, prefer **standard attributes** (`[[nodiscard]]`, `[[maybe_unused]]`, `[[fallthrough]]`, `[[deprecated]]`) where they suffice.

Also see vendor spellings such as `[[gnu::...]]`, `[[clang::...]]`, `[[msvc::...]]` in compiler manuals.

---

## 7.5 Afternotes

Attributes in C++17 improve **portability of common warnings** that were previously compiler-specific pragmas (`__attribute__((warn_unused_result))`, etc.). The book positions them as part of making C++ **easier to teach** and **safer by default** through opt-in diagnostics.

---

## Summary table

| Attribute | Effect | Typical pitfall |
|-----------|--------|-----------------|
| `[[nodiscard]]` | warn on ignored return | not inherited on overrides |
| `[[maybe_unused]]` | suppress unused-name warn | cannot tag raw statements |
| `[[fallthrough]]` | suppress switch fallthrough warn | must be `[[fallthrough]];` |

---

## ASCII diagram: attribute placement (functions, simplified)

```
┌─────────────────────────────────────────────────────────────┐
│  [[nodiscard]] ReturnType function_name(Params) const;      │
│   ^ attribute before decl-specifiers                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ReturnType [[nodiscard]] function_name(Params) const;      │
│               ^ attribute after declarator-id (allowed)     │
└─────────────────────────────────────────────────────────────┘
```

(Exact parsing rules follow the C++ grammar; compilers accept common layouts.)
