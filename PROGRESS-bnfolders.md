# PROGRESS — окно p-bnfolders: bignum-семья на папки-модули

Бриф: `d:/Sources/nv-lang/nova/scratch38/BRIEF_bnfolders.md`. Рабочая копия:
`d:/Sources/nv-lang/nova-bnfolders`, ветка `p-bignum-folders` (от `main`
`b9b398e`). В `main` не вливать, не пушить — сдаётся интегратору.

## Задача

Каждый член семьи (BigInt/BigDecimal/BigFloat/BigRat) был ФАЙЛ-модулем, а его
тесты/верификация лежали СОСЕДНИМИ файлами — но с ОТДЕЛЬНЫМ `module`
(`bignum.bigint` vs `bignum.bigint_test` vs `bignum.bigint_bench` и т.д.),
т.е. чужими модулями друг другу. Из-за этого приватные хелперы (`school_mul`,
`karatsuba_mul`) пришлось искусственно `export`-ить только ради
кросс-модульной сверки в верификационном тесте (окно p-bnsplit).

Образец — `std/src/collections/vec/` в репозитории `nova`: `_module.nv` +
co-equal peers, все с ОДНИМ `module collections.vec`.

## Что сделано

Все переезды — `git mv` (не копирование). Путь модуля НЕ изменился нигде
(`bignum.bigint` остаётся `bignum.bigint` и т.д.) — только физическая
раскладка файлов и совпадение `module`-строки у всех peers папки.

### bigint/ (module bignum.bigint)

- `src/bigint.nv` → `src/bigint/core.nv`.
- `src/bigint_test.nv` → `src/bigint/core_test.nv`; `module` сменён с
  `bignum.bigint_test` на `bignum.bigint`; убран самоимпорт
  `import bignum.bigint.{BigInt}` (тот же модуль теперь).
- `src/bigint_bench.nv` → `src/bigint/mul_equiv_test.nv` — **не `bench.nv`**:
  файл не использует настоящий bench-синтаксис (`bench "..." { measure {...} }`,
  см. `bench/micro/hello.nv` в репе `nova`), а состоит из обычных `test`-блоков,
  сверяющих `karatsuba_mul` со `school_mul` на совпадение результата. Значит
  это тест эквивалентности, а не бенчмарк замера скорости — назвать его
  `bench.nv` было бы враньём про содержимое. `module` сменён на
  `bignum.bigint`, убран самоимпорт `import bignum.bigint.{karatsuba_mul,
  school_mul}` — обе функции теперь видны напрямую как приватные пиры.
- `school_mul`/`karatsuba_mul` в `core.nv`: **`export` снят**, обе снова
  `fn` (приватные module-private). Сняты и комментарии
  «`export` только ради кросс-модульной bench-сверки» — они больше не
  описывают реальность.
- `src/bigint/_module.nv` создан (по образцу `vec/_module.nv`): поясняющий
  комментарий о модели папки-модуля + `module bignum.bigint`, без items.

### bigdecimal/ (module bignum.bigdecimal)

- `src/bigdecimal.nv` → `src/bigdecimal/core.nv`.
- `src/bigdecimal_test.nv` → `src/bigdecimal/core_test.nv`; `module` сменён
  с `bignum.bigdecimal_test` на `bignum.bigdecimal`; убран самоимпорт
  `import bignum.bigdecimal.{BigDecimal, MathContext, RoundingMode, ...}`
  (перекрёстный `import bignum.bigint.{BigInt}` — оставлен, это чужой модуль).
- `src/bigdecimal/_module.nv` создан.

### bigfloat/ (module bignum.bigfloat)

- `src/bigfloat.nv` → `src/bigfloat/core.nv`.
- `src/bigfloat_test.nv` → `src/bigfloat/core_test.nv`; `module` сменён с
  `bignum.bigfloat_test` на `bignum.bigfloat`; убран самоимпорт
  `import bignum.bigfloat.{BigFloat, PrecisionContext, ...}` (`import
  bignum.bigint.{BigInt}` — оставлен).
- `src/bigfloat/_module.nv` создан.

### bigrat/ (module bignum.bigrat)

- `src/bigrat.nv` → `src/bigrat/core.nv`.
- `src/bigrat_test.nv` → `src/bigrat/core_test.nv`; `module` сменён с
  `bignum.bigrat_test` на `bignum.bigrat`; убран самоимпорт
  `import bignum.bigrat.{BigRat}` (кросс-модульные `bignum.bigint`,
  `bignum.bigdecimal`, `bignum.bigfloat`, `bignum` — оставлены).
- `src/bigrat_slow.nv` → **`src/bigrat/core_slow.nv`**, НЕ `slow_test.nv`,
  как было в исходной раскладке брифа. Причина: slow-lane (Plan 156/D376)
  детектится по `is_slow_file_stem(stem) = stem.ends_with("_slow")`
  (`compiler-codegen/src/test_runner.rs`) — суффикс `_slow` ОБЯЗАН быть
  самым внешним токеном имени файла (канонический порядок
  `<core>[_<os>][_test][_slow]`). `slow_test.nv` кончается на `_test`, а не
  на `_slow`, и молча выпал бы из slow-lane — файл начал бы гоняться в
  дефолтном `nova test` вместо `--include-slow`. `core_slow.nv` сохраняет
  и суффикс `_slow`, и парность имени с `core.nv`/`core_test.nv`. `module`
  сменён с `bignum.bigrat_slow` на `bignum.bigrat`; убран самоимпорт
  `import bignum.bigrat.{BigRat}`.
- `src/bigrat/_module.nv` создан.

### Что НЕ тронуто

- `src/bignum.nv` (module `bignum`, только `Sign`/`ParseNumberError`) —
  общее семьи, вне scope переезда, брифом прямо исключено.
- `src/repro_direct_test.nv`, `src/repro_parse_test.nv`, `src/repro_test.nv`
  — не члены bignum-семьи (репро-тесты несвязанной фичи компилятора,
  свой `module bignum.repro_*`), брифом не упомянуты, оставлены как есть.
- Импорты в других файлах пакета путей `bignum.bigint`/`bignum.bigdecimal`/
  `bignum.bigfloat`/`bignum.bigrat` — не менялись (путь модуля не менялся,
  править было нечего). `README.md`, `nova.toml` — не тронуты (тот же путь
  модуля; `nova.toml` не перечисляет модули явно, резолв from `src/`).
- `nova.sh` — скопирован из `nova-bigint` (env-обёртка для гейтов), но НЕ
  добавлен в git: в `nova-bigint` он тоже untracked (`?? nova.sh`),
  копия — чисто рабочее удобство этого окна.

## Проверка успеха

- `school_mul`/`karatsuba_mul` — приватные (`fn`, без `export`) в
  `src/bigint/core.nv`. Верификационный тест `src/bigint/mul_equiv_test.nv`
  видит их напрямую, без импорта — цель брифа достигнута.
- Число `test`-блоков: 252 (было 252, `main` vs рабочая копия — идентично).
  Число `assert(`-вызовов: 726 (было 726) — не уменьшилось.

## Гейты (вердикты дословно — см. REPORT)

`./nova.sh test src`, `./nova.sh check src --strict-effects`,
`./nova.sh lint src` — все зелёные, числовой сдвиг объяснён построчно
в финальном отчёте (папки-модули схлопывают несколько file-модулей в
ОДИН compile-unit — меньше строк PASS/SKIP при том же наборе тестов).
