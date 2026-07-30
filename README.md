# nova-bigint

Целые произвольной точности (BigInt) на чистом Nova — аналог Python `int` / Rust `num-bigint` / Go `math/big`.

## Возможности (V1)

- **Арифметика:** `+`, `-`, `*`, `/`, `%`, `div_rem`, `pow`
- **Сравнение:** `==`, `!=`, `<`, `>`, `compare`
- **Знак/модуль:** `sign`, `is_zero`, `is_neg`, `is_one`, `neg`, `abs`
- **Конверсии:**
  - `T @to_bigint()` — любой `Ints` (`i8`..`i64`, `u8`..`u64`)
  - `i128 @to_bigint()` — библиотечный 128-битный тип
  - `str @to_bigint()` — десятичная строка → `Result[BigInt, ParseBigIntError]`
  - `BigInt @to_int()` → `Option[int]`
  - `BigInt @to_i128()` → `Option[i128]`
  - `BigInt @to_str()` → десятичная строка
- **Ошибки — значениями:** деление на ноль → `Err(DivisionByZero)`, разбор строки → `Err(ParseBigIntError)` с категорией (`Empty`, `InvalidCharacter`, `OnlySign`). `panic` только на внутренние инварианты.

## Чего НЕТ в V1

- `gcd`, `modpow`, битовые операции (`&`, `|`, `<<`, `>>`)
- `BigRat`, `BigFloat`
- Неявные коэрсии `int → BigInt`
- Литеральная форма `123456789012345678901n`
- Toom-3 умножение

Всё перечисленное — кандидаты в V2 по отдельным решениям.

## BigDecimal (V1) — десятичное произвольной точности

`type BigDecimal value { mant BigInt, scale int }` — значение = `mant × 10^{-scale}`.
Поверх BigInt (план 236, `docs/plans/236-bigdecimal.md` в репе `nova`); своей
арифметики лимбов нет — вся арифметика через `BigInt`.

- **Арифметика:** `+`/`-`/`*` (desugar `@plus`/`@minus`/`@times`), `@neg`, `@abs`.
  Оператор `/` НЕ десугарится — `@div(other, MathContext)` требует явной точности.
- **Сравнение/равенство:** `@compare` (выравнивание scale, без нормализации),
  `@equal`/`@hash` (нормализуют оба операнда — паритет Rust `bigdecimal`: `1.0 == 1`).
- **Округление:** `RoundingMode` (`HalfEven|HalfUp|HalfDown|Down|Up|Ceiling|Floor`),
  `MathContext { precision int, rm RoundingMode }`, `@round(ctx)` (значащие цифры),
  `@rescale(target, rm)` (десятичные знаки; `target < 0` — округление до `10^|target|`).
- **Конверсии:** `T @to_bigdecimal()` (все члены `Ints` + `i128`), `str @to_bigdecimal()
  -> Result[BigDecimal, ParseBigDecimalError]`, `@to_str(scale_pad = 0)`, `@to_int()`/
  `@to_i128() -> Option[...]`.
- **Нормализация:** `@normalize()` — убирает конечные нули мантиссы; **lazy** — не
  вызывается в конструкторах/операциях, только в `@equal`/`@hash`/явно.
- Деление на ноль — panic (паритет `int`, D423), не `Result`.

### Чего нет в BigDecimal V1

`pow`/`sqrt`, оператор `/`, мутабельные `@*_assign`, цепные операции с
авто-переносом точности, интеграция с generic-числовыми type-sets (D310),
неявные коэрсии/литералы, кэш малых степеней 10, Toom-Cook.

```nova
import bigint.bigdecimal.{BigDecimal, MathContext, HalfEven}

ro price = "19.99".to_bigdecimal()!!
ro tax = "0.01".to_bigdecimal()!!
assert(price.plus(tax).to_str() == "20.00")     // не 20.000000000000004, как в f64

ro third = (1).to_bigdecimal().div((3).to_bigdecimal(), MathContext.new(4, HalfEven))
assert(third.to_str() == "0.3333")
```

## Представление

```nova
type Sign enum Neg | Zero | Pos

type BigInt value {
    sign Sign
    limbs []u32
}
```

- Value-record на стеке: копия = Sign-clr + slice-header (~24 байта).
- Данные лимбов — на GC-куче, разделяются при копировании.
- Иммутабельность гарантирует отсутствие alias-мутаций.
- `==` — структурное (value-record, D328).
- Нормализация: нет ведущих нулевых лимбов; ноль = `Zero` + пустые limbs (каноничен).

## Умножение

Карацуба `O(n^log₂3)` с переключением на школьное `O(n²)` при n < 16 лимбов.

## Деление

Алгоритм Кнута D (нормализация делителя, догадка частного с коррекцией, денормализация остатка). Trunc-к-нулю (паритет `int`/C99/Rust).

## Подключение

```toml
[dependencies]
bigint = { git = "https://github.com/nv-lang/nova-bigint", version = "0.1" }
```

```nova
import bigint.{BigInt, Sign, ParseBigIntError, DivError}

let a = 42.to_bigint()
let b = "12345678901234567890".to_bigint()!!
assert(a + b > a)
```

## Границы применимости

- Все аллокации — на GC-куче; каждая иммутабельная операция создаёт новый `BigInt`.
- Мутирующих `@add_assign`/`@mul_assign` в V1 НЕТ осознанно: они ломали бы гарантию иммутабельности value-record (копия разделяет буфер лимбов — мутация испортила бы «неизменяемую» копию), а обёртка над `@plus` никакой экономии не даёт. Buffer-reuse — задача V2 (либо внутри самих операций, либо отдельный мутабельный тип-аккумулятор).
- Операторы `/` и `%` через desugar НЕ работают (возвращают `Result`). Используйте явные `@div_rem`/`@div`/`@rem`.

## Тестирование

```sh
nova test src
```

## Лицензия

MIT OR Apache-2.0
