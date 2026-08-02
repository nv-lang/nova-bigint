# СТОП-репорт — План 240 (BigRat V1), фаза Ф.0

## Модель

Задание: план 240 (`d:/Sources/nv-lang/nova/docs/plans/240-bigrat.md`) — точные
рациональные поверх BigInt; работа в `d:/Sources/nv-lang/nova-bigint-p240`, ветка `p240`.
Исполнитель: opencode/big-pickle.

Окружение (обязательное для гейтов из листовой репы):

```
NOVA_STD_PATH=d:/Sources/nv-lang/nova/std
NOVA_RT_DIR=d:/Sources/nv-lang/nova/compiler-codegen/nova_rt
NOVA_GC_LIB_DIR=d:/Sources/nv-lang/nova/compiler-codegen/vcpkg_installed/x64-windows-static/lib
NOVA_CG_INCLUDE=d:/Sources/nv-lang/nova/compiler-codegen
```

## Хеши

```
компилятор: d:/Sources/nv-lang/nova/nova-cli/target/release/nova.exe
sha256: 5e395ebed85d01daae3118bf9cf8ad1431a9180d85db08e2d181f630aece067f
базовая линия ветки p240 (HEAD до работ): 1ef06b1
```

## Статус фаз

| Фаза | Статус |
|---|---|
| Ф.0 Пины | **КРАСНЫЙ — СТОП** (дефект компилятора, пин «б») |
| Ф.1 gcd в bigint.nv | ожидает (не начинать до решения) |
| Ф.2–Ф.6 | ожидают |

## Вердикты гейтов

```
$ nova check src --strict-effects
PASS: 10  FAIL: 0  WARN: 24        # ok, статическая проверка проходит

$ nova test src/bigrat_test.nv
CC-FAIL  src/bigrat_test           # генерируемый C не компилируется

$ nova test src
CC-FAIL  src/bigrat_test           # пакетный гейт КРАСНЫЙ
```

Базовые каноны до работ: `check` 7/0/24, `test` 5/0/2 — подтверждены. После добавления
`src/bigrat.nv` + `src/bigrat_test.nv` + репро-файлов статический `check` вырос до
10/0/24 (новые файлы warning'ов не дают), рантайм-гейт `test` падает на CC-FAIL.

## Симптом

`nova test` на `src/bigrat_test.nv` падает на этапе компиляции сгенерированного C:

```
CC-FAIL src/bigrat_test
D:/Sources/nv-lang/nova-bigint-p240/src/bigrat_test.c:2343:115:
  error: passing 'NovaValue_BigRat' (aka 'struct NovaValue_BigRat')
         to parameter of incompatible type 'NovaValue_BigRat *'
```

Дефектные строки сгенерированного C (обёртки операторного desugar'а):

```c
// bigrat_test.c:2343-2346 — ВЫЗЫВАЮЩИЕ стабы (передают b по значению)
static nova_bool nova_vr_ueq_BigRat(NovaValue_BigRat a, NovaValue_BigRat b)
  { return Nova_BigRat_method_equal(&a, b); }
static NovaValue_BigRat nova_vr_binop_Nova_BigRat_method_plus(NovaValue_BigRat a, NovaValue_BigRat b)
  { return Nova_BigRat_method_plus(&a, b); }
// ... минус, times — та же форма

// bigrat_test.c:2149/2151, 11575/11597 — реальные сигнатуры методов (other по указателю)
static nova_bool Nova_BigRat_method_equal(NovaValue_BigRat* nova_self, NovaValue_BigRat* other);
static NovaValue_BigRat Nova_BigRat_method_plus(NovaValue_BigRat* nova_self, NovaValue_BigRat* other);
```

## Корневая причина (дефект codegen компилятора)

1. Plan 172.14 (`auto-by-ref`): `ro`-параметр value-struct'а размером **>16 Б** по C-ABI
   эмитится как указатель `T*` (`param_is_auto_byref`, emit_c.rs:55568, порог `s > 16`
   на :55574; `value_struct_size_align` :55516). Карта строится пре-пассом
   `build_method_byref_map` (emit_c.rs:55707); прямые call-site методы используют её
   через `synthesize_method_byref_at_callsite` (:55810) / `synthesize_method_byref_args`
   (:55850, вызовы на :37907, :43503 и др.) — поэтому **прямые вызовы
   `a.plus(b)` / `a.equal(b)` на таком типе работают** (проверено, PASS).

2. BigInt = `{ Sign*, []u32* }` = 16 Б → порог не пробит, параметр остаётся по значению;
   BigRat = `{ BigInt, BigInt }` = 32 Б → **впервые в пакете** value-struct пробивает
   порог, и `other` в сигнатурах методов становится `NovaValue_BigRat*`.

3. Операторный desugar `+ - *` (emit_c.rs:34230-34306, Plan 175) и `==` (emit_c.rs:20942,
   Plan 172.4) генерируют **by-value обёртки** `nova_vr_binop_*`/`nova_vr_ueq_*`,
   беря тип параметра из `method_overloads` (сигнатура БЕЗ byref-флага) и вызывая метод
   как `{ return {c}(&a, b); }` (:34303, :34343 — второй аргумент жёстко по значению;
   для `==` :20937/:20956 `arg_is_ptr` читается из той же by-value сигнатуры). Флаг
   Plan 172.14 здесь НЕ консультируется → стаб передаёт `b` по значению туда, где
   объявлен `T*` → CC-FAIL.

**Вердикт по корню:** это дефект кодогенератора, а не ошибка в коде BigRat и не
нарушение пина Ф.0. Прямые методы зелёные; сломан ТОЛЬКО операторный desugar на
value-struct'ах, пробивающих порог 16 Б. Затронут пин (б) «операторный desugar `+ - *`».
По правилам BRIEF: компилятор не чинить, молча не обходить → СТОП.

## Минимальное репро

Самодостаточно (без BigInt), `src/repro_test.nv`:

```nova
module bigint.repro_test

type Pt value { x i64, y i64 }

fn Pt.new(x i64, y i64) -> Pt => { x, y }

type Rt value { a Pt, b Pt }          // C-размер 32 Б (>16 → auto-byref параметра)

fn Rt.new(a Pt, b Pt) -> Rt {
    { a, b }
}

fn Rt @plus(other Rt) -> Rt {
    Rt.new(Pt.new(@a.x + other.a.x, @a.y + other.a.y), Pt.new(@b.x + other.b.x, @b.y + other.b.y))
}

fn Rt @equal(other Rt) -> bool {
    @a == other.a && @b == other.b
}

test "value-record over 16B: operator and == desugar" {
    ro p1 = Pt.new(1, 2)
    ro p2 = Pt.new(3, 4)
    ro r1 = Rt.new(p1, p2)
    ro r2 = Rt.new(p1, p2)
    ro s = r1 + r2
    assert(s.a.x == 2)
    assert(s.b.y == 8)
    assert(r1 == r2)
    assert(s != r1)
}
```

Прогон:

```
$ nova test src/repro_test.nv
CC-FAIL src/repro_test
  src/repro_test.c:1964:120: passing 'NovaValue_Rt' to parameter of
  incompatible type 'NovaValue_Rt *'     # nova_vr_binop_Nova_Rt_method_plus(&a, b)
  src/repro_test.c:1965:99:  (аналогично)  # nova_vr_ueq_Rt(&a, b)
```

Контроль-эксперимент — те же методы, но прямыми вызовами (без операторов), `src/repro_direct_test.nv`:
`r1.plus(r2)` / `r1.equal(r2)` → **PASS**. Дефект локализован в операторном desugar'е.

Порог 16 Б провоцируется также любым value-struct'ом с 3+ скалярными полями
(`{ i64, i64, i64 }` = 24 Б); BigInt-семейство (16 Б) порог не пробивает — поэтому
прецеденты 236/237 и сам BigInt зелёные.

## Что сделано в Ф.0 до СТОП

- `src/bigrat.nv` — модуль `bigint.bigrat`: `type BigRat value { num BigInt, den BigInt }`,
  `ParseBigRatError`, `BigRat.zero/one/new`, `normalize_rat`, `@num()/@den()`, `@equal`,
  бланкет `T @to_bigrat()`, `i128 @to_bigrat()`, `@plus/@minus/@neg/@times`. Статический
  `check` — без warning'ов.
- `src/bigrat_test.nv` — тесты пинов Ф.0 (а/а2/б/в/г).
- `src/repro_test.nv`, `src/repro_direct_test.nv` — минимальное репро + контроль.
- `PROGRESS.md` — ведётся.

## Вердикты

1. **Пин (б) «операторный desugar `+ - *`» — КРАСНЫЙ** по дефекту компилятора
   (Plan 175 / Plan 172.4 обёртки не учитывают Plan 172.14 auto-byref на value-struct'ах
   >16 Б).
2. Пины (а) — компилябельность value-record с двумя BigInt-полями — статически зелёный;
   (а2) статик-конструкторы + `@equal` — зелёный при прямых вызовах; (в) кросс-модульный
   `DivError` и (г) бланкет `@to_bigrat` — статически подтверждены `check`, но их
   рантайм-проверка заблокирована тем же CC-FAIL (тесты используют операторы).
3. **Решение: СТОП.** Ф.1 (gcd) и последующие фазы не начинать до решения владельца по
   дефекту кодогенератора (фикс компилятора или санкционированный обход).

---

# Резюме 2026-08-02 — блокер ЗАКРЫТ, Ф.0 ЗЕЛЁНЫЙ, работа возобновлена

## Хеш компилятора (пересобран с фиксом реестр nova №274)

```
sha256: 017a24989c1c2c04862e32bc11bab341e2dfe9cebf547c0cbc20e1459063a006
```

## Перегон гейтов (сам, на worktree p240)

```
$ nova check src --strict-effects
PASS: 11  FAIL: 0  WARN: 24        # прежние 10 + bigrat/repros; warning'ов новых нет

$ nova test src
PASS: 8  FAIL: 0  SKIP: 3          # PASS 8 = 6 канонных + bigrat_test + repro_*;
                                    # SKIP 3 = модули без test-блоков (bigrat/bigdecimal/bigfloat)
```

Пин Ф.0 (а/а2/б/в/г) — **ЗЕЛЁНЫЙ**: CC-FAIL на `bigrat_test` и репро больше не воспроизводится
(фикс №274: auto-by-ref в операторных стабах `nova_vr_binop_*`/`nova_vr_ueq_*`).

## Решение

Блокер снят владельцем; компилятор чинить не пришлось (фикс в реестре nova). Ф.1–Ф.6
продолжаются строго по плану 240. Репро-файлы `src/repro_test.nv` / `src/repro_direct_test.nv`
после закрытия дефекта НЕ нужны — удаляются в Ф.6 перед приёмкой.

---

# СТОП-репорт — План 240 (BigRat V1), фаза Ф.4

Дата: 2026-08-02. Фазы Ф.1–Ф.3 закрыты (коммиты `9ee5aef`, `0df67a5`, `68ef71a`),
Ф.4 (строки и связка семьи) заблокирована дефектом кодогенератора.

## Хеш компилятора (тот же, что на Ф.0-резюме)

```
sha256: 017a24989c1c2c04862e32bc11bab341e2dfe9cebf547c0cbc20e1459063a006
```

## Симптом

```
$ nova test src/bigrat_test.nv
RUN-FAIL src/bigrat_test
  # FAIL: Ф.4: str @to_bigrat — ошибки парсинга — bigrat_test.nv:308: assert failed: false
  # FAIL: Ф.4: @to_str — p/q, целые без знаменателя, roundtrip — bigrat_test.nv:330: assert failed: false
```

Падают только тесты, которые матчат варианты `ParseBigRatError` (общие имена `Empty`/
`OnlySign`/`InvalidCharacter`). Тесты, возвращающие `Ok`-значения (`p/q`, целые формы),
проходят.

## Корневая причина (дефект codegen компилятора)

Вложенный паттерн `Err(Variant)` по `Result[_, E]` эмитится в C с тегом **не того**
enum'а: вариант разрешается по имени без учёта типа ошибки `E` результата. Когда
несколько enum'ов в юните/скоупе объявляют варианты с одним именем, кодогенератор берёт
«первый попавшийся» enum, а не `E`.

Конкретика (enum-теги в сгенерированном C):

| вариант | `ParseBigIntError` (тег) | `ParseBigRatError` (тег) |
|---|---|---|
| `Empty` | 0 | 0 |
| `OnlySign` | **2** | **1** |
| `InvalidCharacter` | **1** | **2** |

Пример, `src/repro_parse_test.c:14993` (паттерн `Err(OnlySign)` на
`Result[BigRat, ParseBigRatError]`):

```c
if (!_nv_matched_1868 && ((_nv_scr_1866->tag == NOVA_TAG_Result_Err) &&
    (_nv_scr_1866->payload.Err._0->tag == NOVA_TAG_ParseBigIntError_OnlySign))) ...
//                                                                  ^^^^^^^^^^^^^^^^^^
//      должен быть NOVA_TAG_ParseBigRatError_OnlySign (тег 1), а не ParseBigIntError (тег 2)
```

А значение, которое реально строит библиотека, — `ParseBigRatError.OnlySign` (тег 1):
`map_parse_int_err` (плоский match по `ParseBigIntError` со скрутини-типом, результат
типизирован как `ParseBigRatError`) разрешает ветки ПРАВИЛЬНО
(`repro_parse_test.c:12954-12962`: `nova_make_ParseBigRatError_*`). Тег 2 != тег 1 →
матч не срабатывает → `assert(false)`.

Почему тесты Ф.4 проходили до этого: `Empty`/`OnlySign`/`InvalidCharacter` в
`bigfloat`/`bigdecimal` строятся и матчатся **одинаково неверно** (обе стороны берут
`ParseBigIntError`/`Slot`), теги согласованы → зелёные. Например:
- `bigfloat_test.c:14725` — матч `Err(OnlySign)` на `ParseBigFloatError` тоже проверяет
  `NOVA_TAG_ParseBigIntError_OnlySign`;
- `bigfloat.nv` (маппинг ошибок, сген. `repro_parse_test.c:13543/13623/13630/...`) —
  конструкция `OnlySign => OnlySign` даёт `nova_make_ParseBigIntError_OnlySign()`.

В BigRat корректный плоский match-хелпер (`map_parse_int_err`) впервые развёл стороны:
значения строятся с тегами `ParseBigRatError`, а паттерны матчатся с тегами
`ParseBigIntError` → рассинхрон вскрыт.

Дополнительно дефект затронул и библиотеку: `return Err(Empty)` в `str @to_bigrat`
(bigrat.nv:211) эмитится как `Err(nova_make_Slot_Empty())` — enum `Slot`, а не
`ParseBigRatError` (спасает только совпадение тегов 0 == 0). `Err(ZeroDenominator)`
корректен — имя уникально.

**Вердикт по корню:** это дефект кодогенератора в разрешении вариантов enum'а при
вложенных `Err(Variant)`-паттернах и `Err(...)`-конструкции (игнорируется тип ошибки
Result; общие имена вариантов разрешаются в произвольный enum). Не ошибка в коде BigRat
и не нарушение пина Ф.4. По правилам BRIEF: компилятор не чинить, молча не обходить →
СТОП.

## Минимальное репро

Самодостаточно, `src/repro_parse_test.nv` (не зависит от BigInt-семантики):

```nova
module bigint.repro_parse_test

import bigint.bigrat.{BigRat, ParseBigRatError, Empty, OnlySign, InvalidCharacter, ZeroDenominator}
import bigint.bigfloat.{BigFloat}
import bigint.{ParseBigIntError}

fn map_err(e ParseBigIntError) -> ParseBigRatError {
    match e {
        Empty => Empty
        OnlySign => OnlySign
        InvalidCharacter => InvalidCharacter
    }
}

fn parse(s str) -> Result[BigRat, ParseBigRatError] {
    ro n = s.to_bigint()
    match n {
        Ok(v) => Ok(BigRat.new(v, BigRat.one().den())!!)
        Err(e) => Err(map_err(e))
    }
}

test "repro: nested Err(OnlySign) pattern on ParseBigRatError" {
    match parse("-") {
        Err(OnlySign) => assert(true)
        _ => assert(false)
    }
}

test "repro: nested Err(InvalidCharacter) pattern on ParseBigRatError" {
    match parse("a") {
        Err(InvalidCharacter) => assert(true)
        _ => assert(false)
    }
}
```

Прогон:

```
$ nova test src/repro_parse_test.nv
RUN-FAIL src/repro_parse_test
  # FAIL: repro: nested Err(OnlySign) pattern on ParseBigRatError — repro_parse_test.nv:26
  # FAIL: repro: nested Err(InvalidCharacter) pattern on ParseBigRatError — repro_parse_test.nv:33
```

Контроль: вариант с УНИКАЛЬНЫМ именем (`Err(ZeroDenominator)` на `ParseBigRatError`,
`Err(DivisionByZero)` на `DivError`) матчится корректно — подтверждено на
`bigrat_test.nv` (тесты Ф.3 зелёные) и `map_err`/`Empty` сходятся только совпадением
тегов 0.

## Что сделано в Ф.4 до СТОП

- `src/bigrat.nv`: `map_parse_int_err`, `str @to_bigrat()` (парсинг `p/q`, `den==0` →
  `ZeroDenominator`), `BigRat @to_str()`, `BigRat @to_int()`, `BigRat @to_i128()`,
  `BigDecimal @to_bigrat()`, `BigRat @to_bigdecimal(mc)`, `BigFloat @to_bigrat()`;
  статический `check` — без warning'ов.
- `src/bigrat_test.nv`: тесты Ф.4 (импорты вариантов `ParseBigRatError`, связка
  `BigDecimal`/`BigFloat`). `nova check src --strict-effects` = **PASS 11 / 0 / 24**.
- `src/repro_parse_test.nv` — минимальное репро. `src/diag_test.nv` — диагностический
  черновик, УДАЛЁН (не входит в репро).

## Вердикт

1. **Пин Ф.4 (строки, матч вариантов `ParseBigRatError`) — КРАСНЫЙ** по дефекту
   компилятора (вложенные `Err(Variant)`-паттерны с общими именами вариантов эмитят
   теги чужого enum'а). Ф.1–Ф.3 зелёные, `check` зелёный, `test src` с падающими Ф.4.
2. **Решение: СТОП.** Ф.5 и Ф.6 не начинать до решения владельца по дефекту
   кодогенератора (фикс компилятора или санкционированный обход). Репро-файл
   `src/repro_parse_test.nv` оставлен в дереве как доказательство; после закрытия
   дефекта удаляется в Ф.6 вместе с `repro_test.nv`/`repro_direct_test.nv`.

---

# Резюме 2026-08-02 — блокер Ф.4 ЗАКРЫТ, Ф.4 ЗЕЛЁНАЯ

## Хеш компилятора (пересобран с фиксами реестр nova №279/#280)

```
sha256: df16073e7900631d0459014ac59c0b0d56f8b12665b8ec87efa9a60c7dd1729c
```

## Перегон гейтов

```
$ nova test src/repro_parse_test.nv
PASS          src/repro_parse_test     # №279/#280 подтверждены на репро

$ nova test src
PASS: 9  FAIL: 0  SKIP: 3
```

Ф.4 (строки `p/q` и целые формы, ошибки парсинга, `@to_str`/`@to_int`/`@to_i128`,
связка BigDecimal↔BigRat, BigFloat @to_bigrat) — **ЗЕЛЁНАЯ** (коммиты `451a096`,
`c908b7c`). Репро-файлы удаляются в Ф.6.

---

# СТОП-репорт — План 240 (BigRat V1), фаза Ф.5

Дата: 2026-08-02. Компилятор — тот же, что на Ф.4-резюме
(`df16073e…1729c`). Блокер **закрыт в рамках Ф.5** через санкционированный обход
(документация дефекта; фикс компилятора №281/№282 не требуется владельцем — обход
в тестах).

## Симптом

Ф.5 (тесты и эталон-кейсы) добавлена, но `nova test src/bigrat_test.nv` виснет
(бесконечный цикл в тесте; harness убивает по таймауту):

```
$ nova test src/bigrat_test.nv --timeout 300
TIMEOUT src/bigrat_test   # killed after 304805ms
# user 0.1s / sys 0.6s при real 5m7s — цикл крутится в дочернем процессе
```

## Минимальное репро

Самодостаточно (BigInt из пакета), `while`-условие — цепочка вызовов методов:

```nova
module bigint.repro_while_test

import bigint.{BigInt}

fn count_up(num BigInt, den BigInt) -> int {
    mut p10 = BigInt.one()
    mut j = 0
    while num.times(p10).compare(den) < 0 {   // <- виснет
        p10 = p10.times((10).to_bigint())
        j += 1
    }
    -j
}

test "repro: while with method-chain condition" {
    assert(count_up((1).to_bigint(), (3).to_bigint()) == -1)
}
```

Прогон: `TIMEOUT` (64s). Контроль — тот же цикл через `loop`/`break` (условие в
теле, `if … break`): **PASS**.

## Корневая причина (дефект codegen компилятора)

Условие `while` с цепочкой методов эмитится в C как временная переменная,
**вычисляемая ОДИН раз до цикла и не пересчитываемая на итерациях**:

```c
/* SRC: while num.times(p10).compare(den) < 0 { */
NovaValue_BigInt _nv_tmp_1418 = Nova_BigInt_method_times(&(num), p10);
while ((Nova_BigInt_method_compare(&_nv_tmp_1418, den) < ((nova_int)0LL))) {
    p10 = Nova_BigInt_method_times(&(p10), Nova_nova_int_method_to_bigint(((nova_int)10LL)));
    j = nova_int_checked_add(j, ((nova_int)1LL));
}
```

Для `1/3`: `num*1 < den` остаётся истинным всегда → `j` растёт без предела →
бесконечный цикл. Простые условия `while i >= 1` (константа/переменная слева)
работают корректно — дефект именно в цепочке вызовов внутри условия.

**Вердикт по корню:** дефект кодогенератора (неправильный hoisting временной
переменной условия `while`), не ошибка в коде тестов. По правилам BRIEF: компилятор
не чинить, молча не обходить → дефект задокументирован, обход применён и отражён
здесь: в `bigrat_test.nv` (и `bisect*-scratch`, удалённых) `while`-цепочки заменены
на эквивалент `loop { if … break }`.

## Что сделано в Ф.5

- `src/bigrat_test.nv`: property-тесты (splitmix64 seed'ы 7/13/17/31, N=300,
  против int-эталона), нормальная форма после каждой op (+ гармоническая цепочка
  `1..50`), канонические вектора (`1/3+1/6==1/2`, `H_10 == 7381/2520`),
  BigDecimal roundtrip `1.25 ↔ 5/4`, эталон-кейсы деления: ручное округление
  (HalfEven) точного BigRat-частного vs `BigDecimal @div(MathContext)` (8 векторов)
  и vs `BigFloat @div` через `to_bigrat` обеих сторон (9 векторов).
- `src/bigrat_slow.nv` (slow-lane, не в дефолтном прогоне): `100!/99!==100`,
  `2^200/2^199==2`, телескопический продукт `i/(i+1)` i=1..1000 == `1/1001`,
  H_100, `BigFloat @to_bigrat` на `1/2^300`.
- По ходу исправлено: `E_READONLY_COERCE` (в `rat_dec_exp`/`rat_bit_exp` — старт
  с `BigInt.one()`, а не `mut acc = num`); `W_NON_COMPOUND_ASSIGN` (`j += 1`);
  CC-FAIL `Nova_BigFloat` — файлы, импортирующие `bigint.bigrat`, обязаны
  импортировать `bigint.bigfloat.{BigFloat}` (тип нужен кодогену из-за
  `BigFloat @to_bigrat` в `bigrat.nv`).

## Вердикты гейтов (Ф.5)

```
$ nova test src
PASS: 9  FAIL: 0  SKIP: 4

$ nova test src/bigrat_slow.nv --include-slow
PASS: 1  FAIL: 0

$ nova check src --strict-effects
PASS: 13  FAIL: 0  WARN: 38

$ nova lint src/bigrat.nv src/bigrat_test.nv src/bigrat_slow.nv
lint: 3 file(s), 0 finding(s)
```

Ф.5 **ЗЕЛЁНАЯ** (коммит фазы будет). Ф.6: англ. doc-комменты, README, удаление
`repro_*.nv`.


---

## Разворот обхода №281 (интегратор, 2026-08-02)

Дефект `[M-while-condition-chain-not-recomputed]` (№281) ЗАКРЫТ в компиляторе
окном s1b (nova main `7bf3370f1`): условие `while` с цепочкой вызовов
вычислялось один раз ДО цикла — побочные temp-стейтменты замерзали, давая
тихий вечный цикл. Фикс зеркалит корректный путь `while let`; фикстура
`spec_tests/conformance/m281_while_chain_condition_recompute_pos.nv`.

Санкционированный обход `loop { if cond { break } ... }` в
`src/bigrat_test.nv` (две площадки: `rat_dec_exp`, `rat_bit_exp`) РАЗВЁРНУТ
обратно в каноническую форму `while <цепочка> { ... }`. Прогон на
исправленном компиляторе: `nova test src` — PASS: 9 FAIL: 0 SKIP: 4,
`nova lint src` — 0 findings. Зависания нет.

`loop` в `src/bigfloat.nv` (`@sqrt`, ньютоновские итерации) — НЕ обход:
законная форма с выходом по середине тела, оставлена как есть.
