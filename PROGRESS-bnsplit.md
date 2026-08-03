# PROGRESS — окно p-bnsplit: BigInt в свой модуль

Бриф: `d:/Sources/nv-lang/nova/scratch38/BRIEF_bnsplit.md`. Рабочая копия:
`d:/Sources/nv-lang/nova-bnsplit`, ветка `p-bignum-split`.

## Задача

До правки `src/bignum.nv` (`module bignum`) нёс И общие типы семьи
(`Sign`, `ParseNumberError`), И весь `BigInt` (тип, методы, `DivError`,
приватные хелперы школьного/карацубного умножения и деления Кнута) — артефакт
механического переименования пакета bigint → bignum. Асимметрично: у
`bigrat`/`bigfloat`/`bigdecimal` — каждый в своём модуле, а BigInt лежал в
корне вместе с общим.

## Что сделано

- `src/bignum.nv` пересобран заново: теперь несёт ТОЛЬКО общее —
  `Sign` (`Neg | Zero | Pos`) и `ParseNumberError` (8 вариантов). Без
  импортов (типы самодостаточны).
- `src/bignum.nv` (старый, с BigInt) → `git mv` → `src/bigint.nv`,
  `module bignum.bigint`; добавлен `import bignum.{Sign, Neg, Zero, Pos,
  ParseNumberError, Empty, OnlySign, InvalidCharacter}` для собственных нужд
  (конверсии/парсинг). `DivError`/`DivisionByZero` остались в этом файле —
  они принадлежат BigInt, не общей семье.
- `src/bignum_test.nv` → `git mv` → `src/bigint_test.nv`. Ловушка D78: этот
  файл — ОТДЕЛЬНЫЙ sibling-модуль `bignum.bigint_test` (не может делить
  `module bignum.bigint` с `bigint.nv`, будучи отдельным флэт-файлом — то же
  правило "папка=один модуль из co-equal файлов", что и у
  bigdecimal/bigfloat/bigrat + их `_test`-сиблингов). Экспортируемых
  `BigInt`/`Sign`/`ParseNumberError`-вариантов явно импортировать НЕ
  пришлось: `match`-паттерны на голых именах вариантов (`Zero`, `Neg`,
  `DivisionByZero`, ...) резолвятся по ожидаемому типу скрутини БЕЗ импорта
  (подтверждено гейтом: импорт вариантов давал `unused-import`); явно нужен
  был только сам `BigInt` (вызовы `BigInt.zero()` и т.п.).
- `src/bignum_bench.nv` → `git mv` → `src/bigint_bench.nv`, тот же
  sibling-модуль `bignum.bigint_bench`. Верификационный тест сверяет
  `karatsuba_mul` против `school_mul` — обе были приватными хелперами
  `bignum.nv`. После раскола на sibling-модули (а не merged-peer, как было у
  старого `bignum`/`bignum_test`/`bignum_bench` под одним именем `bignum`)
  прямой доступ пропал. Решение: `export` у `school_mul`/`karatsuba_mul` в
  `bigint.nv` — с комментарием, что это исключительно ради кросс-модульной
  bench-сверки, не часть стабильного публичного API семейства. Тест НЕ
  ослаблен и не удалён — просто восстановлена видимость.
- Импорты `bignum.{BigInt, ...}` разъехались на `bignum.{Sign/ParseNumberError-варианты}`
  + `bignum.bigint.{BigInt, ...}` в 9 файлах: `bigdecimal.nv`, `bigfloat.nv`,
  `bigrat.nv`, `bigrat_slow.nv`, `bigdecimal_test.nv`, `bigfloat_test.nv`,
  `bigrat_test.nv`, `bigint.nv`, `bigint_test.nv` (плюс `repro_parse_test.nv`
  — там `BigInt` не понадобился явно, метод `to_bigint()` резолвится и без
  него). Итого поправлено **10 import-строк** в исходниках + пример импорта
  в README.md (`import bignum.{BigInt, Sign, ParseNumberError, DivError}` →
  два импорта: `bignum.{Sign, ParseNumberError}` +
  `bignum.bigint.{BigInt, DivError}`).
- Грепом `bignum.{` по всему `src/` и `README.md` — не осталось ни одного
  неразъехавшегося импорта (проверено после финальной правки).
- Скопирован (не закоммичен, по конвенции) `./nova.sh` из `nova-bigint` —
  в свежей рабочей копии его не было.

## Гейты (дословно)

`./nova.sh check src --strict-effects`: **PASS 14 / FAIL 0 / WARN 16**
(до правки: 13/0/16 — все 16 warning'ов ТЕ ЖЕ самые unused-import,
предсуществовавшие в bigfloat/bigrat_test/bigdecimal_test/repro_parse_test;
PASS +1 — исключительно от появления лишнего файла из-за раскола
bignum.nv на два: bignum.nv + bigint.nv).

`./nova.sh test src`: **PASS 8 / FAIL 0 / SKIP 6** (до правки: 9/0 SKIP4).
Сдвиг ОБЪЯСНИМ структурно, не потерей покрытия: раньше `bignum.nv` (корень,
без своих test-блоков) и `bignum_test.nv` делили ОДНО имя модуля `bignum`
(merged-peer), поэтому раннер считал файл `src/bignum` PASS "по соседству".
После раскола `bigint.nv`/`bignum.nv` — sibling-модули со своими
`_test`-файлами (`bigint_test.nv`, ...), как уже было устроено у
bigdecimal/bigrat/bigfloat. Теперь `bigint.nv` и `bignum.nv` сами по себе
без test-блоков и честно попадают в SKIP "no test blocks" — ровно тот же
паттерн, что уже был у bigdecimal.nv/bigrat.nv/bigfloat.nv. Новые SKIP:
`bignum`, `bigint` (было 3 SKIP: bigdecimal/bigrat/bigfloat + slow-lane
bigrat_slow = 4; стало 4 + эти два = 6). Все прежние ассерты по-прежнему
выполняются и проходят — просто внутри `bigint_test`/`bigint_bench` (PASS),
не потеряны.

`./nova.sh lint src`: **14 файлов, 0 findings** (до правки: 13 файлов,
0 findings — файл прибавился от раскола, находок не прибавилось).

## Дефекты компилятора

Новых номеров не заводил. Один нюанс поведения зафиксирован в этом отчёте
(не дефект): `match`-паттерны на голых enum-вариантах резолвятся по типу
скрутини без явного импорта имени варианта — асимметрично с обычными
identifier-выражениями (`x == Zero` требует импорт `Zero`, а
`match x { Zero => ... }` — нет). Полезно знать при будущих module-расколах
в этом пакете.
