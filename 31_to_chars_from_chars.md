# Chapter 31: `std::to_chars()` and `std::from_chars()`

Tutorial aligned with *C++17 - The Complete Guide* by Nicolai M. Josuttis. **Page references**: **pp. 385-391**.

Header: **`<charconv>`**. These functions provide **low-level**, **locale-independent** numeric parsing and formatting with **no allocations**. They are intended for **serialization**, **protocol** coding, **logging**, and hot paths where **`sprintf`**, **`iostreams`**, or **`std::to_string`** are too heavy (p. 385).

---

## 31.1 Motivation (p. 385)

### Design goals

| Aspect | `to_chars` / `from_chars` | Many alternatives |
|--------|-----------------------------|-------------------|
| Allocations | No heap (you pass buffer) | `to_string`, streams may allocate |
| Locale | **C locale** numeric rules | `sprintf` / streams locale-dependent |
| Errors | `std::errc` on result | Exceptions or parse flags vary |
| Speed | Tuned for tight loops | Often slower (p. 385) |

Josuttis contrasts these with **`sprintf` / `sscanf`**, **`stoi`**, **`to_string`**, and streams for both **performance** and **predictability** (p. 385).

---

## 31.2 Example usage (p. 386)

### Common includes

```cpp
#include <charconv>
#include <string>
#include <array>
#include <system_error>
```

---

## 31.2.1 `from_chars()` (p. 386)

### Result type

```cpp
struct from_chars_result {
    const char* ptr;
    std::errc ec;
};
```

- **`ptr`**: one past the last character consumed **on success**, or where parsing **stopped** / **invalid** (Josuttis clarifies error cases in context; see standard for full semantics).
- **`ec`**: `std::errc{}` (default-constructed = **success** / "no error" marker in this API) vs other **error codes**.

### Parsing integers

```cpp
#include <charconv>
#include <string>

bool parse_int_example(const std::string& str, int& out) {
    auto [ptr, ec] = std::from_chars(str.data(), str.data() + str.size(), out);
    if (ec != std::errc{}) {
        return false;  // [ERROR] overflow, invalid pattern, etc.
    }
    if (ptr != str.data() + str.size()) {
        // Trailing junk: decide policy
        return false;
    }
    return true;  // [OK] whole string consumed
}
```

### Base parameter (hex example)

```cpp
unsigned value = 0;
const char hex[] = "2A";
auto r = std::from_chars(hex, hex + sizeof(hex) - 1, value, 16);
// expect r.ec == std::errc{} and value == 42
```

**Bases:** typically **2-36** for integer parsing per standard (p. 386).

---

## 31.2.2 `to_chars()` (p. 387)

### Result type

```cpp
struct to_chars_result {
    char* ptr;
    std::errc ec;
};
```

On success, characters are written to **[buf, ptr)** -- **`ptr`** is **one past** the last written character (Josuttis, p. 387).

### Format an integer into a stack buffer

```cpp
#include <charconv>

void print_42_to_buffer() {
    std::array<char, 20> buf{};
    auto [ptr, ec] = std::to_chars(buf.data(), buf.data() + buf.size(), 42);
    if (ec != std::errc{}) {
        // [ERROR] buffer too small / other error
        return;
    }
    // valid range: [buf.data(), ptr)
}
```

### Base parameter

```cpp
std::array<char, 32> buf{};
int v = 255;
auto r = std::to_chars(buf.data(), buf.data() + buf.size(), v, 16);
// writes lowercase hex without "0x" prefix by default rules (verify flags/options)
```

---

## 31.3 Floating-point round-trip support (p. 388)

C++17 introduces **consistent** `to_chars` / `from_chars` for **floating point** in a way that supports **round-trip**: formatting with `to_chars` then parsing with `from_chars` recovers the **same** value for the **supported** formats and implementations meeting the standard guarantees (Josuttis, p. 388).

### `chars_format` enumeration

Typical values (see `<charconv>`):

- `std::chars_format::scientific`
- `std::chars_format::fixed`
- `std::chars_format::hex`
- `std::chars_format::general` (often default-like combination behavior)

**Example sketch:**

```cpp
#include <charconv>
#include <array>

double x = 1.375;
std::array<char, 64> buf{};

auto t = std::to_chars(buf.data(), buf.data() + buf.size(), x);
// pick format/precision as needed

if (t.ec == std::errc{}) {
    double y{};
    auto f = std::from_chars(buf.data(), t.ptr, y);  // parse what was written
    // Josuttis: round-trip guarantees within the specified formatting rules (p. 388)
}
```

**Practical note:** precision and format flags affect **shortest** vs **fixed** representations; Josuttis compares strategies relevant to **JSON**-like output and scientific dumps (p. 388).

---

## ASCII diagram: round-trip

```
  numeric value v  ──to_chars──>  [ buf .......... ]  (no allocations)
       ^                                   │
       │                                   │
       └──────── from_chars ───────────────┘
                 recover v'  where v' == v under standard guarantees (p. 388)
```

---

## ASCII diagram: performance comparison (conceptual)

```
Workload: format 1e6 integers

std::ostringstream / iostreams  ─────────────────────────►  slow
sprintf / snprintf            ─────────────────►        medium
to_chars (fixed buffer)       ─────►                     fast (typical; p. 385)

Legend:
  ─────►   higher time / more overhead

* Actual results depend on standard library, CPU, format, base.
```

---

## Error handling reminders

- **No exceptions** from `to_chars`/`from_chars` themselves -- check **`ec`** (p. 386-387).
- **Buffer too small** yields **`std::errc::value_too_large`** (typical) -- increase buffer or adapt (p. 387).
- **Integer overflow** on parse -> **`std::errc::result_out_of_range`** (typical).

---

## When not to use `charconv`

- Need **locale-specific** decimal separators or thousands separators -> use streams/`printf` family with locale.
- Need **exact** legacy format compatibility with arbitrary **`printf`** formats -- may still require `snprintf`.

---

## Reference

Josuttis: **Chapter 31**, **pp. 385-391**.
