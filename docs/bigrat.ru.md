# BigRat

[English](bigrat.md) | **Русский**

`BigRat` — точное рациональное число, модуль `bignum.bigrat`, аналог
`big.Rat` из Go / `num-rational` из Rust. Своей арифметики лимбов у него
нет — каждая операция делегируется в [`BigInt`](bigint.ru.md). Как он
соотносится с остальной семьёй `bignum` — см. [overview.ru.md](overview.ru.md).

## Представление и нормальная форма

```nova
type BigRat value {
    num BigInt
    den BigInt
}
```

Значение = `num / den`. Каждый существующий `BigRat` — в **нормальной
форме**, и каждая операция её поддерживает:

- `den > 0` — знак живёт целиком в `num`.
- `gcd(|num|, den) == 1` — несократимо.
- Ноль каноничен: `0/1`.

В отличие от *lazy*-нормализации `BigDecimal`/`BigFloat`, у `BigRat` она
**жадная** — каждый конструктор и каждая арифметическая операция сразу
сокращают результат. Для рациональных это не косметика: без этого
`num`/`den` росли бы без ограничения вдоль цепочки операций (сложение
`n` дробей может взорвать знаменатели комбинаторно); сокращение после
каждого шага держит их в границах реального информационного содержания
результата.

## Построение

```nova
ro r = BigRat.new((1).to_bigint(), (2).to_bigint())!!
assert(r.num() == (1).to_bigint())
assert(r.den() == (2).to_bigint())

ro a = BigRat.new((-1).to_bigint(), (-2).to_bigint())!!
assert(a.num() == (1).to_bigint())
assert(a.den() == (2).to_bigint())
```

`BigRat.new(num, den) -> Result[BigRat, DivError]` строит `BigRat` из
числителя и знаменателя, сокращая к нормальной форме; `den == 0` даёт
`Err(DivisionByZero)` (паритет с `BigInt @div_rem`), а не панику.
`BigRat.zero()`/`BigRat.one()` — две константы.

```nova
ro r = (42).to_bigrat()
ro neg = (-7).to_bigrat()
```

`T @to_bigrat()` для любого члена `Ints` плюс `i128` строит `n/1` —
бесотказная, всегда точная.

## Предикаты и наблюдатели

`@is_zero()`, `@is_int()` (`den == 1` в нормальной форме),
`@sign() -> Sign` (знаменатель всегда положителен, поэтому знак живёт
целиком в числителе). `@num()`/`@den()` — свойства-читатели двух полей.

## Сравнение и равенство

```nova
ro a = BigRat.new((2).to_bigint(), (4).to_bigint())!!
ro b = BigRat.new((-6).to_bigint(), (8).to_bigint())!!
assert(a.compare(b) > 0)
```

`@equal(other)` (за которым стоит `==`) — **структурное**, покомпонентное
сравнение `num`/`den` — корректно именно потому, что нормальная форма
делает `(num, den)` уникальной для значения, без шага выравнивания
(в отличие от `BigDecimal`, которому приходится нормализовать перед
сравнением). `@compare(other) -> int` кросс-умножает: `num_a·den_b <=>
num_b·den_a` (корректно, потому что `den > 0` с обеих сторон — знаки
отслеживать не нужно, и никакой плавающей точки нигде не участвует).
`@hash()` согласован с `@equal`.

## Арифметика

```nova
ro a = BigRat.new((1).to_bigint(), (2).to_bigint())!!
ro b = BigRat.new((1).to_bigint(), (3).to_bigint())!!
ro s = a.plus(b)
assert(s.num() == (5).to_bigint())
assert(s.den() == (6).to_bigint())
```

`@plus`/`@minus`/`@times` точные — округление для точных рациональных
невозможно в принципе, поэтому, в отличие от `BigDecimal`/`BigFloat`, им
вообще не нужен аргумент-контекст. `@neg`/`@abs` тоже точные.

```nova
ro third = BigRat.new((1).to_bigint(), (3).to_bigint())!!
ro sixth = BigRat.new((1).to_bigint(), (6).to_bigint())!!
assert(third.plus(sixth) == BigRat.new((1).to_bigint(), (2).to_bigint())!!)
```

`@div(other)` и `@recip()` оба возвращают `Result[BigRat, DivError]` —
**оператор `/` НЕ десугарится** (симметрия с правилом «`/` не оператор» у
`BigDecimal`, D9). В отличие от `BigDecimal`/`BigFloat`, где деление на
ноль *паникует*, у `BigRat` деление на ноль — **значение-ошибка**:
нулевой знаменатель `BigRat` может возникнуть только из данных,
переданных вызывающим (числитель другого `BigRat`, или разобранная
строка), а не из нарушения внутреннего инварианта, поэтому это остаётся
`Result` от начала до конца.

## Строковые конверсии

```nova
ro a = "5".to_bigrat()!!
ro c = "1/2".to_bigrat()!!
assert(a.to_str() == "5")
assert(c.to_str() == "1/2")
```

`str @to_bigrat() -> Result[BigRat, ParseNumberError]` принимает простое
целое (`"5"`, `"-5"`) или дробь `"p/q"` (`"1/2"`, `"-1/2"`, `"1/-2"`);
обе части делегируются в `str @to_bigint()`, поэтому используют его же
`ParseNumberError` — адаптер не нужен. Нулевой знаменатель в форме `p/q`
— `Err(ZeroDenominator)`, отдельный от `DivisionByZero` у `BigInt`
вариант (это возникает при разборе непроверенного текста, а не при
вызове деления). `@to_str() -> str` печатает `"num/den"`, либо просто
целое, если `den == 1`. Прямой десятичной строковой формы здесь нет
(`"1.25"`) — десятичный текст сначала проводится через `BigDecimal`
(см. ниже).

`@to_int()`/`@to_i128() -> Option[...]` усекают к нулю (паритет с
делением `int`), `None` при переполнении целевого типа.

## Мосты к BigDecimal и BigFloat

```nova
ro from_dec = "1.25".to_bigdecimal()!!.to_bigrat()
assert(from_dec == BigRat.new((5).to_bigint(), (4).to_bigint())!!)

ro back = BigRat.new((5).to_bigint(), (4).to_bigint())!!.to_bigdecimal(MathContext.new(10, HalfEven))
assert(back == "1.25".to_bigdecimal()!!)
```

`BigDecimal @to_bigrat() -> BigRat` **точная**: `mant / 10^scale`,
сокращённая к нормальной форме (отрицательный `scale` — целое с
неявными конечными нулями — вместо этого домножает на `10^|scale|`).
Это же и способ распарсить десятичную строку в `BigRat`, поскольку сам
`str @to_bigrat()` понимает только целую и `p/q`-формы: сначала пройдите
через `str @to_bigdecimal()`. `BigRat @to_bigdecimal(mc MathContext) ->
BigDecimal` — обратное направление, и она **округляет** — реализована
как `BigDecimal(num, 0).div(BigDecimal(den, 0), mc)`, поэтому ей, как и
любой другой теряющей точность конверсии, нужен контекст.

`BigFloat @to_bigrat() -> BigRat` тоже **точная**: `exp >= 0` даёт
`num = mant << exp, den = 1`; `exp < 0` даёт `den = 1 << |exp|` (мантисса
`BigFloat` после нормализации всегда нечётна, поэтому её НОД со степенью
двойки равен 1 — шаг сокращения даже не нужен).

Во всей семье действует одно и то же правило из
[overview.ru.md](overview.ru.md): **точные направления не требуют
контекста, теряющие точность — требуют** — `BigInt`/`BigDecimal`/
`BigFloat` → `BigRat` все точны; обратное, `BigRat` → `BigDecimal`,
требует `MathContext`, потому что десятичное разложение рационального
может быть бесконечным (`1/3`).

## Связанные документы

- [overview.ru.md](overview.ru.md) — карта семьи `bignum`, общее правило
  точных/теряющих точность конверсий
- [bigint.ru.md](bigint.ru.md) — тип, которому в итоге делегирует каждая
  операция `BigRat`
- [bigdecimal.ru.md](bigdecimal.ru.md) — мост `BigDecimal ↔ BigRat` выше
- [bigfloat.ru.md](bigfloat.ru.md) — мост `BigFloat → BigRat` выше
- [`src/bigrat/core.nv`](../src/bigrat/core.nv) — полный исходник
- [`src/bigrat/core_test.nv`](../src/bigrat/core_test.nv) — полный набор
  тестов; [`core_slow.nv`](../src/bigrat/core_slow.nv) — медленная
  дорожка property-тестов (`nova test --include-slow`)
