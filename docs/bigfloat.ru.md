# BigFloat

[English](bigfloat.md) | **Русский**

`BigFloat` — двоичное число с плавающей точкой произвольной точности,
модуль `bignum.bigfloat`, аналог MPFR или `big.Float` из Go.
`@plus`/`@minus`/`@times`/`@div` принимают явный `PrecisionContext`;
`@neg`/`@abs` его не требуют. Под капотом каждая операция — свёртка
`BigInt` с раундом в конце. Как он соотносится с остальной семьёй
`bignum` — см. [overview.ru.md](overview.ru.md).

## Представление

```nova
type BigFloat value {
    mant BigInt
    exp int
}
```

Значение = `mant × 2^exp`. Мантисса — целое (fixed-point, точка
концептуально справа); `exp` может быть отрицательным. Копия — указатель
на `BigInt` плюс `int`.

## PrecisionContext

```nova
type PrecisionContext value {
    prec int
    rm RoundingMode
}
```

Каждая чувствительная к округлению операция (`@plus`, `@minus`, `@times`,
`@div`, `@sqrt`) принимает явный `PrecisionContext` — `prec` это
**значащие биты** мантиссы (не десятичные цифры, в отличие от
`MathContext` у `BigDecimal`), `rm` переиспользует enum `RoundingMode`
от `BigDecimal`. `PrecisionContext.new(prec, rm)` паникует
(`requires prec >= 2`) ниже 2 бит — при `prec < 2` любое округление
схлопывается в `0` или `±∞` по знаку.

```nova
ro ctx = PrecisionContext.new(53, HalfEven)
```

## Построение

```nova
ro x = 42.to_bigfloat(ctx)
ro y = (1.0).to_bigfloat()
ro z = "0b101.01p0".to_bigfloat(ctx)!!
assert(z.to_f64() == Some(5.25))
```

- `T @to_bigfloat(ctx)` для любого члена `Ints` плюс `i128` — точная
  (целые всегда точно помещаются как `mant × 2^0`), контекст берётся
  ради симметрии API, но никогда не округляет фактически.
- `f64 @to_bigfloat() -> BigFloat` — **точная**, контекст не нужен: она
  распаковывает IEEE 754 binary64 побитово (нормализованные,
  субнормальные и ±0 — все обрабатываются без потерь). `±Inf`/`NaN`
  паникуют — представления бесконечности/NaN в V1 нет.
- `str @to_bigfloat(ctx) -> Result[BigFloat, ParseNumberError]`
  принимает два формата: десятичный (парсится так же, как у
  `BigDecimal`, затем конвертируется) и двоичный — `[sign] '0b'
  bin-digits ['.' bin-frac] 'p' [sign] digits`, например `"0b101.01p0"`
  = 5.25, `"-0b1.1p-1"` = -0.75. Двоичная форма определяется по
  префиксу `0b`; суффикса-литерала `bf` нет.
- `BigFloat.zero()` / `BigFloat.one()` — две константы.
- `BigFloat.new(mant, exp)` — прямое построение, без нормализации.

## Предикаты

`@sign() -> Sign`, `@is_zero()`, `@is_pos()`, `@is_neg()`,
`@is_integer()` (нормализует, затем проверяет `exp >= 0`).

## Сравнение и равенство

`@compare(other) -> int` идёт по быстрому пути через msb (позицию
старшего значащего бита): сначала сравнивает `exp + bit_length(mant)`, и
только при равенстве позиций msb переходит к сдвигу выравнивания
(ограниченному разницей битовой длины мантисс) — округления нет никогда.
`@equal(other)` (за которым стоит `==`) сначала нормализует **оба**
операнда (lazy-нормализация означает, что мантиссы после арифметики
могут оставаться чётными). `@hash()` согласован с `@equal`.

## Арифметика

```nova
ro a = BigFloat.new(4.to_bigint(), 0)
ro b = BigFloat.new(2.to_bigint(), 0)
ro sum = a.plus(b, ctx)
assert(sum.equal(BigFloat.new(6.to_bigint(), 0)))
```

`@plus`/`@minus`/`@times`/`@div` — все принимают явный
`PrecisionContext`, десугара оператором ни у одной из них нет (паритет
с `@div` у `BigDecimal`: округление всегда требует явной точности).
`@neg`/`@abs` контекста не требуют — смена знака точна. `@div` паникует
на нулевом делителе (`requires !other.mant.is_zero()`) — тот же паритет
D423, что и у `BigDecimal`.

```nova
ro four = BigFloat.new(4.to_bigint(), 0)
ro root = four.sqrt(ctx)
assert(root.equal(BigFloat.new(2.to_bigint(), 0)))
```

`@sqrt(ctx)` использует метод Ньютона — **V1: в пределах ±1 ulp, не
correctly-rounded** — и паникует на отрицательном аргументе
(`requires !@mant.is_neg()`).

## Округление

`@round(prec int, rm RoundingMode) -> BigFloat` округляет до `prec`
значащих **бит** (не путать с `frac_digits` у `to_str`, который считает
десятичные цифры после запятой). При `prec >=` текущей битовой длины —
no-op.

## Нормализация

`@normalize() -> BigFloat` убирает конечные нулевые биты мантиссы
(увеличивая `exp` соответственно), оставляя мантиссу нечётной или нулём
— **lazy**: не вызывается никаким конструктором или арифметической
операцией, только `@equal`/`@hash`/явным вызовом, та же дисциплина, что
и у `BigDecimal`.

## Конверсии наружу

```nova
ro back = z.to_f64()
assert(back == Some(5.25))
```

- `@to_f64() -> Option[f64]` — проверяемая, единственный
  correctly-rounded раунд (round-to-nearest-ties-to-even), `None` при
  переполнении/антипереполнении за пределами диапазона `f64`,
  `Some(0.0)` для нуля; субнормали поддержаны.
- `@to_int()`/`@to_i64()`/`@to_u64() -> Option[...]` — проверяемые, через
  `@to_bigint()` и проверку диапазона.
- `@to_bigint() -> BigInt` — усечение к нулю (дробная часть отбрасывается):
  `exp >= 0` — сдвиг влево, `exp < 0` — сдвиг модуля вправо.
- `@to_str(frac_digits int = 6) -> str` — через `BigDecimal`:
  `@to_bigdecimal()` → `@rescale(frac_digits, HalfEven)` → `@to_str()`.

## Мост к BigDecimal

```nova
ro bf = BigFloat.new(5.to_bigint(), -1)   // 5 * 2^-1 = 2.5
ro bd = bf.to_bigdecimal()
assert(bd.to_str() == "2.5")
```

`@to_bigdecimal() -> BigDecimal` **точная**: при `exp >= 0` значение уже
целое (`mant × 2^exp`); при `exp < 0` она домножает на `5^|exp|`, получая
`mant × 5^|exp| / 10^|exp|`. `BigDecimal @to_bigfloat(ctx) -> BigFloat` —
обратное направление, и она **округляет** до `ctx.prec` — вместо этого
она делит на `5^scale`, поэтому ей нужен контекст. Большой `|exp|`/
`scale` (за пределами ~10⁵) даёт промежуточную `5^k`, которая сама по
себе `O(n²)`-размерный `BigInt` — задокументированная цена, не баг.

## Связанные документы

- [overview.ru.md](overview.ru.md) — карта семьи `bignum`, общее правило
  точных/теряющих точность конверсий
- [bigint.ru.md](bigint.ru.md) — тип, которому в итоге делегирует каждая
  операция `BigFloat`
- [bigdecimal.ru.md](bigdecimal.ru.md) — десятичный аналог и мост выше
- [bigrat.ru.md](bigrat.ru.md) — точный мост `BigFloat ↔ BigRat`
- [`src/bigfloat/core.nv`](../src/bigfloat/core.nv) — полный исходник
- [`src/bigfloat/core_test.nv`](../src/bigfloat/core_test.nv) — полный
  набор тестов
