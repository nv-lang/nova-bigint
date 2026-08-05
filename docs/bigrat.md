# BigRat

**English** | [Русский](bigrat.ru.md)

`BigRat` is an exact rational number — module `bignum.bigrat`, analogous
to Go's `big.Rat`/Rust's `num-rational`. Addition, subtraction, and
multiplication use the ordinary `+`/`-`/`*` operators, with no rounding
ever possible; division returns a `Result` (see below). Under the hood
every operation delegates to [`BigInt`](bigint.md). See
[overview.md](overview.md) for how it relates to the rest of the `bignum`
family.

## Representation and normal form

```nova
type BigRat value {
    num BigInt
    den BigInt
}
```

Value = `num / den`. Every `BigRat` in existence is in **normal form**,
and every operation maintains it:

- `den > 0` — the sign lives entirely in `num`.
- `gcd(|num|, den) == 1` — irreducible.
- Zero is canonical: `0/1`.

Unlike `BigDecimal`/`BigFloat`'s *lazy* normalization, `BigRat`'s is
**greedy** — every constructor and every arithmetic operation reduces the
result immediately. For rationals this isn't cosmetic: without it,
`num`/`den` would grow without bound across a chain of operations (adding
`n` fractions can blow denominators up combinatorially); reducing after
every step keeps them bounded by the actual information content of the
result.

## Constructing

```nova
ro r = BigRat.new(1.to_bigint(), 2.to_bigint())!!
assert(r.num() == 1.to_bigint())
assert(r.den() == 2.to_bigint())

ro a = BigRat.new((-1).to_bigint(), (-2).to_bigint())!!
assert(a.num() == 1.to_bigint())
assert(a.den() == 2.to_bigint())
```

`BigRat.new(num, den) -> Result[BigRat, DivError]` builds a `BigRat` from
a numerator and denominator, reducing to normal form; `den == 0` yields
`Err(DivisionByZero)` (parity with `BigInt @div_rem`) rather than a
panic. `BigRat.zero()`/`BigRat.one()` are the two constants.

```nova
ro r = 42.to_bigrat()
ro neg = (-7).to_bigrat()
```

`T @to_bigrat()` for any `Ints` member plus `i128` builds `n/1` —
infallible, always exact.

## Predicates and observers

`@is_zero()`, `@is_int()` (`den == 1` in normal form), `@sign() -> Sign`
(denominator is always positive, so the sign lives entirely in the
numerator). `@num()`/`@den()` are property-methods reading the two
fields.

## Comparison and equality

```nova
ro a = BigRat.new(2.to_bigint(), 4.to_bigint())!!
ro b = BigRat.new((-6).to_bigint(), 8.to_bigint())!!
assert(a.compare(b) > 0)
```

`@equal(other)` (behind `==`) is a **structural**, component-wise
comparison of `num`/`den` — correct precisely because normal form makes
`(num, den)` unique per value, with no alignment step needed (unlike
`BigDecimal`, which has to normalize before comparing). `@compare(other)
-> int` cross-multiplies: `num_a·den_b <=> num_b·den_a` (valid because
`den > 0` on both sides, so no sign flips to track — and no floating
point involved anywhere). `@hash()` is consistent with `@equal`.

## Arithmetic

```nova
ro a = BigRat.new(1.to_bigint(), 2.to_bigint())!!
ro b = BigRat.new(1.to_bigint(), 3.to_bigint())!!
ro s = a + b
assert(s.num() == 5.to_bigint())
assert(s.den() == 6.to_bigint())
```

`+`/`-`/`*` are exact — no rounding is possible for exact rationals, so
unlike `BigDecimal`/`BigFloat` they need no context argument at all;
under the hood they desugar to `@plus`/`@minus`/`@times`. `@neg`/`@abs`
are exact too.

```nova
ro third = BigRat.new(1.to_bigint(), 3.to_bigint())!!
ro sixth = BigRat.new(1.to_bigint(), 6.to_bigint())!!
assert(third + sixth == BigRat.new(1.to_bigint(), 2.to_bigint())!!)
```

`@div(other)` and `@recip()` both return `Result[BigRat, DivError]` — the
**`/` operator does NOT desugar** (symmetry with `BigDecimal`'s "`/` is
not an operator" rule, D9). Unlike `BigDecimal`/`BigFloat`, where
division by zero *panics*, `BigRat` division by zero is an **error
value**: a `BigRat` denominator of zero can only arise from data the
caller handed in (another `BigRat`'s `num`, or a parsed string), not from
an internal invariant violation, so it stays a `Result` all the way
through.

## String conversions

```nova
ro a = "5".to_bigrat()!!
ro c = "1/2".to_bigrat()!!
assert(a.to_str() == "5")
assert(c.to_str() == "1/2")
```

`str @to_bigrat() -> Result[BigRat, ParseNumberError]` accepts a plain
integer (`"5"`, `"-5"`) or a `"p/q"` fraction (`"1/2"`, `"-1/2"`,
`"1/-2"`); both parts delegate to `str @to_bigint()`, so they share its
`ParseNumberError` — no adapter needed. A zero denominator in the `p/q`
form is `Err(ZeroDenominator)`, a distinct variant from `BigInt`'s
`DivisionByZero` (this comes from parsing untrusted text, not a division
call). `@to_str() -> str` prints `"num/den"`, or just the integer when
`den == 1`. There is no direct decimal string form here (`"1.25"`) —
route decimal text through `BigDecimal` first (see below).

`@to_int()`/`@to_i128() -> Option[...]` truncate toward zero (parity with
`int` division), `None` on overflow of the target type.

## Bridges to BigDecimal and BigFloat

```nova
ro from_dec = "1.25".to_bigdecimal()!!.to_bigrat()
assert(from_dec == BigRat.new(5.to_bigint(), 4.to_bigint())!!)

ro back = BigRat.new(5.to_bigint(), 4.to_bigint())!!.to_bigdecimal(MathContext.new(10, HalfEven))
assert(back == "1.25".to_bigdecimal()!!)
```

`BigDecimal @to_bigrat() -> BigRat` is **exact**: `mant / 10^scale`,
reduced to normal form (a negative `scale` — an integer with implicit
trailing zeros — multiplies by `10^|scale|` instead). This is also how
you parse a decimal string into a `BigRat`, since `str @to_bigrat()`
itself only understands integer and `p/q` forms: go through
`str @to_bigdecimal()` first. `BigRat @to_bigdecimal(mc MathContext) ->
BigDecimal` is the reverse direction and **rounds** — it's implemented as
`BigDecimal(num, 0).div(BigDecimal(den, 0), mc)`, so it needs a context
like any other lossy conversion.

`BigFloat @to_bigrat() -> BigRat` is likewise **exact**: `exp >= 0` gives
`num = mant << exp, den = 1`; `exp < 0` gives `den = 1 << |exp|` (a
`BigFloat` mantissa is always odd after normalization, so its gcd with a
power of two is 1 — no reduction step is even needed).

Across the whole family, the rule is the same one from
[overview.md](overview.md): **exact directions need no context, lossy
directions require one** — `BigInt`/`BigDecimal`/`BigFloat` → `BigRat`
are all exact; the reverse, `BigRat` → `BigDecimal`, needs a
`MathContext` because a rational's decimal expansion can be infinite
(`1/3`).

## Related documents

- [overview.md](overview.md) — the `bignum` family map, the shared
  exact-vs-lossy conversion rule
- [bigint.md](bigint.md) — the type every `BigRat` operation ultimately
  delegates to
- [bigdecimal.md](bigdecimal.md) — the `BigDecimal ↔ BigRat` bridge above
- [bigfloat.md](bigfloat.md) — the `BigFloat → BigRat` bridge above
- [`src/bigrat/core.nv`](../src/bigrat/core.nv) — full source
- [`src/bigrat/core_test.nv`](../src/bigrat/core_test.nv) — full test
  suite; [`core_slow.nv`](../src/bigrat/core_slow.nv) — slow-lane
  property tests (`nova test --include-slow`)
