# Chapter 18: `std::byte` (The Complete Guide, pp. 179-184)

This chapter follows *C++17 - The Complete Guide* by Nicolai M. Josuttis. Page references use **p.**

---

## 18.1 Using `std::byte` (p. 179)

### Raw memory byte, not arithmetic

`std::byte` models a single **raw byte** of storage. It is **not** a numeric integer type in the interface sense: you are not supposed to do `+` / `-` on bytes as if they were `int`. The intent is clearer code for **binary buffers**, bit masks, and low-level layouts.

### Relaxed enum initialization (C++17) (p. 179)

`std::byte` is a scoped enumeration backed by `unsigned char`. C++17 allows **list-initialization** from an integer **if** the conversion is to enum:

```cpp
#include <cstddef>

std::byte b{42};
std::byte b2{0xFF};
```

This is the idiomatic way to construct a byte with a known bit pattern.

### Initialization constraints (p. 180)

List initialization with `{…}` is the primary form; **direct initialization** `()` and **copy** forms `=` are ill-formed for the usual integer-to-byte path:

```cpp
std::byte b1{42};     // [OK]
// std::byte b2(42);  // [ERROR] (not list-init from int the way enum class allows)
// std::byte b3 = 42; // [ERROR] (no viable conversion)
// std::byte b4 = {42}; // [ERROR] same issue as copy-init from int

// std::byte b5[] {1};           // [ERROR] no implicit conversion from int
std::byte b6[] {std::byte{1}};  // [OK] each element explicitly std::byte

std::byte b_uninit;   // [OOPS] indeterminate value if automatic storage
std::byte b_zero{};   // [OK] value-initialized (zero)
```

### `sizeof` and bit width (pp. 180-181)

```cpp
static_assert(sizeof(std::byte) == 1);  // [OK] always 1 byte

#include <limits>
constexpr int bits =
    std::numeric_limits<std::underlying_type_t<std::byte>>::digits;
// Usually 8: same as unsigned char (via std::numeric_limits<unsigned char>::digits)
```

### ADL and `to_integer` (p. 180)

`std::to_integer` is **not** found by argument-dependent lookup when you call it **unqualified** with a `std::byte` argument—the usual "just write `to_integer`" pattern fails:

```cpp
// std::cout << to_integer<int>(b1) << '\n';  // [ERROR] ADL does not find std::to_integer

std::cout << std::to_integer<int>(b1) << '\n';  // [OK] fully qualified

// if (b2) { }  // [ERROR] no bool conversion
if (std::to_integer<bool>(b2)) { }  // [OK] explicit numeric path to bool
```

### Converting to integer: `std::to_integer<int>(b)` (p. 180)

When you need a numeric `int` (for logging, indexing, arithmetic on the *numeric* value):

```cpp
#include <type_traits>

int x = std::to_integer<int>(b);
```

Use the explicit conversion pathway rather than relying on unsafe C-style casts everywhere.

### Bitwise operations (p. 180)

`std::byte` supports:

- `|`, `&`, `^`, `~`
- `<<=`, `>>=` (and the non-assigning shifts on promoted operands per operator rules)

Example:

```cpp
std::byte a{0b1010'0000};
std::byte b{0b0000'1111};
std::byte c = a & b;
```

---

## 18.2 Types and operations (p. 181)

### Definition (p. 181)

Josuttis quotes the spirit of the standard definition:

```cpp
namespace std {
    enum class byte : unsigned char {};
}
```

So each `std::byte` has the **same size and alignment** as `unsigned char`.

### No general arithmetic (p. 181)

There are **no** `operator+`, `operator-`, `*`, `/` for `std::byte`. If you want numeric addition on byte values, convert via `to_integer`, add as integers, then convert back (with range checks if needed).

**Rationale** (as Josuttis explains): preventing accidental mixing of "byte oriented" logic with normal math, and preventing silent integral promotions that obscure bit logic.

### No implicit conversions (p. 182)

You cannot implicitly use `std::byte` where an `int` is expected. This forces explicit `to_integer` or an explicit cast, reducing bugs.

### I/O: cast for output (p. 182)

```cpp
#include <iostream>

std::byte b{  static_cast<std::byte>(42)  };
std::cout << std::to_integer<unsigned>(b) << '\n';
```

There is no `operator<<` for `std::byte` in the standard stream interface in the same way as for `int`; you choose the integer type whose representation you want to print (decimal, or print hex yourself).

### Binary I/O with `std::bitset` (p. 183)

Print the bit pattern of a byte (including leading zeros) via a `bitset` sized to the byte width:

```cpp
#include <bitset>
#include <limits>
#include <iostream>
#include <string>

using ByteBitset =
    std::bitset<std::numeric_limits<unsigned char>::digits>;

std::byte b1{0b0010'1010};
std::cout << ByteBitset{std::to_integer<unsigned>(b1)};  // e.g. 00101010
std::string s = ByteBitset{std::to_integer<unsigned>(b1)}.to_string();  // [OK] "00101010..."
```

### `std::to_chars` / `std::from_chars` and formatted binary (pp. 183-184)

`std::to_chars` can write a **binary** representation **without** leading zeros (output length varies). For **input**, a practical pattern is **`istream` + `bitset`** for a fixed-width binary field; converting back needs an explicit cast because list-initialization from `unsigned long` can **narrow** relative to `std::byte`:

```cpp
#include <charconv>
#include <array>
#include <bitset>
#include <istream>
// ...

std::array<char, 64> buf{};
auto r = std::to_chars(buf.data(), buf.data() + buf.size(),
                       std::to_integer<unsigned>(b1), 2);  // base 2, no leading zeros

// Example: input operator using bitset (fixed width = digits per byte)
std::istream& operator>>(std::istream& is, std::byte& out) {
    ByteBitset bs;
    if (is >> bs) {
        unsigned long v = bs.to_ulong();
        out = static_cast<std::byte>(v);  // [OK] explicit; list-init would narrow
    }
    return is;
}
```

(Josuttis ties these techniques to readable hex/binary dumps without treating `std::byte` as an arithmetic type.)

### Compound bitwise example (p. 183)

```cpp
std::byte b1{0x0F};
std::byte b2{0xF0};
std::byte b3 = b1 & b2;  // 0x00
```

---

## ASCII: byte operations

```
 Individual byte (8 bits on typical platforms)
 ┌───┬───┬───┬───┬───┬───┬───┬───┐
 │b7 │b6 │b5 │b4 │b3 │b2 │b1 │b0 │
 └───┴───┴───┴───┴───┴───┴───┴───┘
          ^
          interpreted as raw pattern, not "number" until to_integer

 AND (&): both bits 1 -> 1
 OR  (|): either 1 -> 1
 XOR (^): different -> 1
 NOT (~): flip bits (subject to type promotion rules in expressions)

 Shift (<<, >>): move pattern left/right; often combined with &= on larger buffers
```

**Contrast with `unsigned char` used as "small integer":**

```
 unsigned char: participates in arithmetic, promotes to int easily
 std::byte:     bitwise-focused API; explicit path to integer math
```

---

## When to use `std::byte` (summary)

- **Binary protocols**, **file headers**, **checksum regions**: operations are naturally bitwise.
- **Interoperability**: you still often need `reinterpret_cast` to/from `byte*` when interfacing with C APIs; keep such casts localized and documented (Josuttis warns about strict aliasing and alignment in neighboring chapters on low-level code).

---

## Small worked example: OR a mask into the first byte

`std::byte` already provides `operator|` in the standard library; use it directly:

```cpp
#include <cstddef>
#include <cstdint>

enum class Flags : std::uint32_t { Read = 1, Write = 2 };

void set_write(std::byte* word, std::size_t len_bytes) {
    if (len_bytes == 0) return;
    const std::byte mask{
        static_cast<std::byte>(static_cast<std::uint32_t>(Flags::Write))};
    word[0] = word[0] | mask;  // illustrative: lowest byte only
}
```

(In real code, endianness and endian-safe reads/writes matter; this only illustrates bitwise `|` on `std::byte`.)

---

## References in Josuttis

- **§18.1** (p. 179): motivation, list-init vs other forms, ADL/`to_integer`, `sizeof`/digits.
- **§18.2** (p. 181): precise operations; **I/O** (decimal, **bitset** binary strings, **`to_chars`** / stream patterns from pp. 183-184).

Use the printed index for cross-references to `std::vector<std::byte>`-style buffers in other parts of the book.
