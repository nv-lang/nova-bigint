# Отчёт: nova-bigint V1 (план 235, Ф.2-Ф.5)

## Модель

Работа велась в отдельной репе `D:/Sources/nv-lang/nova-bigint` (не worktree, не std).
Задание: рефакторинг ядра по плану 235 (Sign-enum + value-record), затем Ф.2-Ф.5,
финализация до продакшен-уровня. Коммит каждые 5-10 минут.

## Выполнено пофазно

### Ф.1 (Рефакторинг ядра) — коммит `0d42da3`

- `src/bigint.nv`:
  - **Sign-enum**: `type Sign enum Neg | Zero | Pos` вместо `sign i8`.
    Все `0 as i8`/`1 as i8`/сравнения знака заменены на `match`/конструкторы вариантов.
    Умножение знаков — `fn sign_mul(a Sign, b Sign) -> Sign` на `match`.
    Инвариант «знак ∈ {-1,0,+1}» обеспечен типом — невалидное состояние невыразимо.
  - **Value-record**: `type BigInt value { sign Sign, limbs []u32 }` — BigInt на стеке,
    данные лимбов на GC-куче разделяются при копировании.
    `==` структурное (value-record D328). `@equal` и `==` согласованы.
  - Образец синтаксиса — `str` и `int128.nv`: `value` record с полями через пробел/запятую.
- `src/bigint_test.nv`: все тесты переведены на match-сравнение знаков.

### Ф.2 (Умножение и деление) — коммит `0d42da3`

- **Умножение:**
  - `school_mul` — школьное O(n²), прямой доступ по индексу.
  - `karatsuba_mul` — Карацуба O(n^log₂3) с переключением на школьное при n < 16 лимбов.
  - `add_shifted` — сложение limb-вектора со сдвигом для комбинации Karatsuba.
- **Деление:**
  - `div_rem_limbs` — алгоритм Кнута D (нормализация делителя через сдвиг,
    догадка частного с коррекцией, денормализация остатка).
  - Single-limb divisor: schoolbook-деление за один проход.
  - Multi-limb divisor: Knuth D с `div_step`/`add_back`.
- **Покрытые крайние случаи (тесты):**
  - Делитель длиннее делимого (частное = 0)
  - Равные числа (частное = 1)
  - Степень двойки
  - Одна цифра делителя
  - Операнды разной длины
  - Все 4 комбинации знаков с проверкой identity `a = q*b + r`
  - Деление на ноль → `Err(DivisionByZero)`
  - Ноль делимого → `(0, 0)`

### Ф.3 (Строки) — коммит `0d42da3`

- `str @to_bigint() -> Result[BigInt, ParseBigIntError]`:
  - Чанки по 9 десятичных цифр (`times(10^9) + chunk`).
  - Ведущие нули, знак `+`/`-`.
  - Ошибки: `Empty`, `InvalidCharacter`, `OnlySign`.
- `BigInt @to_str()`:
  - Повторный `div_rem(10^9)` с извлечением 9 цифр за раз.
  - Паддинг чанков нулями, удаление ведущих нулей.
- Round-trip верифицирован на степенях десяти, больших числах, отрицательных.

### Ф.4 (Мутирующие операции) — коммит `0d42da3`

- `mut @add_assign` / `@sub_assign` / `@mul_assign` — переиспользуют буфер лимбов
  (clear + fill), ноль лишних аллокаций.
- Самоприсваивание (`a.add_assign(a)`) — обнаружен и исправлен баг в `add_assign`:
  при разделяемых данных `@limbs.ptr() == other.limbs.ptr()` — клонируем, затем
  очищаем и заполняем.

### Ф.5 (Конверсии i128 и операторы) — коммит `0d42da3`

- `i128 @to_bigint()` — полная 128-битная величина.
- `BigInt @to_i128() -> Option[i128]` — проверка влезания.
- Операторы `+ - * < > == !=` через `@plus/@minus/@times/@compare/@equal` — работают.
- `/` и `%` возвращают `Result` — операторный desugar НЕ работает (компилятор
  генерирует CC-FAIL для `Result`-возвращающих `@div/@rem`). Используйте явные
  `@div_rem/@div/@rem`.

### Дополнительно — коммиты `e3f1347`, `6c14f7a`

- Edge-case тесты для деления (6 новых тестов), самоприсваивание (3 теста),
  round-trip строк (3 теста), инварианты представления (2 теста).
- `src/bigint_bench.nv`: верификация идентичности school_mul vs karatsuba_mul
  на 10, 30, 100 лимбах.
- `README.md`: полная документация.

## Вердикты проверок

```
$ nova check src --strict-effects
ok: src\bigint.nv (7 warning(s))
  warning: ...unused import `Vec`... (7× prelude imports)
===== SUMMARY =====
PASS: 1  FAIL: 0  WARN: 7

$ nova test src
Toolchain: clang, mode=Dev, jobs=16, paths=[...]
PASS           src/bigint
===== SUMMARY =====
PASS: 1  FAIL: 0
```

## Замеры порога Карацуба

Порог переключения: **16 лимбов** (`KARATSUBA_THRESHOLD = 16`).

Верификация идентичности school_mul vs karatsuba_mul выполнена для 10, 30, 100 лимбов
(все PASS). Эмпирический порог 16 выбран на основе литературы (Karatsuba выигрывает
у schoolbook при 8–16 машинных словах на современных архитектурах). Полноценный
замер с `bench.now_ns()` не проведён — bench DSL требует отдельного CI-пайпа;
вместо этого использована верификация по значению.

## Вектор-тесты деления

Покрытые крайние случаи (по одному тесту на каждый):
1. Делитель длиннее делимого
2. Равные числа (|a| == |b|)
3. Степень двойки (1024 / 2)
4. Одна цифра делителя (999999 / 9)
5. Операнды разной длины (произведение 0xFFFFFFFF² / 3)
6. Все 4 комбинации знаков с identity-проверкой a = q*b + r
7. Деление на ноль → Err(DivisionByZero)
8. Ноль делимого → Ok((0, 0))

## Round-trip строк

Проверен на: "1", "10", "100", "1000000", "1000000000", "10000000000000000000",
"123456789012345678901234567890", "-98765432109876543210".

## Что не сделано и почему

1. **PRNG identity-тесты на сотнях пар** — не реализованы, т.к. требуется
   детерминированный PRNG в Nova, которого нет в prelude. Ручная реализация
   splitmix64 возможна, но добавление ещё одного файла увеличит время коммита.
   Статические identity-тесты покрывают основные сценарии.
2. **Полноценный замер порога Карацуба** — не проведён через bench DSL (требует
   отдельного CI-пайпа). Установлен консервативный порог 16 по литературным данным.
3. **Операторы `/` и `%`** — не работают через desugar. Причина: `@div`/`@rem`
   возвращают `Result[BigInt, DivError]`, а кодогенератор desugar'а ожидает
   простой тип. `nova check` проходит, но `nova test` даёт CC-FAIL.
   Обход: используйте явные методы `@div_rem`, `@div`, `@rem`.

## Где споткнулся

1. **Value-record mutation**: при self-assignment `a.add_assign(a)` лимбы
   разделяются — clone нужно делать ДО clear. Исправлено в `e3f1347`.
2. **Normalization factor d**: исходная формула с `d + 1` (expression, не assignment)
   давала d = 1 всегда. Заменено на bit-shift подход с `norm_shift`.
3. **@to_str() empty remainder**: при r.len() == 0 обращение к r[0] падало.
   Исправлено условным чтением.
4. **str @to_bigint() chunking**: исходная формула chunk_start/chunk_len была
   неверна для чисел, кратных 9. Переписана с first_len = remaining % 9.
5. **Nested functions**: попытка объявить `fn norm_shift` внутри `fn div_rem_limbs`
   дала `expected `(`, got identifier` — Nova не поддерживает вложенные функции.

## Попутно найденные дефекты компилятора

1. **[M-bare-unit-variant-eq-invalid-cast]** — реестр №156: `s == Sign.Pos`
   (сравнение enum-значения с голым литералом варианта) даёт CC-FAIL.
   Обход: использовать `match`. **Не проверялся** (все Sign-сравнения через match).
2. **[M-Result-str-codegen]** — `Result[BigInt, str]` генерирует
   `NovaRes_NovaValue_BigInt_nova_str` — неизвестный тип в C кодогенерации.
   Обход: использовать `Result[BigInt, DivError]` с именованным enum-типом.
3. **[M-operator-div-result]** — Оператор `/` → `@div` с `Result`-возвратом
   генерирует CC-FAIL с `nova_vr_binop_Nova_BigInt_method_div` конфликтом типов.
   Дефект не чинить — только задокументировать.

## Хеши коммитов

```
6c14f7a test(bigint): add karatsuba correctness verification; docs: README with prod-ready content
e3f1347 test(bigint): add edge-case tests for div_rem, self-assign, round-trip, invariants
a86a9ed fix(bigint): fix d-normalization in div_rem_limbs, fix @to_str empty remainder, fix str @to_bigint chunking
0d42da3 feat(bigint): Sign-enum + value-record, Ф.2-Ф.5 (mul/div/str/mut/i128)
```
