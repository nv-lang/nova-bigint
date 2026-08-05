# BigFloat

**English** | [Русский](bigfloat.ru.md)

`BigFloat` is an arbitrary-precision binary floating-point number —
module `bignum.bigfloat`, analogous to MPFR or Go's `big.Float`. All
arithmetic is `BigInt` convolutions with a rounding pass at the end. See
[overview.md](overview.md) for how it relates to the rest of the `bignum`
family.

## Representation

```nova
type BigFloat value {
    mant BigInt
    exp int
}
```

Value = `mant × 2^exp`. The mantissa is an integer (fixed-point, radix
point conceptually at the right); `exp` may be negative. A copy is a
`BigInt` heap pointer plus an `int`.

## PrecisionContext

```nova
type PrecisionContext value {
    prec int
    rm RoundingMode
}
```

Every rounding-sensitive operation (`@plus`, `@minus`, `@times`, `@div`,
`@sqrt`) takes an explicit `PrecisionContext` — `prec` is **significant
bits** of the mantissa (not decimal digits, unlike `BigDecimal`'s
`MathContext`), `rm` reuses `BigDecimal`'s `RoundingMode` enum.
`PrecisionContext.new(prec, rm)` panics (`requires prec >= 2`) below 2
bits — at `prec < 2` any rounding collapses to `0` or `±∞` by sign.

```nova
ro ctx = PrecisionContext.new(53, HalfEven)
```

## Constructing

```nova
ro x = 42.to_bigfloat(ctx)
ro y = (1.0).to_bigfloat()
ro z = "0b101.01p0".to_bigfloat(ctx)!!
assert(z.to_f64() == Some(5.25))
```

- `T @to_bigfloat(ctx)` for any `Ints` member plus `i128` — exact
  (integers always fit exactly as `mant × 2^0`), the context is taken for
  API symmetry but never actually rounds.
- `f64 @to_bigfloat() -> BigFloat` — **exact**, no context needed: it
  unpacks IEEE 754 binary64 bit-for-bit (normalized, subnormal, and ±0
  are all handled losslessly). `±Inf`/`NaN` panic — V1 has no
  infinity/NaN representation.
- `str @to_bigfloat(ctx) -> Result[BigFloat, ParseNumberError]` accepts
  two formats: decimal (parsed the same way as `BigDecimal`, then
  converted) and binary — `[sign] '0b' bin-digits ['.' bin-frac] 'p'
  [sign] digits`, e.g. `"0b101.01p0"` = 5.25, `"-0b1.1p-1"` = -0.75.
  The binary form is detected by the `0b` prefix; there is no `bf`
  literal suffix.
- `BigFloat.zero()` / `BigFloat.one()` — the two constants.
- `BigFloat.new(mant, exp)` — direct construction, no normalization.

## Predicates

`@sign() -> Sign`, `@is_zero()`, `@is_pos()`, `@is_neg()`,
`@is_integer()` (normalizes, then checks `exp >= 0`).

## Comparison and equality

`@compare(other) -> int` takes an msb (most-significant-bit-position)
fast path: it compares `exp + bit_length(mant)` first, and only falls
back to an alignment shift (bounded by the mantissa bit-length
difference) when the msb positions tie — no rounding involved, ever.
`@equal(other)` (behind `==`) normalizes **both** operands first (lazy
normalization means post-arithmetic mantissas can still be even).
`@hash()` is consistent with `@equal`.

## Arithmetic

```nova
ro a = BigFloat.new(4.to_bigint(), 0)
ro b = BigFloat.new(2.to_bigint(), 0)
ro sum = a.plus(b, ctx)
assert(sum.equal(BigFloat.new(6.to_bigint(), 0)))
```

`@plus`/`@minus`/`@times`/`@div` all take an explicit `PrecisionContext`
— there is no operator desugaring for any of them (parity with
`BigDecimal`'s `@div`: rounding always needs an explicit precision).
`@neg`/`@abs` need no context — sign flips are exact. `@div` panics on a
zero divisor (`requires !other.mant.is_zero()`), same D423 parity as
`BigDecimal`.

```nova
ro four = BigFloat.new(4.to_bigint(), 0)
ro root = four.sqrt(ctx)
assert(root.equal(BigFloat.new(2.to_bigint(), 0)))
```

`@sqrt(ctx)` uses Newton's method — **V1: within ±1 ulp, not
correctly-rounded** — and panics on a negative argument
(`requires !@mant.is_neg()`).

## Rounding

`@round(prec int, rm RoundingMode) -> BigFloat` rounds to `prec`
significant **bits** (do not confuse with `to_str`'s `frac_digits`,
which counts decimal digits after the point). With `prec >=` the
current bit length it is a no-op.

## Normalization

`@normalize() -> BigFloat` removes trailing zero bits from the mantissa
(increasing `exp` to compensate), leaving the mantissa odd or zero —
**lazy**: not called by any constructor or arithmetic operation, only by
`@equal`/`@hash`/an explicit call, same discipline as `BigDecimal`.

## Conversions out

```nova
ro back = z.to_f64()
assert(back == Some(5.25))
```

- `@to_f64() -> Option[f64]` — checked, single correctly-rounded
  conversion (round-to-nearest-ties-to-even), `None` on overflow/
  underflow outside the `f64` range, `Some(0.0)` for zero; subnormals
  are supported.
- `@to_int()`/`@to_i64()`/`@to_u64() -> Option[...]` — checked, via
  `@to_bigint()` then a range check.
- `@to_bigint() -> BigInt` — truncation toward zero (drops the
  fractional part): `exp >= 0` shifts left, `exp < 0` shifts the
  magnitude right.
- `@to_str(frac_digits int = 6) -> str` — via `BigDecimal`:
  `@to_bigdecimal()` → `@rescale(frac_digits, HalfEven)` → `@to_str()`.

## The BigDecimal bridge

```nova
ro bf = BigFloat.new(5.to_bigint(), -1)   // 5 * 2^-1 = 2.5
ro bd = bf.to_bigdecimal()
assert(bd.to_str() == "2.5")
```

`@to_bigdecimal() -> BigDecimal` is **exact**: for `exp >= 0` the value
is already an integer (`mant × 2^exp`); for `exp < 0` it multiplies by
`5^|exp|` to get `mant × 5^|exp| / 10^|exp|`. `BigDecimal @to_bigfloat(ctx)
-> BigFloat` is the reverse direction and **rounds** to `ctx.prec` — it
divides by `5^scale` instead, so it needs the context. Large `|exp|`/
`scale` (beyond ~10⁵) make an intermediate `5^k` that is itself an
`O(n²)`-sized `BigInt` — a documented cost, not a bug.

## Related documents

- [overview.md](overview.md) — the `bignum` family map, the shared
  exact-vs-lossy conversion rule
- [bigint.md](bigint.md) — the type every `BigFloat` operation ultimately
  delegates to
- [bigdecimal.md](bigdecimal.md) — the decimal counterpart and the bridge
  above
- [bigrat.md](bigrat.md) — the exact `BigFloat ↔ BigRat` bridge
- [`src/bigfloat/core.nv`](../src/bigfloat/core.nv) — full source
- [`src/bigfloat/core_test.nv`](../src/bigfloat/core_test.nv) — full test
  suite
