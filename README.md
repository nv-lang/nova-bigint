**English** | [Русский](README.ru.md)

# nova-bignum

Arbitrary-precision numbers (BigInt/BigRat/BigDecimal/BigFloat) in pure
Nova — a counterpart to Python's `int`/`decimal`/`fractions`, Rust's
`num-bigint`/`num-rational`, and Go's `math/big`.

## Features (V1)

- **Arithmetic:** `+`, `-`, `*`, `/`, `%`, `div_rem`, `pow`
- **Comparison:** `==`, `!=`, `<`, `>`, `compare`
- **Sign/magnitude:** `sign`, `is_zero`, `is_neg`, `is_one`, `neg`, `abs`
- **Number theory:** `gcd` (binary Stein's algorithm)
- **Conversions:**
  - `T @to_bigint()` — any `Ints` member (`i8`..`i64`, `u8`..`u64`)
  - `i128 @to_bigint()` — the library's 128-bit type
  - `str @to_bigint()` — decimal string → `Result[BigInt, ParseNumberError]`
  - `BigInt @to_int()` → `Option[int]`
  - `BigInt @to_i128()` → `Option[i128]`
  - `BigInt @to_str()` → decimal string
- **Errors as values:** division by zero → `Err(DivisionByZero)`, string
  parsing → `Err(ParseNumberError)` with a category (`Empty`,
  `InvalidCharacter`, `OnlySign`, ...). One `ParseNumberError` for the whole
  package — shared across BigInt/BigRat/BigDecimal/BigFloat (Plan 243
  Ф.U/У.1). `panic` only for internal invariants.

## Not in V1

- `modpow`, bitwise operations (`&`, `|`, `<<`, `>>`)
- Implicit `int → BigInt` coercions
- The literal form `123456789012345678901n`
- Toom-3 multiplication

Everything listed is a V2 candidate, subject to separate decisions.

## BigDecimal (V1) — arbitrary-precision decimal

`type BigDecimal value { mant BigInt, scale int }` — value = `mant ×
10^{-scale}`. Built on top of BigInt (Plan 236,
`docs/plans/236-bigdecimal.md` in the `nova` repo); it has no limb
arithmetic of its own — all arithmetic goes through `BigInt`.

- **Arithmetic:** `+`/`-`/`*` (desugar to `@plus`/`@minus`/`@times`),
  `@neg`, `@abs`. The `/` operator does NOT desugar — `@div(other,
  MathContext)` requires explicit precision.
- **Comparison/equality:** `@compare` (aligns scale, no normalization),
  `@equal`/`@hash` (normalize both operands — parity with Rust's
  `bigdecimal`: `1.0 == 1`).
- **Rounding:** `RoundingMode`
  (`HalfEven|HalfUp|HalfDown|Down|Up|Ceiling|Floor`), `MathContext {
  precision int, rm RoundingMode }`, `@round(ctx)` (significant digits),
  `@rescale(target, rm)` (decimal places; `target < 0` rounds to
  `10^|target|`).
- **Conversions:** `T @to_bigdecimal()` (all `Ints` members + `i128`),
  `str @to_bigdecimal() -> Result[BigDecimal, ParseNumberError]`,
  `@to_str(scale_pad = 0)`, `@to_int()`/`@to_i128() -> Option[...]`.
- **Normalization:** `@normalize()` — strips trailing zeros from the
  mantissa; **lazy** — not invoked by constructors/operations, only by
  `@equal`/`@hash`/explicitly.
- Division by zero — panic (parity with `int`, D423), not a `Result`.

### Not in BigDecimal V1

`pow`/`sqrt`, the `/` operator, mutable `@*_assign` methods, chained
operations with auto-carried precision, integration with generic numeric
type-sets (D310), implicit coercions/literals, a small-powers-of-10 cache,
Toom-Cook.

```nova
import bignum.bigdecimal.{BigDecimal, MathContext, HalfEven}

ro price = "19.99".to_bigdecimal()!!
ro tax = "0.01".to_bigdecimal()!!
assert(price.plus(tax).to_str() == "20.00")     // not 20.000000000000004, like f64

ro third = (1).to_bigdecimal().div((3).to_bigdecimal(), MathContext.new(4, HalfEven))
assert(third.to_str() == "0.3333")
```

## BigFloat (V1) — arbitrary-precision binary floating point

`type BigFloat value { mant BigInt, exp int }` — value = `mant × 2^exp`
(Plan 237; analogous to MPFR/`big.Float`). Precision is set explicitly via
`PrecisionContext { precision int, rm RoundingMode }`.

- **Arithmetic:** `@plus`/`@minus`/`@times`/`@div`/`@sqrt` — all take an
  explicit `PrecisionContext`; `@neg`/`@abs` are exact; `@round(prec, rm)`.
- **Comparison/equality:** `@compare` (exact, no rounding), `@equal`/`@hash`
  (normal form).
- **Conversions:** `T @to_bigfloat(ctx)` (all `Ints` members + `i128` —
  exact; `f64` — exact, no ctx needed; `str` — decimal and `0b`-binary →
  `Result[BigFloat, ParseNumberError]`), `@to_f64()`/`@to_int()`/
  `@to_i64()`/`@to_u64() -> Option[...]`, `@to_bigint()` (trunc),
  `@to_str(frac_digits = 6)`, a `BigDecimal <-> BigFloat` bridge (via ctx).
- Division/`sqrt` at zero — panic (parity with `int`, D423).

## BigRat (V1) — exact rational numbers

`type BigRat value { num BigInt, den BigInt }` — an irreducible fraction,
`den > 0`, zero is canonically `0/1` (Plan 240; analogous to
`big.Rat`/`num-rational`). Every operation returns normal form — `@equal`
is structural, with no alignment step.

- **Arithmetic:** `@plus`/`@minus`/`@times` (exact, no rounding),
  `@div`/`@recip` → `Result[BigRat, DivError]` (zero is an error value, not
  a panic: unlike the `int` parity above, here the error arises from the
  data).
- **Observers:** `@num()`/`@den()`, `@sign()`, `@is_zero()`, `@is_int()`,
  `@compare`/`@equal`/`@hash`.
- **Conversions:** `T @to_bigrat()` (all `Ints` members + `i128` — exact),
  `str @to_bigrat()` (forms `"3/4"`, `"-12"`, decimal `"1.25"`) →
  `Result[BigRat, ParseNumberError]`, `@to_str()` (`"num/den"` or an
  integer), `@to_int() -> Option[int]`; bridges: `BigDecimal @to_bigrat()`
  (exact), `BigRat @to_bigdecimal(mc)` (rounded per `MathContext`),
  `BigFloat @to_bigrat()` (exact).

Conversion family rule: **exact directions need no context; lossy
directions require an explicit `MathContext`/`PrecisionContext`** (no
hidden rounding).

## Representation

```nova
type Sign enum Neg | Zero | Pos

type BigInt value {
    sign Sign
    limbs []u32
}
```

- Value-record on the stack: a copy is Sign-clr + slice-header (~24
  bytes).
- Limb data lives on the GC heap, shared across copies.
- Immutability guarantees no alias mutations.
- `==` is structural (value-record, D328).
- Normalization: no leading zero limbs; zero is canonically `Zero` + empty
  limbs.

## Multiplication

Karatsuba `O(n^log₂3)`, falling back to schoolbook `O(n²)` for n < 16
limbs.

## Division

Knuth's Algorithm D (divisor normalization, quotient-digit guess with
correction, remainder denormalization). Truncates toward zero (parity
with `int`/C99/Rust).

## Usage

```toml
[dependencies]
bignum = { git = "https://github.com/nv-lang/nova-bignum", version = "0.1" }
```

```nova
import bignum.{BigInt, Sign, ParseNumberError, DivError}

let a = 42.to_bigint()
let b = "12345678901234567890".to_bigint()!!
assert(a + b > a)
```

## Limitations

- All allocations happen on the GC heap; every immutable operation
  creates a new `BigInt`.
- Mutating `@add_assign`/`@mul_assign` are deliberately absent in V1: they
  would break the value-record immutability guarantee (a copy shares the
  limb buffer — mutating it would corrupt the "immutable" copy), and a
  wrapper over `@plus` buys no savings. Buffer reuse is a V2 concern
  (either inside the operations themselves, or as a separate mutable
  accumulator type).
- The `/` and `%` operators do NOT desugar (they would have to return a
  `Result`). Use the explicit `@div_rem`/`@div`/`@rem` instead.

## Testing

```sh
nova test src
```

## License

MIT OR Apache-2.0
