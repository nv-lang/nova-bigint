# nova-bignum overview

**English** | [Русский](overview.ru.md)

`nova-bignum` is an arbitrary-precision numbers package for [Nova](https://nv-lang.org):
`BigInt` (unbounded integers), `BigRat` (exact rationals), `BigDecimal`
(arbitrary-precision decimal), and `BigFloat` (arbitrary-precision binary
floating point) — a counterpart to Python's `int`/`decimal`/`fractions`,
Rust's `num-bigint`/`num-rational`, and Go's `math/big`.

All four types are value-records: a copy is a small stack header (a sign
plus a slice/pointer), the limb/mantissa data lives on the GC heap and is
shared across copies, and immutability means no operation ever mutates a
value another binding still holds. `BigDecimal`, `BigFloat`, and `BigRat`
have no limb arithmetic of their own — every one of them is built on top of
`BigInt` and delegates its arithmetic to it.

## The family, at a glance

| Type | Representation | Precision | Typical use |
|---|---|---|---|
| [`BigInt`](bigint.md) | `sign Sign, limbs []u32` | Exact, unbounded | Integer math beyond `i64`/`u64`/`i128` — factorials, cryptography-adjacent arithmetic, big counters |
| [`BigRat`](bigrat.md) | `num BigInt, den BigInt` (irreducible) | Exact | Fractions that must never accumulate rounding error — ratios, exact rate math |
| [`BigDecimal`](bigdecimal.md) | `mant BigInt, scale int` (value = `mant × 10^{-scale}`) | Explicit, decimal | Money and anything decimal-facing where `f64`'s binary rounding (`0.1 + 0.2 != 0.3`) is unacceptable |
| [`BigFloat`](bigfloat.md) | `mant BigInt, exp int` (value = `mant × 2^exp`) | Explicit, binary | Extended-precision `f64`-like math — more significant bits than `f64`'s 53, still binary floating point |

### When to use which

- Need an integer wider than `i128`, with no fractional part ever? **`BigInt`.**
- Need a fraction that stays exact forever — no drift after a thousand
  operations? **`BigRat`.**
- Need decimal semantics (money, prices, anything a human reads in base 10)
  with an explicit, controllable rounding policy? **`BigDecimal`.**
- Need `f64`-like binary floating point but with more precision than 53
  significant bits, and you're fine reasoning in base 2? **`BigFloat`.**

Conversions between family members follow one rule everywhere: **exact
directions need no context; lossy directions require an explicit
`MathContext` (decimal) or `PrecisionContext` (binary)** — there is no
hidden rounding anywhere in the package. `BigInt → BigRat/BigDecimal` is
always exact; `BigDecimal → BigRat` and `BigFloat → BigRat` are exact
(a decimal or binary fraction is always representable as an exact
rational); `BigRat → BigDecimal`, `BigRat → BigFloat`, and
`BigDecimal ↔ BigFloat` are lossy and take a context.

## Minimal example

```nova
import bignum.{Sign, ParseNumberError}
import bignum.bigint.{BigInt, DivError}

ro a = 42.to_bigint()
ro b = "12345678901234567890".to_bigint()!!
assert(a + b > a)
```

## Module map

| Module | What it holds |
|---|---|
| `bignum` (root) | `Sign` (`Neg`/`Zero`/`Pos`), `ParseNumberError` — the shared parse-error enum used by every `str @to_*()` conversion across the whole family |
| `bignum.bigint` | `BigInt`, `DivError` |
| `bignum.bigdecimal` | `BigDecimal`, `RoundingMode`, `MathContext` |
| `bignum.bigfloat` | `BigFloat`, `PrecisionContext` |
| `bignum.bigrat` | `BigRat` |

## Errors as values

Two failure shapes recur across the whole family:

- **Division by zero.** `BigInt`/`BigRat` return it as a value —
  `Result[_, DivError]` with the single variant `DivisionByZero`.
  `BigDecimal`/`BigFloat` panic on it instead (parity with the primitive
  `int`/`f64` division-by-zero contract) — it is a design choice for those
  two types, not an oversight.
- **String parsing.** Every `str @to_bigint()`/`@to_bigdecimal()`/
  `@to_bigfloat()`/`@to_bigrat()` returns `Result[_, ParseNumberError]`,
  one shared enum for the whole package (`Empty`, `OnlySign`,
  `InvalidCharacter`, `MultiplePoints`, `MultipleExponents`,
  `EmptyExponent`, `EmptyMantissa`, `ZeroDenominator`) — no per-type
  variant or adapter.

`panic` is reserved for internal invariants and for the division-by-zero/
sqrt-of-negative contracts noted above — never for a value the caller
handed in through a `Result`-returning API.

## Contents of this doc set

- [bigint.md](bigint.md) — `BigInt`: representation, arithmetic, Karatsuba
  multiplication, Knuth Algorithm D division, bounds
- [bigdecimal.md](bigdecimal.md) — `BigDecimal`: scale, `RoundingMode`,
  `MathContext`
- [bigfloat.md](bigfloat.md) — `BigFloat`: mantissa/exponent, precision,
  `f64` conversions
- [bigrat.md](bigrat.md) — `BigRat`: exact fractions, normal form,
  bridges to `BigDecimal`/`BigFloat`

## Related documents

- [`README.md`](../README.md) — package pitch, install, full feature/limits list
- `docs/plans/235-bigint.md`, `236-bigdecimal.md`, `237-bigfloat.md`,
  `240-bigrat.md`, `243-*.md` in the `nova` repo — the design plans this
  package implements
- [`src/bigint/core_test.nv`](../src/bigint/core_test.nv),
  [`src/bigdecimal/core_test.nv`](../src/bigdecimal/core_test.nv),
  [`src/bigfloat/core_test.nv`](../src/bigfloat/core_test.nv),
  [`src/bigrat/core_test.nv`](../src/bigrat/core_test.nv) — the full test
  suites; every example in this doc set is adapted from one of them
