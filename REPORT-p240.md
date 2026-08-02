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
