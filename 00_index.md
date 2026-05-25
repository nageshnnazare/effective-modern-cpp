# C++17 - The Complete Guide: Master Index

This document is a **reader and study companion** for the first edition of *C++17 - The Complete Guide* by Nicolai M. Josuttis. Page references use the numbering of the printed **first edition** (452 pages, publication date **2019-12-20**). Minor shifts of plus or minus a few pages can occur between printings; treat ranges as **approximate** outside the explicitly cited front sections of Part I.

---

## Book metadata

| Field | Value |
|-------|-------|
| Title | C++17 - The Complete Guide |
| Edition | First Edition |
| Author | Nicolai M. Josuttis |
| Publication date | 2019-12-20 |
| Approximate length | 452 pages |
| Focus | New C++17 **language** and **library** features for practitioners |

---

## High-level structure (six parts)

The book groups features into six parts. The diagram below shows how they fit together.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      C++17 - The Complete Guide                              │
│                         (452 pages, first edition)                           │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│ Part I          │       │ Part II         │       │ Part III        │
│ Basic Language  │  ──►  │ Template        │  ──►  │ New Library     │
│ Ch. 1-8         │       │ Features        │       │ Components      │
│ pp. 1-74        │       │ Ch. 9-14        │       │ Ch. 15-20       │
│                 │       │ pp. 75-132      │       │ pp. 133-248     │
└────────┬────────┘       └────────┬────────┘       └────────┬────────┘
         │                        │                          │
         │    ┌───────────────────┴───────────────────┐      │
         │    │                                       │      │
         ▼    ▼                                       ▼      ▼
┌─────────────────┐                           ┌─────────────────┐
│ Part IV         │                           │ Part V          │
│ Library Ext.    │──────────────────────────►│ Expert          │
│ & Modifications │                           │ Utilities       │
│ Ch. 21-28       │                           │ Ch. 29-33       │
│ pp. 249-336     │                           │ pp. 339-404     │
└────────┬────────┘                           └────────┬────────┘
         │                                             │
         │                    ┌────────────────────────┘
         │                    │
         ▼                    ▼
           ┌─────────────────┐
           │ Part VI         │
           │ Final General   │
           │ Hints           │
           │ Ch. 34-35       │
           │ pp. 405-412     │
           └─────────────────┘
```

**Note on the page gap between Part IV and Part V:** the printed layout can include **front matter offsets, blank pages, or part-opening pages** between roughly **pp. 337-338** and **p. 339**. The ranges below follow the part boundaries stated in the book's sectional organization.

---

## Companion files in this folder (numbered tutorials)

| File | Chapter focus |
|------|----------------|
| `01_structured_bindings.md` | Ch. 1 - Structured Bindings |
| `02_if_switch_init.md` | Ch. 2 - `if` and `switch` with initialization |
| `03_inline_variables.md` | Ch. 3 - Inline variables |

Additional chapters are covered only in this master index for now.

---

## Part I - Basic Language Features (Chapters 1-8, pages 1-74)

### Chapter 1 - Structured Bindings (approximately pp. 3-19)

**Subtopics**

- Syntax `auto [a, b, c] = expr;` and the **anonymous object** model behind it.
- Qualifiers: `const auto&`, `auto&`, `auto&&`, and alignment (`alignas`) applied to the backing object.
- Non-decay behavior for arrays versus `auto`.
- Move semantics: `auto` vs `auto&&` structured bindings.
- Supported sources: `struct`/`class` (public members), inheritance limitations, raw arrays, `std::pair`, `std::tuple`, `std::array`.
- Integrating with **`std::tie`** for assignment into existing objects.
- **Tuple-like protocol**: `std::tuple_size`, `std::tuple_element`, `get` overload set; `if constexpr`; `decltype(auto)` for read vs write access.
- Afternotes: proposal history (Herb Sutter, Bjarne Stroustrup, Gabriel Dos Reis).

See `01_structured_bindings.md`.

---

### Chapter 2 - `if` and `switch` with Initialization (approximately pp. 21-24)

**Subtopics**

- `if (init; condition)` lifetime and visibility rules across **then** and **else**.
- Typical idioms: scoped logging streams, mutex guards, avoiding accidental **unnamed temporaries** with RAII.
- Multiple declarations in the init statement; combination with **structured bindings** and associative-container `insert`.
- `switch (init; condition)`; **name visibility** for the whole `switch`.
- Afternotes: Thomas Koppe.

See `02_if_switch_init.md`.

---

### Chapter 3 - Inline Variables (approximately pp. 25-31)

**Subtopics**

- Motivation: **ODR** issues for `static` data members and header-defined globals.
- `inline` data members at namespace scope and in classes; relationship to **inline functions**.
- `constexpr static` data members as **definitions** in C++17.
- `inline` with **`thread_local`** storage; interaction between global, thread-local, and instance storage.
- Afternotes: David Krauss (motivation), Hal Finkel and Richard Smith (proposal).

See `03_inline_variables.md`.

---

### Chapter 4 - Aggregates and Base Classes (approximately pp. 32-39)

**Subtopics**

- Extended aggregate rules in C++17: aggregates **with base classes**.
- When braces-initialization traverses base subobjects.
- Interaction with **structured bindings** (only direct members of one class; see Ch. 1).

---

### Chapter 5 - Guaranteed Copy Elision and Materialization (approximately pp. 40-48)

**Subtopics**

- **Mandatory copy elision** in additional contexts (prvalue passing/return).
- Temporary **materialization** rules as they affect lifetimes and idioms.
- Relationship to **move** optimization and `return` idioms.

---

### Chapter 6 - Nested Namespaces, Diagnostics, Includes, and Order (approximately pp. 49-58)

**Subtopics**

- Nested namespace definitions with **`namespace A::B::C { ... }`** syntax.
- **`static_assert`** accepting a single argument (message optional in C++17 with new overloading).
- **`__has_include`** preprocessing feature.
- Refinements to **expression evaluation order** for certain new cases in C++17.

---

### Chapter 7 - Enumerations, Initialization, Exceptions, and Allocation (approximately pp. 59-66)

**Subtopics**

- Relaxed **scoped enum** initialization from the underlying type where permitted.
- Fixes for list-initialization with **`auto`**.
- **`noexcept`** as part of the **function type** (type system impact).
- **`new`/`delete`** features for **over-aligned** data.

---

### Chapter 8 - Literals and Attributes (approximately pp. 67-74)

**Subtopics**

- UTF-8 character literals (**`u8`** / `char` UTF-8 literal rules as of C++17).
- **Hexadecimal floating-point** literals.
- New standard attributes: **`[[nodiscard]]`**, **`[[fallthrough]]`**, **`[[maybe_unused]]`**.
- Attribute placement extensions: attributes on **enumerators** and **namespaces**, and the **`using`** prefix for attributes.

---

## Part II - Template Features (Chapters 9-14, pages 75-132)

### Chapter 9 - Type Traits: Overview and General Improvements (approximately pp. 75-84)

**Subtopics**

- Why type traits matter for generic code in C++17.
- `_v` variable templates recap and consistency with C++14.
- General library hygiene improvements affecting traits and `<type_traits>`.

---

### Chapter 10 - `is_aggregate`, Swapping Traits (approximately pp. 85-93)

**Subtopics**

- **`std::is_aggregate`**: detecting aggregate types under new rules.
- **`std::is_swappable`** and **`std::is_swappable_with`**: modeling `swap` compatibility.

---

### Chapter 11 - Invocability and Result Traits (approximately pp. 94-103)

**Subtopics**

- **`std::is_invocable`**, **`std::is_invocable_r`**.
- **`std::invoke_result`** (replaces **`result_of`** idioms in new code).

---

### Chapter 12 - `has_unique_object_representation` (approximately pp. 104-110)

**Subtopics**

- When bitwise copying is meaningful.
- Relationship to padding, triviality, and hashing strategies.

---

### Chapter 13 - `conjunction`, `disjunction`, `negation` (approximately pp. 111-121)

**Subtopics**

- Short-circuiting metafunctions in the library.
- Composing trait predicates without instantiating impossible types.

---

### Chapter 14 - Template Support Topics and Migration Notes (approximately pp. 122-132)

**Subtopics**

- Practical patterns combining multiple traits.
- Replacing deprecated trait machinery in legacy codebases.

---

## Part III - New Library Components (Chapters 15-20, pages 133-248)

### Chapter 15 - `std::optional` (approximately pp. 133-150)

**Subtopics**

- Modeling **might-be-absent** values without pointers.
- Monadic-style idioms: `value()`, `value_or`, `operator*`, `operator->`.
- Exception behavior and **in-place** construction.

---

### Chapter 16 - `std::variant` (approximately pp. 151-168)

**Subtopics**

- Safe discriminated unions.
- Valueless-by-exception state; exception guarantees.
- Visitors and `std::get`; `std::holds_alternative`.

---

### Chapter 17 - Polymorphism with `std::variant` (approximately pp. 169-178)

**Subtopics**

- Encoding open sets with `variant` plus `visit`.
- Comparison to classical inheritance and to `union` idioms.

---

### Chapter 18 - `std::string_view` (approximately pp. 179-193)

**Subtopics**

- Non-owning string references.
- Lifetime pitfalls (dangling views).
- Interoperation with `std::string` and C strings.

---

### Chapter 19 - Designing String Interfaces with `string` and `string_view` (approximately pp. 194-207)

**Subtopics**

- Choosing parameters: by `string_view` vs `const std::string&`.
- Return types: when ownership must transfer.

---

### Chapter 20 - `std::filesystem` (approximately pp. 208-248)

**Subtopics**

- **`path`**, **`file_status`**, **`directory_iterator`**, **`recursive_directory_iterator`**.
- Normalization, absolute/relative paths, permissions.
- Portability notes vs Boost.Filesystem and the filesystem TS.

---

## Part IV - Library Extensions and Modifications (Chapters 21-28, pages 249-336)

### Chapter 21 - Parallel STL Introduction (approximately pp. 249-262)

**Subtopics**

- Execution policies: **sequenced**, **parallel**, **parallel unsequenced**.
- Correctness requirements on iterators and callable objects.

---

### Chapter 22 - Parallel Algorithms: `for_each_n`, Reductions, Scans (approximately pp. 263-276)

**Subtopics**

- **`std::for_each_n`**.
- **`reduce`**, **`transform_reduce`**, **`inclusive_scan`**, **`exclusive_scan`**, and transform variants.

---

### Chapter 23 - `clamp`, `sample`, `size`, `empty`, `data`, `as_const` (approximately pp. 277-286)

**Subtopics**

- Bounds clamping and random **sampling**.
- Uniform free functions for generic code over containers and arrays.

---

### Chapter 24 - Container Improvements (approximately pp. 287-298)

**Subtopics**

- **Splicing** for associative/unordered containers; **merge** patterns.
- Incomplete type support where extended.
- **`insert_or_assign`**, **`try_emplace`**, **`emplace` fixes**.

---

### Chapter 25 - Exception Specifications, String Conversions, `string::data` (approximately pp. 299-310)

**Subtopics**

- Stronger **`noexcept`** guarantees on selected operations.
- Elementary numeric conversions for strings.
- Non-const **`data()`** for `std::string`.

---

### Chapter 26 - Numeric Extensions (approximately pp. 311-322)

**Subtopics**

- **`gcd`**, **`lcm`**, three-argument **`hypot`**.
- Special math functions introduced or extended in C++17 (as covered in the book's numeric chapter).

---

### Chapter 27 - Multi-Threading Extensions (approximately pp. 323-332)

**Subtopics**

- **`std::shared_mutex`** and reader-writer patterns.
- **`std::scoped_lock`** variadic locking.
- Atomic **`is_always_lock_free`** and **hardware interference size** queries.

---

### Chapter 28 - Deprecated and Removed Features (approximately pp. 333-336)

**Subtopics**

- Removed **dynamic exception specifications**, **`register`**, **`auto_ptr`**, etc.
- Guidance for mechanical upgrades.

---

## Part V - Expert Utilities (Chapters 29-33, pages 339-404)

### Chapter 29 - Advanced Lifetime and Pointer Optimizations (approximately pp. 339-351)

**Subtopics**

- **`std::launder`** and strict aliasing interactions (as presented in the book).
- Low-level object models relevant to custom allocators.

---

### Chapter 30 - Class Template Argument Deduction (CTAD) (approximately pp. 352-364)

**Subtopics**

- Deduction guides for class templates.
- Interaction with constructors and `initializer_list`.

---

### Chapter 31 - `constexpr` Templates in Practice (approximately pp. 365-377)

**Subtopics**

- **`if constexpr`** in template code paths.
- Compile-time branch pruning idioms.

---

### Chapter 32 - Fold Expressions and Variadic Templates (approximately pp. 378-390)

**Subtopics**

- Unary and binary folds.
- Translating recursive metaprograms to folds.

---

### Chapter 33 - Expert Library Interop and Catch-All Topics (approximately pp. 391-404)

**Subtopics**

- Cross-feature interactions highlighted in the closing technical chapters (for example, combining traits, `optional`, parallel algorithms, and filesystem in realistic components).

---

## Part VI - Final General Hints (Chapters 34-35, pages 405-412)

### Chapter 34 - Adopting C++17 in Real Projects (approximately pp. 405-408)

**Subtopics**

- Compiler and standard library coverage notes (as of the book's publication horizon).
- Migration strategies: enabling flags, incremental adoption, library ripples.

---

### Chapter 35 - Summary Outlook (approximately pp. 409-412)

**Subtopics**

- Consolidation of themes across the book.
- Pointers toward **C++20** directions as discussed in the closing remarks.

---

## Suggested reading order inside Part I

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Ch. 1        │ ──► │ Ch. 2        │ ──► │ Ch. 3        │
│ Structured   │     │ if/switch    │     │ Inline       │
│ bindings     │     │ init         │     │ variables    │
└──────────────┘     └──────────────┘     └──────────────┘
        │                                           │
        └───────────────────┬───────────────────────┘
                            ▼
                   Later chapters build on
                   brace-init, RAII, and ODR
```

---

## How to use this index with the book

1. Read the **book chapter** for the authoritative standard wording, edge cases, and the author's commentary.
2. Use this folder's **detailed notes** (`01_...`, `02_...`, `03_...`) for expanded examples and diagrams.
3. Cross-check **page numbers** against your physical or PDF copy if a reference does not match; use the **section titles** as the stable key.

---

## Reference

Nicolai M. Josuttis, *C++17 - The Complete Guide*, first edition, published 2019-12-20, approximately 452 pages.

Public companion material and example tables of contents (subject to change without notice): https://www.cppstd17.com/
