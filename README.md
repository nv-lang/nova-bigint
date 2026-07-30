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
- **Мутирующие операции:** `add_assign`, `sub_assign`, `mul_assign` — переиспользуют буфер лимбов, ноль лишних аллокаций в циклах.
- **Ошибки — значениями:** деление на ноль → `Err(DivisionByZero)`, разбор строки → `Err(ParseBigIntError)` с категорией (`Empty`, `InvalidCharacter`, `OnlySign`). `panic` только на внутренние инварианты.

## Чего НЕТ в V1

- `gcd`, `modpow`, битовые операции (`&`, `|`, `<<`, `>>`)
- `BigDecimal`, `BigRat`, `BigFloat`
- Неявные коэрсии `int → BigInt`
- Литеральная форма `123456789012345678901n`
- Toom-3 умножение

Всё перечисленное — кандидаты в V2 по отдельным решениям.

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
- Для циклов используйте `@add_assign`/`@mul_assign` — они переиспользуют буфер лимбов.
- Операторы `/` и `%` через desugar НЕ работают (возвращают `Result`). Используйте явные `@div_rem`/`@div`/`@rem`.

## Тестирование

```sh
nova test src
```

## Лицензия

MIT OR Apache-2.0
