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
| Ф.0 Пины | **ЗЕЛЁНЫЙ** (блокер закрыт, реестр nova №274) | 0bd3ac5 (скаффолд) |
| Ф.1 gcd в bigint.nv | ожидает | — |
| Ф.2 Ядро bigrat | ожидает | — |
| Ф.3 Арифметика | ожидает | — |
| Ф.4 Строки и связка семьи | ожидает | — |
| Ф.5 Тесты + эталон-кейсы | ожидает | — |
| Ф.6 Закрытие | ожидает | — |

Базовая линия (до работы): `nova check src --strict-effects` = 7/0/24,
`nova test src` = 5/0/2 (SKIP 2 — bigdecimal/bigfloat без тел).

## Текущее состояние (после снятия блокера)

- Компилятор пересобран (фикс №274: auto-by-ref в операторных стабах), хеш
  `017a24989c1c2c04862e32bc11bab341e2dfe9cebf547c0cbc20e1459063a006`.
- `nova check src --strict-effects`: **PASS 11 / FAIL 0 / WARN 24** — статика зелёная.
- `nova test src`: **PASS 8 / FAIL 0 / SKIP 3** — Ф.0 пины (а/а2/б/в/г) зелёные.
- СТОП-репорт: `REPORT-p240.md` (история; блокер закрыт — см. резюме в конце файла).
- Дальше: Ф.1 (gcd) → Ф.6 по плану.
