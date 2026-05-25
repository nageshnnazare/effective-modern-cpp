# Chapter 24: Substring and Subsequence Searchers

*Reference: Nicolai M. Josuttis, **C++17 - The Complete Guide**, Chapter 24, pp. 293-298.*

Searchers encapsulate preprocessing of the **pattern** so repeated searches in a **text** reuse auxiliary tables (Boyer-Moore / Horspool). They integrate with `std::search` via overloads that take a **searcher object**.

---

## 24.1 Using Substring Searchers (p. 293)

Headers: `<functional>` (searcher types), `<algorithm>` (`search`).

### Searcher types

| Searcher | Idea |
|----------|------|
| `std::default_searcher` | Classic forward algorithm (delegates to `search` / naive-style behavior; baseline). |
| `std::boyer_moore_searcher` | Boyer-Moore with bad-character + good-suffix heuristics; strong on large alphabets / long patterns. |
| `std::boyer_moore_horspool_searcher` | Simplified BM variant (bad-character style only); often good performance / less preprocessing. |

All are function objects with call operator matching `search` expectations.

---

## 24.1.1 Using with `search()` (p. 293)

```cpp
#include <string>
#include <algorithm>
#include <functional>

int main() {
    std::string text = "abcdaababcabcd";
    std::string pat  = "abcabcd";

    auto pos = std::search(
        text.begin(), text.end(),
        std::boyer_moore_searcher(pat.begin(), pat.end()));

    if (pos != text.end()) {
        // match starts at pos (iterator to first element of occurrence)
    }
}
```

**Why use a searcher?**

- When the same **pattern** is reused across many texts (or large corpus), preprocessing cost amortizes.
- For one-off tiny patterns, `default_searcher` or plain `search` may be simpler / competitive.

---

## 24.1.2 Using Searchers Directly (p. 295)

Searchers are **callable** as `searcher(haystack_first, haystack_last)` and return a **pair** of iterators delimiting the match (or `{last, last}` if none).

### Structured bindings

```cpp
#include <string>
#include <functional>

bool contains_with_bm(std::string_view text, std::string_view pat) {
    std::boyer_moore_searcher bm(pat.begin(), pat.end());
    auto [beg, end] = bm(text.begin(), text.end());
    return beg != end;
}
```

### Repeated scans with `std::tie`

**Direct call** (same as using `std::search` with a searcher): `auto [beg, end] = searcher(text.begin(), text.end());` — `[OK]` if `beg != end`, `[ERROR]` match is empty when `beg == end` (often both equal `text.end()`).

**`for` loop with structured bindings and `std::tie` (advance `beg` from prior `end`):**

```cpp
#include <string>
#include <functional>
#include <tuple>

void find_all_for(std::string_view text, std::string_view pat) {
    std::boyer_moore_searcher bms(pat.begin(), pat.end());
    for (auto [beg, end] = bms(text.begin(), text.end());
         beg != text.end();
         std::tie(beg, end) = bms(end, text.end())) {
        // use match [beg, end)
        (void)beg;
        (void)end;
    }
}
```

```cpp
#include <iostream>
#include <string>
#include <functional>
#include <tuple>

void find_all(std::string_view text, std::string_view pat) {
    std::boyer_moore_horspool_searcher hp(pat.begin(), pat.end());
    auto first = text.begin();
    auto last  = text.end();

    while (true) {
        auto sub = std::tie(first, last); // not needed if using structured bindings each iteration
        auto beg_end = hp(first, last);
        auto beg = beg_end.first;
        auto end = beg_end.second;
        if (beg == end) break;
        std::cout << "match at offset " << (beg - text.begin()) << '\n';
        first = beg + 1; // continue search (overlapping) -- adjust per requirements
    }
}
```

**Caution:** advancing `first` defines whether you allow overlapping matches.

---

## 24.2 General Subsequence Searchers (p. 296)

`std::search` with searchers generalizes beyond `char`: any **forward iterator** with an equality notion works. Patterns and texts can be arrays, `deque`, custom iterators---subject to searcher constructor storing copies or referencing iterator ranges **(lifetime!)**.

**Lifetime pitfall:**

```cpp
// [ERROR] pattern iterators dangle if pat is destroyed before searcher use
auto make_bad() {
    std::string pat = "abc";
    return std::boyer_moore_searcher(pat.begin(), pat.end());
}
```

Store pattern data **outliving** the searcher, or use `string_view` carefully with contiguous storage lifetime.

---

## 24.3 Searcher Predicates (p. 297)

Searchers accept optional **hash** and **equality** callables for non-byte types or case-insensitive compares.

### Case-insensitive search sketch

```cpp
#include <functional>
#include <algorithm>
#include <cctype>
#include <string>
#include <string_view>

struct ci_equal {
    bool operator()(char a, char b) const noexcept {
        return std::tolower(static_cast<unsigned char>(a)) ==
               std::tolower(static_cast<unsigned char>(b));
    }
};

struct ci_hash {
    std::size_t operator()(char c) const noexcept {
        return std::hash<char>{}(static_cast<char>(
            std::tolower(static_cast<unsigned char>(c))));
    }
};

bool contains_ci(std::string_view text, std::string_view pat) {
    std::boyer_moore_searcher bm(
        pat.begin(), pat.end(),
        ci_hash{},
        ci_equal{});
    auto [b, e] = bm(text.begin(), text.end());
    return b != e;
}
```

**Notes:**

- Hash and equality must be **consistent** (`eq(a,b)` implies `hash(a)==hash(b)` for the domain you use).
- For `char` Boyer-Moore, custom hash defines how skips are computed under your equality semantics.

---

## ASCII: complexity comparison (typical / qualitative)

```
Legend:
  n = text length
  m = pattern length
  s = alphabet size (matters for BM bad-character tables)

┌─────────────────────────┬──────────────────────────────────────┐
│ Naive / default forward │ O(n*m) worst case, simple            │
├─────────────────────────┼──────────────────────────────────────┤
│ Boyer-Moore             │ Often sublinear scans; O(n*m) worst  │
│                         │ but strong practical on long pat     │
├─────────────────────────┼──────────────────────────────────────┤
│ Boyer-Moore-Horspool    │ Simpler tables; good typical perf    │
│                         │ depends on data / pattern structure  │
└─────────────────────────┴──────────────────────────────────────┘

        n grows ────────────────────────────────►
        preprocessing cost of BM may dominate when m is tiny
                       │
                       ▼
              measure on your workload (Josuttis emphasizes practice:
              Chapter 24 pp. 293-298)
```

---

## When to choose which

| Scenario | Suggestion |
|----------|------------|
| One-shot, small `m` | `default_searcher` or `search` |
| Large `m`, many scans | `boyer_moore_searcher` |
| Want simpler tables, still BM flavor | `boyer_moore_horspool_searcher` |
| Need locale-aware compare | prefer higher-level APIs; custom predicates must encode rules |

---

## Cross-reference

- `std::search` general interface: `<algorithm>`.
- Parallel algorithms (Chapter 22) are orthogonal; substring searchers are sequential iterator-based utilities.
