# nova-bignum — обзор

[English](overview.md) | **Русский**

`nova-bignum` — пакет чисел произвольной точности для [Nova](https://nv-lang.org):
`BigInt` (целые без ограничения размера), `BigRat` (точные рациональные
числа), `BigDecimal` (десятичное произвольной точности) и `BigFloat`
(двоичное с плавающей точкой произвольной точности) — аналог Python
`int`/`decimal`/`fractions`, Rust `num-bigint`/`num-rational`, Go `math/big`.

Все четыре типа — value-record: копия — небольшой заголовок на стеке
(знак плюс slice/указатель), данные лимбов/мантиссы живут на GC-куче и
разделяются при копировании, а иммутабельность означает, что ни одна
операция не мутирует значение, на которое ссылается другая переменная.
У `BigDecimal`, `BigFloat` и `BigRat` своей арифметики лимбов нет —
каждый из них построен поверх `BigInt` и делегирует ему всю арифметику.

## Семья одним взглядом

| Тип | Представление | Точность | Типичное применение |
|---|---|---|---|
| [`BigInt`](bigint.ru.md) | `sign Sign, limbs []u32` | Точная, без ограничения | Целочисленная математика за пределами `i64`/`u64`/`i128` — факториалы, крипто-смежная арифметика, большие счётчики |
| [`BigRat`](bigrat.ru.md) | `num BigInt, den BigInt` (несократимая) | Точная | Дроби, которые не должны накапливать ошибку округления — точные отношения, точная математика ставок |
| [`BigDecimal`](bigdecimal.ru.md) | `mant BigInt, scale int` (значение = `mant × 10^{-scale}`) | Явная, десятичная | Деньги и всё десятично-ориентированное, где двоичное округление `f64` (`0.1 + 0.2 != 0.3`) недопустимо |
| [`BigFloat`](bigfloat.ru.md) | `mant BigInt, exp int` (значение = `mant × 2^exp`) | Явная, двоичная | Математика в духе `f64` расширенной точности — значащих бит больше, чем 53 у `f64`, но всё ещё двоичная плавающая точка |

### Какой тип брать

- Нужно целое шире `i128`, без дробной части вообще? **`BigInt`.**
- Нужна дробь, которая остаётся точной вечно — без дрейфа после тысячи
  операций? **`BigRat`.**
- Нужна десятичная семантика (деньги, цены, всё, что человек читает в
  десятичной системе) с явной, управляемой политикой округления?
  **`BigDecimal`.**
- Нужна двоичная плавающая точка в духе `f64`, но точнее 53 значащих бит,
  и рассуждать в двоичной системе — нормально? **`BigFloat`.**

Конверсии между членами семьи подчиняются одному правилу везде:
**точные направления не требуют контекста; теряющие точность —
требуют явного `MathContext` (десятичный) или `PrecisionContext`
(двоичный)** — скрытых округлений в пакете нет нигде. `BigInt → BigRat/
BigDecimal` всегда точна; `BigDecimal → BigRat` и `BigFloat → BigRat`
точны (десятичная или двоичная дробь всегда представима точной
рациональной); `BigRat → BigDecimal`, `BigRat → BigFloat` и
`BigDecimal ↔ BigFloat` теряют точность и требуют контекста.

## Минимальный пример

```nova
import bignum.{Sign, ParseNumberError}
import bignum.bigint.{BigInt, DivError}

ro a = 42.to_bigint()
ro b = "12345678901234567890".to_bigint()!!
assert(a + b > a)
```

## Карта модулей

| Модуль | Что содержит |
|---|---|
| `bignum` (корень) | `Sign` (`Neg`/`Zero`/`Pos`), `ParseNumberError` — общий enum ошибок разбора, используемый каждой конверсией `str @to_*()` во всей семье |
| `bignum.bigint` | `BigInt`, `DivError` |
| `bignum.bigdecimal` | `BigDecimal`, `RoundingMode`, `MathContext` |
| `bignum.bigfloat` | `BigFloat`, `PrecisionContext` |
| `bignum.bigrat` | `BigRat` |

## Ошибки — значениями

Во всей семье повторяются две формы отказа:

- **Деление на ноль.** `BigInt`/`BigRat` возвращают его значением —
  `Result[_, DivError]` с единственным вариантом `DivisionByZero`.
  `BigDecimal`/`BigFloat` вместо этого паникуют (паритет с контрактом
  деления-на-ноль примитивных `int`/`f64`) — для этих двух типов это
  осознанное решение дизайна, а не недосмотр.
- **Разбор строк.** Каждая `str @to_bigint()`/`@to_bigdecimal()`/
  `@to_bigfloat()`/`@to_bigrat()` возвращает `Result[_, ParseNumberError]`,
  один общий enum на весь пакет (`Empty`, `OnlySign`, `InvalidCharacter`,
  `MultiplePoints`, `MultipleExponents`, `EmptyExponent`, `EmptyMantissa`,
  `ZeroDenominator`) — без per-type варианта или адаптера.

`panic` зарезервирован для внутренних инвариантов и для контрактов
деления-на-ноль/sqrt-от-отрицательного, упомянутых выше — никогда для
значения, переданного вызывающим через API, возвращающий `Result`.

## Содержание набора доков

- [bigint.ru.md](bigint.ru.md) — `BigInt`: представление, арифметика,
  умножение Карацубы, деление по алгоритму Кнута D, границы
- [bigdecimal.ru.md](bigdecimal.ru.md) — `BigDecimal`: scale,
  `RoundingMode`, `MathContext`
- [bigfloat.ru.md](bigfloat.ru.md) — `BigFloat`: мантисса/экспонента,
  точность, конверсии `f64`
- [bigrat.ru.md](bigrat.ru.md) — `BigRat`: точные дроби, нормальная
  форма, мосты к `BigDecimal`/`BigFloat`

## Связанные документы

- [`README.md`](../README.md) — питч пакета, установка, полный список
  возможностей/ограничений
- `docs/plans/235-bigint.md`, `236-bigdecimal.md`, `237-bigfloat.md`,
  `240-bigrat.md`, `243-*.md` в репе `nova` — планы дизайна, которые
  реализует этот пакет
- [`src/bigint/core_test.nv`](../src/bigint/core_test.nv),
  [`src/bigdecimal/core_test.nv`](../src/bigdecimal/core_test.nv),
  [`src/bigfloat/core_test.nv`](../src/bigfloat/core_test.nv),
  [`src/bigrat/core_test.nv`](../src/bigrat/core_test.nv) — полные
  наборы тестов; каждый пример этого набора доков адаптирован из одного
  из них
