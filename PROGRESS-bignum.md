# PROGRESS — план 243, Ф.U (окно p-bignum)

Исполнитель: sonnet, ветка `p-bignum`, worktree `d:/Sources/nv-lang/nova-bignum`
(создан от `nova-bigint` main, коммит `3361382`).
Компилятор: `d:/Sources/nv-lang/nova/nova-cli/target/release/nova.exe` через
обёртку `./nova.sh` (не коммитится — уже была в главном чекауте nova-bigint,
скопирована в worktree).

Базовая линия (до правок, все три гейта, полный вывод без усечения):
`nova check src --strict-effects` = 13/0/38, `nova test src` = 9/0 SKIP 4,
`nova lint src` = 0 findings.

| Пункт | Статус | Итог |
|---|---|---|
| У.1 Единый `ParseNumberError` | ЗЕЛЁНЫЙ | 4 перечисления → 1, `map_parse_int_err` и его двойник в repro_parse_test.nv удалены |
| У.2 `BigRat @sign() -> Sign` (№297) | ЗЕЛЁНЫЙ | возвращает `Sign`; `@compare() -> int` не тронут |
| У.3 Переименование пакета в `bignum` | ЗЕЛЁНЫЙ | манифест, корневой модуль+все импорты, README |

## У.1 — единый тип ошибок разбора

`export type ParseNumberError enum Empty | OnlySign | InvalidCharacter | MultiplePoints | MultipleExponents | EmptyExponent | EmptyMantissa | ZeroDenominator`
объявлен в корневом модуле (`bignum.nv`, ранее `bigint.nv`) — канон D406,
без ведущей `|`. Локальные `ParseBigIntError`/`ParseBigRatError`/
`ParseBigDecimalError`/`ParseBigFloatError` удалены целиком (не сохранены
как алиасы — пакет не выпущен).

`map_parse_int_err` (bigrat.nv) и его двойник `map_err` (repro_parse_test.nv)
удалены: с единым типом ошибки `str @to_bigint()` и `str @to_bigrat()`/
`str @to_bigdecimal()`/`str @to_bigfloat()` возвращают ОДИН И ТОТ ЖЕ
`Result[_, ParseNumberError]`, поэтому `?`/`Err(e) => Err(e)` работает без
адаптера. Аналогично в bigfloat.nv `str @to_bigfloat()` больше не переносит
`ParseBigDecimalError` в `ParseBigFloatError` через identity-match — просто
`Err(e) => Err(e)`.

Побочная находка при чистке импортов: match-паттерны (`Err(OnlySign) => ...`)
резолвят имя варианта СТРУКТУРНО через тип (импортированный `ParseNumberError`
или инференс из типа scrutinee) — отдельный импорт варианта нужен ТОЛЬКО для
value-позиции (`Err(Empty)` КАК КОНСТРУКТОР). Использовал это, чтобы почистить
импорты тестовых файлов до фактически нужного минимума — итоговый `check`
WARN меньше канона (16 против 38), подробности в разделе «Гейты».

## У.2 — знак к канону (№297)

`BigRat @sign() -> Sign` (было `int`), реализация — `if @num.is_neg() { Neg }
else if @num.is_zero() { Zero } else { Pos }`. Тесты (`bigrat_test.nv`)
переведены на `== Zero/Pos/Neg`. `@compare() -> int` НЕ тронут.

`sign_num`/`apply_sign` в bigfloat.nv: `apply_sign` уже принимал `Sign` (не
было разнобоя, оставлен как есть — используется). `sign_num(b BigInt) -> int`
был МЁРТВЫМ кодом (0 вызовов) — удалён. `sign_num`/`apply_sign` в
bigdecimal.nv — активно используются (внутренняя int-арифметика знака,
никак не связаны с публичным `BigRat @sign()`) — НЕ тронуты (вне описания
дефекта №297).

№296 (числовые дискриминанты Sign недоступны) — не обходил, не трогал.

## У.3 — переименование пакета в `bignum`

Сделано ВНУТРИ репозитория (без трогания трёх зеркал/потребителей — это
интегратору):

- `nova.toml`: `name = "bigint"` → `name = "bignum"`; описание расширено на
  всю семью; `repository = ".../nova-bigint"` — URL НЕ менял (репозиторий
  физически ещё не переименован, смена URL раньше факта была бы враньём).
- Корневые peer-файлы модуля физически переименованы (иначе
  `E_MODULE_FILE_ORPHAN` — компилятор требует, чтобы head-файл файлового
  модуля совпадал именем с последним сегментом пути):
  `bigint.nv` → `bignum.nv`, `bigint_test.nv` → `bignum_test.nv`,
  `bigint_bench.nv` → `bignum_bench.nv` (`git mv`, история сохранена).
- Все `module bigint`/`module bigint.X` → `module bignum`/`module bignum.X`
  во всех 13 файлах пакета.
- Все `import bigint...` → `import bignum...` во всех файлах.
- Комментарии, упоминающие путь/имя модуля (`bigint.nv`, «из bigint», «в
  bigint.bigrat» и т.п.) — тоже поправлены (грепнул `\bbigint\b` до нуля
  совпадений, `BigInt`-тип не затронут — разный регистр).
- README.md: заголовок, описание, все примеры кода (`import bigint...` →
  `import bignum...`), упоминания `ParseBig*Error` → `ParseNumberError`,
  строка подключения зависимости (`bigint = {...}` → `bignum = {...}`,
  git-URL оставлен как есть по той же причине, что и в манифесте).
- PROGRESS.md/REPORT.md/REPORT-p240.md (старые сессионные отчёты плана 240) —
  НЕ трогал: это исторический журнал «что было сделано под именем на тот
  момент», переписывать задним числом = искажение истории.

### Остаётся ИНТЕГРАТОРУ (не входит в это окно)

- Переименование самого репозитория на трёх зеркалах (github/gitverse/
  sourcecraft): `nova-bigint` → предположительно новое имя (или оставить URL
  репозитория как есть и просто переименовать пакет внутри — решение
  владельца).
- Правка потребителей: `nova.toml`/`Cargo.toml`-подобные манифесты других
  пакетов, зависящих от `bigint` (сейчас поле `name = "bignum"` — зависимость
  по имени пакета сломается, пока потребители не обновят `bignum = {...}`).
- `examples/nova.lock.toml` и подобные файлы блокировок примеров в репе
  `nova`, если они фиксируют пакет `bigint`.
- Ссылки на `bigint`/nova-bigint в планах `nova` (235/236/237/240/243) — если
  где-то говорится "пакет bigint", после факта переименования репозитория
  нужно сверить.
- Git-URL в `nova.toml`/README указывает на `nova-bigint` — обновить ПОСЛЕ
  фактического переименования репозитория на зеркалах (иначе URL будет
  битым до того момента).

## Гейты (дословно, после ВСЕХ трёх пунктов)

```
./nova.sh check src --strict-effects
PASS: 13  FAIL: 0  WARN: 16      (канон 13/0/38 — WARN лучше, см. ниже)

./nova.sh test src
PASS: 9  FAIL: 0  SKIP: 4        (== канон)

./nova.sh lint src
lint: 13 file(s), 0 finding(s)   (== канон, ноль удержан)
```

WARN 16 против канона 38 — НЕ регрессия, а чистка: (а) удаление мёртвого
`sign_num` в bigfloat.nv, (б) удаление одного давно-мёртвого импорта
`ZeroDenominator` в repro_parse_test.nv (был неиспользуем и до этого окна),
(в) в тестовых файлах пришлось выяснить, что match-паттерны не требуют
импорта варианта по имени (см. У.1) — и почистить импорты `Parse*`-семьи
и `Sign`-вариантов до фактически используемых, что убрало ~20
unused-import warning'ов, которые раньше маскировались чуть другой формой
(были на старые имена типов, а не на новое). Полный до/после лог сверен
построчно (не только по итоговой сумме) через параллельный baseline-worktree
на исходном коммите `3361382` — распечатки сохранены в
`d:/Sources/nv-lang/nova/scratch38/{baseline_check.txt,final_check.txt}`.
