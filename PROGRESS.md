# PROGRESS — план 240 (BigRat V1, nova-bigint)

Исполнитель: opencode/big-pickle, ветка `p240`.
Рабочая директория: `d:/Sources/nv-lang/nova-bigint-p240`.
Компилятор: `d:/Sources/nv-lang/nova/nova-cli/target/release/nova.exe`.
Окружение (env для гейтов): `NOVA_STD_PATH=d:/Sources/nv-lang/nova/std`,
`NOVA_RT_DIR=d:/Sources/nv-lang/nova/compiler-codegen/nova_rt`,
`NOVA_GC_LIB_DIR=d:/Sources/nv-lang/nova/compiler-codegen/vcpkg_installed/x64-windows-static/lib`,
`NOVA_CG_INCLUDE=d:/Sources/nv-lang/nova/compiler-codegen`.

| Фаза | Статус | Коммит |
|---|---|---|
| Ф.0 Пины | **СТОП** (дефект codegen: операторный desugar на value-struct >16Б) | — |
| Ф.1 gcd в bigint.nv | ожидает | — |
| Ф.2 Ядро bigrat | ожидает | — |
| Ф.3 Арифметика | ожидает | — |
| Ф.4 Строки и связка семьи | ожидает | — |
| Ф.5 Тесты + эталон-кейсы | ожидает | — |
| Ф.6 Закрытие | ожидает | — |

Базовая линия (до работы): `nova check src --strict-effects` = 7/0/24,
`nova test src` = 5/0/2 (SKIP 2 — bigdecimal/bigfloat без тел).

## Текущее состояние (после СТОП)

- `nova check src --strict-effects`: PASS 10 / FAIL 0 / WARN 24 — статика зелёная
  (включая новые `bigrat.nv`, `bigrat_test.nv`, `repro_test.nv`, `repro_direct_test.nv`).
- `nova test src`: **CC-FAIL** на `src/bigrat_test.nv` (и на репро) — дефект компилятора.
- СТОП-репорт: `REPORT-p240.md` (минимальное репро `src/repro_test.nv`).

### Суть дефекта

Plan 172.14: `ro`-параметр value-struct'а C-размером >16 Б эмитится как `T*`. BigRat
(32 Б) впервые в пакете пробивает порог (BigInt = 16 Б — нет). Прямые вызовы
`a.plus(b)`/`a.equal(b)` работают (проверено), но операторный desugar `+ - *` и `==`
(Plan 175 / 172.4 обёртки `nova_vr_binop_*`/`nova_vr_ueq_*`) передают второй операнд
по значению при объявленном `T*` → CC-FAIL «passing 'NovaValue_BigRat' to parameter of
incompatible type 'NovaValue_BigRat *'». Компилятор не чинить, обход не делать —
ожидание решения владельца.
