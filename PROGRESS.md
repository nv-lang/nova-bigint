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
| Ф.1 gcd в bigint.nv | **ЗЕЛЁНЫЙ** | 9ee5aef |
| Ф.2 Ядро bigrat | **ЗЕЛЁНЫЙ** | 0df67a5 |
| Ф.3 Арифметика | **ЗЕЛЁНЫЙ** | 68ef71a |
| Ф.4 Строки и связка семьи | **ЗЕЛЁНЫЙ** (блокер закрыт, фиксы компилятора №279/#280) | 451a096, c908b7c |
| Ф.5 Тесты + эталон-кейсы | **ЗЕЛЁНЫЙ** | ce1cbc8 |
| Ф.6 Закрытие | ожидает | — |

Базовая линия (до работы): `nova check src --strict-effects` = 7/0/24,
`nova test src` = 5/0/2 (SKIP 2 — bigdecimal/bigfloat без тел).

## Текущее состояние (после Ф.5)

- Компилятор пересобран (фиксы №274, №279/#280), хеш
  `df16073e7900631d0459014ac59c0b0d56f8b12665b8ec87efa9a60c7dd1729c`.
- `nova check src --strict-effects`: **PASS 13 / FAIL 0 / WARN 38** — статика зелёная.
- `nova test src`: **PASS 9 / FAIL 0 / SKIP 4** (SKIP — модули без test-блоков).
- `nova lint` (bigrat.nv, bigrat_test.nv, bigrat_slow.nv): 0 findings.
- Ф.5 закрыта: тесты (property против int-эталона, нормальная форма, канонические
  вектора, BigDecimal roundtrip, эталон-кейсы `@div` HalfEven против ручного
  округления точного BigRat-частного) в `bigrat_test.nv`; большие вектора — в
  `bigrat_slow.nv` (`--include-slow`, PASS 1/0).
- Дефект codegen, обнаруженный на Ф.5 (while-условие с цепочкой методов не
  пересчитывается на итерациях → бесконечный цикл), задокументирован в
  `REPORT-p240.md`; обход — `loop`/`break`.
- Дальше: Ф.6 — закрытие (англ. doc-комменты, README, удаление repro_*.nv).
