# BigDecimal

**English** | [Русский](bigdecimal.ru.md)

`BigDecimal` is an arbitrary-precision decimal number — module
`bignum.bigdecimal`. It has no limb arithmetic of its own; every
operation delegates to [`BigInt`](bigint.md). See [overview.md](overview.md)
for how it relates to the rest of the `bignum` family.

## Representation

```nova
type BigDecimal value {
    mant BigInt
    scale int
}
```

Value = `mant × 10^{-scale}`. `scale` may be negative (an integer with
implicit trailing zeros). A copy is a `BigInt` heap pointer plus an `int`
— roughly 16 bytes on the stack; the mantissa data itself is shared like
any `BigInt`.

## Constructing

```nova
ro price = "19.99".to_bigdecimal()!!
ro zero = BigDecimal.zero()
ro direct = BigDecimal.new((1999).to_bigint(), 2)
```

- `str @to_bigdecimal() -> Result[BigDecimal, ParseNumberError]` — parses
  `[sign] (int-part ['.' [frac-part]] | '.' frac-part) [exp-part]`, e.g.
  `"19.99"`, `".5"`, `"5."`, `"1e-3"`, `"1.5E+2"`. `_` is accepted as a
  digit-group separator and stripped before the scale is computed (Rust/
  Python 3.6+ style). Only ASCII `0`-`9` are digits — anything else is
  `Err(InvalidCharacter)`.
- `T @to_bigdecimal()` for any `Ints` member plus `i128` — always
  `scale = 0`, infallible.
- `BigDecimal.new(mant, scale)` — direct construction, no normalization.
- `BigDecimal.zero()` — the canonical zero (`mant = 0, scale = 0`).

`@scale() -> int` reads the `scale` field (a property-method, not the
rounding operation — see `@rescale` below, which has a different name
specifically to avoid colliding with this reader).

## Arithmetic

```nova
ro a = "0.1".to_bigdecimal()!!
ro b = "0.2".to_bigdecimal()!!
ro c = a.plus(b)
assert(c.to_str(scale_pad: 0) == "0.3")
```

`@plus`/`@minus`/`@times` desugar to `+`/`-`/`*`; `@neg`/`@abs` are exact.
Addition and subtraction align scales (widen the smaller-scale operand by
the needed power of ten) before combining mantissas; multiplication just
adds the scales — none of the three ever rounds.

```nova
ro price = "19.99".to_bigdecimal()!!
ro tax = "0.01".to_bigdecimal()!!
assert(price.plus(tax).to_str() == "20.00")     // not 20.000000000000004, like f64
```

**The `/` operator does NOT desugar** — dividing two `BigDecimal`s is
ambiguous without an explicit precision, so division is only available as
`@div(other, MathContext)`:

```nova
ro third = (1).to_bigdecimal().div((3).to_bigdecimal(), MathContext.new(4, HalfEven))
assert(third.to_str() == "0.3333")
```

Division by zero panics (`requires !other.mant.is_zero()`) — parity with
the primitive `int`/`f64` division-by-zero contract (D423), not a
`Result`.

## Rounding: RoundingMode and MathContext

```nova
type RoundingMode enum HalfEven | HalfUp | HalfDown | Down | Up | Ceiling | Floor

type MathContext value {
    precision int
    rm RoundingMode
}
```

- `HalfEven` — banker's rounding, IEEE 754 default (ties go to the even
  digit); `HalfUp` — school rounding (0.5 → +1); `HalfDown` — 0.5 → 0;
  `Down` — truncate toward zero; `Up` — away from zero; `Ceiling` — toward
  `+∞`; `Floor` — toward `-∞`.
- `MathContext.new(precision, rm)` — `precision` counts **significant
  digits of the mantissa**, not decimal places; it panics
  (`requires precision >= 1`) below 1 — unbounded precision (`0`) is not
  supported in V1 because unbounded `1/3` never terminates.

```nova
ro a = (2).to_bigdecimal()
ro b = (3).to_bigdecimal()
ro mc = MathContext.new(5, HalfUp)
assert(a.div(b, mc).to_str(scale_pad: 0) == "0.66667")
```

`@round(ctx MathContext) -> BigDecimal` rounds to `ctx.precision`
significant digits. `@rescale(target int, rm RoundingMode) -> BigDecimal`
is the scale-based counterpart (analogous to Java's
`setScale(int, RoundingMode)`): `target > scale` pads with zeros without
rounding, `target < scale` rounds away decimal places, and
`target < 0` rounds to `10^|target|` (e.g. `target = -2` rounds to the
nearest hundred).

## Comparison and equality

```nova
ro x = BigDecimal.new((10).to_bigint(), 1)   // 1.0
ro y = BigDecimal.new((1).to_bigint(), 0)    // 1
assert(x.compare(y) == 0)
assert(x == y)
```

`@compare(other) -> int` does NOT normalize — it aligns scales (multiplies
the operand with the smaller scale by the matching power of ten) and
compares mantissas directly. `@equal(other)` (which backs `==`) instead
normalizes **both** operands first — parity with Rust's `bigdecimal`
crate, where `1.0 == 1`. This asymmetry is deliberate: `@compare` never
allocates a normalization pass on the hot path, `@equal` needs the
canonical form to be correct at all. `@hash()` is consistent with
`@equal` (it normalizes too).

## Normalization

`@normalize() -> BigDecimal` strips trailing decimal zeros from the
mantissa, lowering `scale` to match — it is **lazy**: no constructor or
arithmetic operation calls it implicitly, only `@equal`/`@hash`/an
explicit call does. This keeps arithmetic cheap (no O(n²) division-by-10
loop on every operation) at the cost of `BigDecimal` values not being in
one canonical form until you ask for it.

## String output

```nova
ro x = "12.5".to_bigdecimal()!!
assert(x.to_str() == "12.5")
assert(x.to_str(scale_pad: 4) == "12.5000")
```

`@to_str(scale_pad int = 0) -> str` — `scale_pad` is the minimum number
of digits after the decimal point (padded on the right with zeros; `0`
means no padding). The sign is printed first, ahead of any leading zero,
so `-0.5` round-trips correctly.

## Conversions to fixed-width types

`@to_int() -> Option[int]` and `@to_i128() -> Option[i128]` both
normalize first: a positive residual `scale` after normalization (a real
fractional part) yields `None` rather than silently truncating; a
negative `scale` (a whole number with implicit trailing zeros) is
materialized before the range check.

## Not in BigDecimal V1

`pow`/`sqrt`, the `/` operator itself, mutable `@*_assign` methods,
chained operations with auto-carried precision, integration with generic
numeric type-sets, implicit coercions/literals, a small-powers-of-10
cache, Toom-Cook multiplication (inherited from the underlying `BigInt`).

## Related documents

- [overview.md](overview.md) — the `bignum` family map, the shared
  exact-vs-lossy conversion rule
- [bigint.md](bigint.md) — the type every `BigDecimal` operation
  ultimately delegates to
- [bigfloat.md](bigfloat.md) — the `BigDecimal ↔ BigFloat` bridge (via a
  `PrecisionContext`)
- [bigrat.md](bigrat.md) — the exact `BigDecimal ↔ BigRat` bridge
- [`src/bigdecimal/core.nv`](../src/bigdecimal/core.nv) — full source
- [`src/bigdecimal/core_test.nv`](../src/bigdecimal/core_test.nv) — full
  test suite
