# BigInt

**English** | [Русский](bigint.ru.md)

`BigInt` is an arbitrary-precision signed integer — module `bignum.bigint`.
See [overview.md](overview.md) for how it relates to the rest of the
`bignum` family.

## Representation

```nova
type Sign enum Neg | Zero | Pos

type BigInt value {
    sign Sign
    limbs []u32
}
```

- Sign-magnitude: the sign lives in its own `Sign` field, the magnitude in
  `limbs` — the digits of the number in base 2³², least significant first
  (each element holds one 32-bit "digit"), with no leading zero digits.
- Zero is canonical: `sign: Zero, limbs: []` — the only representation of
  zero; every operation normalizes back to it.
- Value-record on the stack: a copy is a `Sign` enum tag plus a
  slice-header, roughly 24 bytes; the limb data itself lives on the GC
  heap and is shared across copies (immutability guarantees no alias
  mutates a shared buffer).
- `==` is structural (value-record equality, D328) and agrees with
  `@equal`/`@compare`.

## Constructing

```nova
ro z = BigInt.zero()
ro o = BigInt.one()

ro a = 42.to_bigint()
ro b = "12345678901234567890".to_bigint()!!
```

- `BigInt.zero()` / `BigInt.one()` — the two constants.
- `T @to_bigint()` for any `Ints` member (`i8..i64`, `u8..u64`, `int`,
  `uint`) — infallible, always exact.
- `i128 @to_bigint()` — the library's 128-bit type, also infallible.
- `str @to_bigint()` — decimal string → `Result[BigInt, ParseNumberError]`;
  accepts an optional leading `+`/`-`, rejects anything that isn't an
  ASCII digit after the sign.

## Predicates

```nova
assert(BigInt.zero().is_zero())
assert(!BigInt.one().is_zero())
assert(BigInt.one().is_one())
assert(((-1).to_bigint()).is_neg())
assert(42.to_bigint().is_even())
```

`@is_zero()`, `@is_neg()`, `@is_pos()`, `@is_one()`, `@is_even()`,
`@sign() -> Sign`.

## Comparison

```nova
assert(42.to_bigint() == 42.to_bigint())
assert(42.to_bigint() != 43.to_bigint())
assert(BigInt.one() > BigInt.zero())
assert((-BigInt.one()) < BigInt.one())
assert(100.to_bigint().compare(50.to_bigint()) > 0)
```

`@equal(other)` backs `==`/`!=`; `@compare(other) -> int` gives the full
signed `-1`/`0`/`1` ordering and backs `<`/`>`.

## Arithmetic

```nova
ro a = 2.to_bigint()
ro b = 3.to_bigint()
assert(a + b == 5.to_bigint())
assert(a - b == (-1).to_bigint())
assert(-a == (-2).to_bigint())
assert(a.abs() == a)
assert(a * b == 6.to_bigint())

ro (q, r) = 100.to_bigint().div_rem(3.to_bigint())!!
assert(q == 33.to_bigint())
assert(r == 1.to_bigint())
```

- `@plus`/`@minus`/`@neg`/`@abs`/`@times` — desugar to `+`/`-`/unary
  `-`/`*` respectively; all infallible and exact.
- `@div_rem(other) -> Result[(BigInt, BigInt), DivError]` — the one
  division primitive; `@div`/`@rem` are thin wrappers over it. **The `/`
  and `%` operators do NOT desugar to them** — division can fail (divide
  by zero), so it stays an explicit method call returning a `Result`.
  Truncates toward zero, remainder carries the sign of the dividend
  (parity with `int`/C99/Rust).
- `@gcd(other) -> BigInt` — binary Euclidean algorithm over `@rem`;
  result is always non-negative, `gcd(0, 0) == 0`.
- `@pow(n uint) -> BigInt` — binary exponentiation, `n >= 0`.

```nova
assert(2.to_bigint().pow(10) == 1024.to_bigint())
assert(48.to_bigint().gcd(36.to_bigint()) == 12.to_bigint())
```

### Bit operations

`@shl(bits int)` / `@shr(bits int)` — unsigned shift of the magnitude
(sign preserved); `@shr` truncates toward zero, matching `@div` by a
power of two. `@bit_length() -> int` — significant bits of the absolute
value (`0` for zero). `@digits() -> int` — decimal digit count (V1:
implemented via `@to_str()`).

```nova
ro x = BigInt.one().shl(100)
assert(x.bit_length() == 101)
assert(x.shr(100) == BigInt.one())
```

## String conversions

```nova
assert(42.to_bigint().to_str() == "42")
assert((-42).to_bigint().to_str() == "-42")

ro v = (12345678901234567890 as u64).to_bigint()
assert(v.to_str() == "12345678901234567890")
```

`@to_str() -> str` — canonical decimal representation, no leading zeros
(except `"0"` itself), sign printed only for negative values.

## Conversions back to fixed-width types

```nova
assert(1234567890.to_bigint().to_int() == Some(1234567890))
```

`@to_int() -> Option[int]` and `@to_i128() -> Option[i128]` — `Some` when
the value fits the target's range, `None` on overflow. There is no
truncating/wrapping variant in V1 — every narrowing is checked.

## Multiplication: Karatsuba

Multiplication uses the Karatsuba algorithm, `O(n^log₂3)` ≈ `O(n^1.585)`,
falling back to schoolbook `O(n²)` multiplication below a 16-limb
threshold (splitting below that size costs more than it saves). Both
paths are exercised against each other for equivalence — see
[`src/bigint/mul_equiv_test.nv`](../src/bigint/mul_equiv_test.nv).
Toom-3 is a V2 candidate, not implemented.

## Division: Knuth's Algorithm D

Division (`@div_rem`) implements Knuth's Algorithm D (TAOCP §4.3.1):
normalize the divisor so its most significant limb has its top bit set,
guess each quotient digit from the top two divisor limbs, correct the
guess when the trial subtraction borrows, then denormalize the
remainder. A single-limb divisor takes a dedicated fast path (plain
limb-by-limb long division, no normalization needed).

```nova
ro a = "1427247692705959881058285969449495137370400945".to_bigint()!!
ro b = "1208925819614629174706189".to_bigint()!!
ro (q, r) = a.div_rem(b)!!
```

## Limitations

- All allocations happen on the GC heap; every operation creates a new
  `BigInt` rather than mutating in place.
- There are no mutating `@add_assign`/`@mul_assign`-style methods in V1:
  since a copy shares the limb buffer, mutating it in place would corrupt
  a value another binding still holds — the value-record immutability
  guarantee would break. Buffer reuse is a V2 concern.
- `modpow` and bitwise `&`/`|` (as opposed to the shifts above) are not
  in V1.
- No implicit `int → BigInt` coercion and no integer-literal suffix form
  (e.g. `123456789012345678901n`) — every conversion is an explicit
  `@to_bigint()` call.

## Related documents

- [overview.md](overview.md) — the `bignum` family map and shared
  conventions (error handling, conversion rules)
- [bigdecimal.md](bigdecimal.md) — `BigDecimal`, built entirely on
  `BigInt` arithmetic
- [`src/bigint/core.nv`](../src/bigint/core.nv) — full source
- [`src/bigint/core_test.nv`](../src/bigint/core_test.nv),
  [`mul_equiv_test.nv`](../src/bigint/mul_equiv_test.nv) — full test suite
