# L08 · трасування і ціна · PayPilot

Здача лабораторної L08: звіт «Observability & Cost» за трьома слайдами, набір діалогових кейсів із бюджетами, звіти ранера й таблиця міграції моделі.

| Що | Файл |
|---|---|
| Звіт, заповнений по PayPilot (три слайди) | [observability-cost-report.md](observability-cost-report.md) |
| Порожній шаблон звіту | [observability-cost-report.template.md](observability-cost-report.template.md) |
| Набір діалогів з бюджетними кейсами | [sets/l05_dialogs.jsonl](sets/l05_dialogs.jsonl) |
| Звіти ранера для `--profile clean` і `--profile lesson-08` | `reports/` |
| Сирі результати сценарію B4 (кіт L08) | [b4.json](b4.json) |
| Знімки Langfuse до слайда 1 | `slide1-*.png` |

## L08

### Команди прогону

Передумови: локальний стенд `~/paypilot/paypilot-stand` на `http://localhost:8000`, `LLM_PROVIDER=openrouter` і ключ у `.env` стенду, Langfuse на `http://localhost:3000` (піднімається разом зі стендом). Ранер скидає базу стенду перед кожним кейсом, тому на спільному стенді його не запускають.

Прогін робить ранер L05 з `~/paypilot/l05`: він читає набір із `l05/sets/`, а копія в `l08/sets/` має збігатися з ним за `set_hash`.

```bash
cd ~/paypilot/l05

# 1. сухий прогін: хеш і покриття набору, нічого не викликає
docker compose run --rm eval --set l05_dialogs --dry-run

# 2. повний набір (daily і release) на справній версії і на lesson-08
docker compose run --rm eval --set l05_dialogs --profile clean --gate all
docker compose run --rm eval --set l05_dialogs --profile lesson-08 --gate all

# 3. повернути стенд на clean: ранер лишає останній профіль, поріг згортки й час
curl -s -X PUT localhost:8000/api/_test/profile -H 'Content-Type: application/json' -d '{"profile":"clean"}'
curl -s -X PUT localhost:8000/api/_test/summarize_after -H 'Content-Type: application/json' -d '{"steps": 8}'
curl -s -X PUT localhost:8000/api/_test/clock -H 'Content-Type: application/json' -d '{"now":null}'
```

Корисні варіації: `--gate daily` (типово) запускає лише детерміновані кейси `*-IDENT`; `--only C07-BUDGET,S2-D14-BUDGET` — лише названі кейси; `--brief` — по рядку на кейс. Код повернення 0, якщо пройшли всі кейси, і 1, якщо впав хоч один. Звіти ранер пише в `l05/reports/<набір>-<профіль>-<час>.json`; їх створює root у Docker, тому до `l08/reports/` їх копіюють, а не переносять.

Сценарій B4 (міграція моделі) іде з кіта L08 (`github.com/sergeytkachenko/paypilot-l08-eval`), не з ранера L05:

```bash
cd <тека кіта>
docker compose run --rm eval --scenario B4 --table --json b4.json   # близько 10–15 хвилин, ~$0,1
cp b4.json ~/paypilot/l08/b4.json
```

### Бюджетні кейси

Файл [sets/l05_dialogs.jsonl](sets/l05_dialogs.jsonl): поруч із діалогами CONV-002/003/004 і MEM-001-* (їхні рядки не змінені) додано 8 кейсів з `assertion: budget`, `oracle: human`. Baseline: `anthropic/claude-haiku-5.5` через OpenRouter, профіль `clean`, 5 прогонів на кейс, 2026-11-05, прайс стенду від 2026-10-23. Усі кейси зафіксовані на `summarize_after: 8`. Пороги прив'язані до цієї моделі: на іншій їх треба перезняти.

| Кейс | Gate · runs | Поріг | Що охороняє |
|---|---|---|---|
| `C07-BUDGET-IDENT` | daily · 1 | `identical_tool_calls_max` 1 | повтори викликів у C-07 (7 кроків); сторож |
| `C07-BUDGET` | release · 5 | `tokens_total` 57 450 | D15: replay результатів інструментів на діалозі з 7 кроків і викликами інструментів |
| `CONV003-BUDGET-IDENT` | daily · 1 | `identical_tool_calls_max` 1 | повтори викликів у діалозі з 10 кроків; сторож |
| `CONV003-BUDGET` | release · 5 | `tokens_total` 84 010 | груба перевитрата; D15 не ловить |
| `CONV004-BUDGET-IDENT` | daily · 1 | `identical_tool_calls_max` 1 | повтори викликів у діалозі з 5 кроків; сторож |
| `CONV004-BUDGET` | release · 5 | `tokens_total` 32 290 | груба перевитрата; D15 не ловить |
| `S2-D14-BUDGET-IDENT` | daily · 1 | `identical_tool_calls_max` 1 | D14: клієнт CUS-0003, порожній `get_transactions`; один крок, бо ранер рахує повтори лише в останньому кроці |
| `S2-D14-BUDGET` | release · 5 | `tokens_total` 18 680, `latency_ms` 9 020 | D14: токени і час діалогу |

Поріг `tokens_total` дорівнює медіані baseline на `clean` плюс 25%; калібрування з трьома порогами (надто вільний, надто тісний, власний) і медіанами обох профілів — наприкінці [звіту](observability-cost-report.md).

**Обмеження ранера L05, яке треба знати перед читанням звітів.** `_budget` додає токени всіх `runs` кейса й порівнює суму з `tokens_total`, а не медіану прогонів, тому чотири release-кейси червоніють на будь-якому профілі, навіть коли середнє на прогін у межах порогу. Працюючий гейт на цьому ранері — daily-кейси `*-IDENT`. Ранер не змінювали; кейси записано за шаблоном (release, `runs: 5`, вердикт за медіаною) для ранера, який рахує медіану. Деталі — у розділі «Прогін набору на lesson-08» звіту.

### Прогони

Набір `l05_dialogs`, `set_hash 3006e3f637bc`, `--gate all`, 2026-11-05, модель `anthropic/claude-haiku-5.5`. Звіти лежать у [reports/](reports/): `l05_dialogs-clean-20261105T192157.json` (clean, 1 111 с) і `l05_dialogs-lesson-08-20261105T194029.json` (lesson-08, 1 233 с). Хеш у них однаковий, тож звіти можна порівнювати. Стенд після прогонів повернуто на `clean`.

| Профіль | Пройшло | Не пройшли |
|---|---|---|
| clean | 7 з 13 | CONV-002, CONV-003, `C07-BUDGET`, `CONV003-BUDGET`, `CONV004-BUDGET`, `S2-D14-BUDGET` |
| lesson-08 | 8 з 13 | `C07-BUDGET`, `CONV003-BUDGET`, `CONV004-BUDGET`, `S2-D14-BUDGET-IDENT`, `S2-D14-BUDGET` |

Бюджетні кейси в звітах видно окремими рядками (`orchestration budget`). Токени release-кейсів ранер додає по 5 прогонах, тому поруч — середнє на прогін (сума ÷ 5), яке й порівнюється з порогом за шаблоном:

| Кейс | Поріг | clean: сума → середнє | lesson-08: сума → середнє | Вердикт за середнім |
|---|---|---|---|---|
| `C07-BUDGET` | 57 450 | 231 052 → 46 210 | 295 115 → 59 023 | D15 ловиться (clean нижче, lesson-08 вище порогу) |
| `CONV003-BUDGET` | 84 010 | 357 865 → 71 573 | 403 450 → 80 690 | обидва нижче; D15 не ловить |
| `CONV004-BUDGET` | 32 290 | 116 395 → 23 279 | 119 018 → 23 804 | обидва нижче; D15 не ловить |
| `S2-D14-BUDGET` | 18 680 токенів, 9 020 мс | 72 087 → 14 417 | 192 360 → 38 472; час 28 434 мс | D14 ловиться за токенами і за часом |

Daily-кейси: `S2-D14-BUDGET-IDENT` — PASS на clean (`identical calls 1`) і **FAIL** на lesson-08 (`identical tool calls 8 > 1`), тобто D14 ловиться детерміновано; решта `*-IDENT` — PASS на обох профілях.

Два застереження. (1) Під ранером L05 чотири release-кейси червоніють на будь-якому профілі через суму прогонів (див. «Бюджетні кейси»); ранер не змінювали, `runs: 5` лишено за шаблоном. (2) CONV-002 і CONV-003 на clean пройшли по 4 з 5: `not_regex` D07 спрацював на слушну згадку другого списання TX-0402 (`duplicate_charge`, 60 днів); це інваріанти з наявного набору, їх не змінювали. Деталі й таблиця — у розділі «Прогін набору на lesson-08» [звіту](observability-cost-report.md).
