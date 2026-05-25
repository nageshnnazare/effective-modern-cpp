# Chapter 5: Mandatory Copy Elision (Passing Unmaterialized Objects)

**Book reference:** *C++17 - The Complete Guide* by Nicolai M. Josuttis (pages 39-46)

Copy elision (formerly called the "named return value optimization" when named, and RVO-related behavior for temporaries) has a long history. **C++17** makes **copy elision mandatory** in specific situations involving **prvalues**, which changes what programs **must** compile even when copy and move constructors are **deleted**.

---

## 5.1 Motivation for Mandatory Copy Elision for Temporaries (p. 39)

### Before C++17: elision was optional

Compilers *could* elide copies/moves, but the program was still required to be **valid** if the compiler chose **not** to elide. Consequences:

- **Copy and move constructors** often had to **exist and be accessible**, even if never called in optimized builds.
- A class with **deleted** copy and move could **not** be returned by value from a factory-style function if the compiler needed those operations for the abstract machine.

### Since C++17: mandatory for certain prvalue cases

For the cases covered by the standard (returning prvalues, throwing prvalues, etc.), the implementation **does not** require a copy/move: the **temporary is constructed directly** in the destination when the rules apply.

**Important nuance:** **Named Return Value Optimization (NRVO)** is still **optional**. A function that returns a **named local** still conceptually needs **copy/move** available in general:

```cpp
MyClass foo() {
    MyClass obj;
    return obj;   // NRVO may apply, but [NOT mandatory]
                  // type may still need move/copy in some cases
}
```

So: **mandatory elision** [OK] for certain **prvalue** returns; **NRVO** still [optional].

---

## 5.2 Benefit of Mandatory Copy Elision for Temporaries (p. 41)

### Performance and abstraction

* **Performance:** You get the "no extra object" guarantee where the rule applies; no reliance on QoI.
* **Generic factories** become simpler and work with more types.

**Factory template (book style):**

```cpp
#include <utility>

template<typename T, typename... Args>
T create(Args&&... args) {
    return T{std::forward<Args>(args)...};
}
```

With mandatory elision, returning the **prvalue** `T{...}` can succeed even when `T` has **no** usable copy/move.

### Examples that become valid

**`std::atomic<int>`:** Typically not movable/copyable like an `int`; returning a prvalue `std::atomic<int>{42}` into the caller's object can work under C++17 rules where it failed before.

**Copy-only type with deleted move:**

```cpp
struct CopyOnly {
    CopyOnly() = default;
    CopyOnly(const CopyOnly&) = default;
    CopyOnly& operator=(const CopyOnly&) = default;
    CopyOnly(CopyOnly&&) = delete;
    CopyOnly& operator=(CopyOnly&&) = delete;
};

CopyOnly ret() {
    return CopyOnly{};   // [OK] since C++17 (prvalue return path)
}
```

---

## 5.3 Clarified Value Categories (p. 42)

### History: lvalue and rvalue (C)

C inherited a simple picture: some expressions refer to **objects with identity** (lvalues), others behave like **values** on the right-hand side (rvalues).

### C++11: five fine-grained categories

* **glvalue** ("generalized lvalue"): has identity, **refers to** an object or function.
* **prvalue** ("pure rvalue"): computes a **value**; typically no identity of its own until materialized.
* **xvalue** ("expiring value"): glvalue that could be moved from (often the result of `std::move`).

**rvalue** = **prvalue OR xvalue**.  
**lvalue** = glvalue that is not an xvalue (historically "plain" lvalue).

### ASCII diagram: C++11 hierarchy (conceptual)

```
                         expression
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
        ┌──────────┐                    ┌──────────┐
        │ glvalue  │                    │  rvalue  │
        │ (identity)                    │          │
        └────┬─────┘                    └────┬─────┘
             │                               │
     ┌──────┴──────┐                  ┌──────┴──────┐
     ▼             ▼                  ▼             ▼
┌────────┐   ┌────────┐       ┌────────┐   ┌────────┐
│ lvalue │   │ xvalue │       │ prvalue│   │ xvalue │
└────────┘   └────────┘       └────────┘   └────────┘
                │                               │
                └─────────── "rvalue" ──────────┘
```

### ASCII diagram: C++17 mental model (p. 44)

Same three **leaf** categories (`lvalue`, `xvalue`, `prvalue`) as in C++11, but the **explanatory axis** shifts:

```
              every expression has a value category
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   ┌─────────────────┐               ┌─────────────────┐
   │     glvalue     │               │     prvalue     │
   │  "has location" │               │ "initializes"   │
   │  (read / refer) │               │ (value only;    │
   │                 │               │  may stay       │
   │                 │               │  unmaterialized)│
   └────────┬────────┘               └────────┬────────┘
            │                                 │
            │ lvalue  vs  xvalue              │
            │ (non-     (expiring,             │
            │  movable  often from             │
   ┌────────┴────────┐  std::move)             │
   ▼                 ▼                         │
 glvalue splits as in C++11 tree above ◄──────┘
        prvalue ──materialization──► xvalue (temporary object)
```

**Reading Josuttis:** mandatory copy elision paths often keep a **prvalue** "in flight" until it **initializes** a result object; **materialization** is what happens when a glvalue context forces a real temporary.

### Examples (informal)

| Expression | Category | Note |
|------------|----------|------|
| Variable name `x` | lvalue | Has identity |
| String literal `"hi"` | lvalue | Array decays; literal has static storage |
| `42`, `3.14` | prvalue | Pure value |
| Temporary from `T()` | prvalue | Until materialized |
| `std::move(x)` | xvalue | Treats object as movable |

---

## 5.3.2 Value Categories Since C++17 (p. 44)

### Refined intuition

* **glvalues:** expressions that refer to **locations** (something in memory or a subobject model).
* **prvalues:** expressions that **initialize**; they **compute a value** and can often exist **without** a named temporary in the abstract machine until needed.

### Materialization (prvalue [->] xvalue)

When a prvalue is used where a glvalue is required, the language **materializes** a temporary object: prvalue [->] **temporary object** [->] treated as xvalue.

**Classic example:** `void f(X&);` cannot bind to a prvalue; `f(X());` **materializes** a temporary `X` and binds a reference to it (in earlier standards this was described differently; C++17 unified temporary creation with materialization).

```
   prvalue expression  (e.g. return value object, literal object init)
            │
            │  "needs a location" context (e.g. binding to reference,
            │   accessing member, some glvalue rules)
            ▼
   ┌────────────────────┐
   │  MATERIALIZATION   │
   └─────────┬──────────┘
             ▼
   temporary object in memory (has identity now)
             │
             ▼
   glvalue (xvalue) referring to that temporary
```

### Call example

```cpp
void f(X x);
f(X());  // prvalue X() may be passed without extra copy/move
        // (subject to passing rules); when a temporary object exists,
         // it is an xvalue/gvalue context as appropriate
```

The book emphasizes **unmaterialized return value passing**: the return path can construct the result **directly** in the caller's slot.

---

## 5.4 Unmaterialized Return Value Passing (p. 45)

### ASCII diagram: copy elision vs NRVO (combined flow)

```
  prvalue return:  return T{args};
        │
        ▼
   ┌─────────────┐      mandatory ──► caller's result object
   │ no named    │      elision         (T constructed once)
   │ temporary   │
   └─────────────┘

  named return:    T x; ...; return x;
        │
        ├── NRVO hits ──► no move/copy (optional optimization)
        │
        └── NRVO misses ──► move or copy (must be valid if selected)
```

### What can be "unmaterialized"

Situations discussed in the book include:

* Returning **non-string literals** as prvalues where applicable.
* Returning **temporary objects** as prvalues.
* `decltype(auto)` and how the **value category** of the return expression propagates.

**Key point:** When returning a **prvalue** of type `T`, you **need not** have copy/move operations; the object is not "moved out of" a temporary in the older sense -- the **result object** is initialized directly.

### Copy elision flow (return)

```
  callee:  return T{args...};
              │
              │ prvalue of type T
              ▼
  ┌───────────────────────────────┐
  │ Mandatory elision context     │
  │ (return statement, prvalue)   │
  └───────────────┬───────────────┘
                  │
                  ▼
  caller's result object memory
  (e.g. variable or temporary in caller)
                  ▲
                  │
        T constructed HERE
        (no intermediate copy/move)
```

**NRVO (named local) flow** remains **[optional]**:

```
  callee:  T obj; ...; return obj;
              │
              │ named lvalue
              ▼
  [MAY elide] ──────────────────► caller's object
       │
       │ if no elision
       ▼
  move/copy (must be valid if required by overload resolution)
```

---

## Summary table

| Scenario | Elision | Copy/move needed? |
|----------|---------|-------------------|
| `return T{};` (prvalue) | Mandatory (C++17) | No |
| `return local;` (named) | Optional (NRVO) | Often yes if no NRVO |
| Parameter pass prvalue | Per overload/init rules | Case-dependent |

---

## Practical advice

1. Prefer **prvalue returns** (`return T{...};`) for factories when you want maximum portability with **non-movable** types.
2. Do **not** assume NRVO; use move or explicit out-parameters if you must support types without move and named locals.
3. Teach your team **`std::is_copy_constructible` is the wrong gate** for "can I return this?" in many C++17 patterns -- understand **prvalue** semantics instead.

---

## Afternotes (chapter context)

The standard wording shift around **prvalues initializing objects** and **guaranteed copy elision** is one of the most impactful C++17 core language changes for **move-less** and **wrapper** types. Josuttis ties this to clearer **value category** teaching and to real-world library design (e.g. factories, atomics, immovable handles).
