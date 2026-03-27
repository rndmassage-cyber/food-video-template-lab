# ADR-0006: First Real Run Validation Policy

## Status

Proposed

## Date

2026-03-26

## Context

После Phase 4 adapter spec частично исправлен под реальную Kie AI документацию.
Однако ряд значений по-прежнему помечен как proposed и требует real-API верификации:
- Точный model identifier для Kling 3.0
- `duration` format (строка vs число)
- `aspect_ratio` format
- Поля внутри `resultJson`
- Failure state string value

Данный ADR определяет политику верификации через один ручной smoke run.

## Decision

### Д-1. Минимальный smoke test

- Template: `orbit_micro_default`, tc_01 (1:1, 5s, three_quarter)
- Mode: **polling** (без callBackUrl)
- Цель: верификация assumptions, не quality scoring

### Д-2. Assumptions для верификации

| ID | Assumption | Метод |
|----|-----------|-------|
| A1 | `model` поле принимает proposed value | API не вернул 4xx/invalid model |
| A2 | `input.duration = "5"` (строка) | Нет ошибки валидации |
| A3 | `input.aspect_ratio = "1:1"` | Нет ошибки валидации |
| A4 | Task states = `waiting`/`queuing`/`generating`/`success` | Наблюдение в poll responses |
| A5 | Terminal shape = `{ taskId, state, resultJson, ... }`, видео в `resultJson` | Field-by-field наблюдение |

### Д-3. Критерии verdict

**confirmed:** Нет расхождений spec и observation.
**partially_confirmed:** Структура верна, детали отличаются.
**contradicted:** Реальное поведение явно противоречит assumption.
**unclear:** Недостаточно данных (run упал до проверки assumption).

### Д-4. Что делать с findings

**confirmed:** `Proposed → Verified (smoke test, {дата})`

**partially_confirmed:** Добавить observation note, operator решает.

**contradicted:**
1. Обновить spec реальным значением
2. `Corrected (smoke test, {дата}): was "{old}", actual "{new}"`
3. Обновить связанные файлы

**unclear:** Записать, планировать targeted verification, не блокировать.

### Д-5. Что обязательно обновить после smoke test

| Файл | Что |
|------|-----|
| `kling-request-format.md` | model identifier, image field name |
| `kling-response-format.md` | resultJson structure |
| `task-status-mapping.md` | failure state string value |
| `ADR-0005` | Закрыть верифицированные открытые вопросы |

### Д-6. Что НЕ верифицировать в smoke test

| Аспект | Почему |
|--------|--------|
| Качество генерации | Не цель |
| Polling intervals оптимальность | Phase 6 |
| Несколько aspect ratios | 1:1 достаточно |
| callBackUrl behaviour | Polling достаточен для V1 |
| Rate limits | Один run не триггерит |

### Д-7. Image hosting policy

Допустим любой временный URL. Метод фиксируется в run record.

### Д-8. Артефакты

| Артефакт | Путь |
|---------|------|
| Run record | `runs/orbit_micro_default/run_orbit_micro_default_smoke_001.run-record.yaml` |
| Smoke checklist | `templates/runs/examples/orbit_micro_default.smoke-run-checklist.md` |
| Provider observation | `templates/adapters/examples/orbit_micro_default.provider-task-observation.md` |

## Consequences

**Получаем:** spec = реальный API; фундамент для Phase 5 без непроверенных assumptions.
**Платим:** нужно решить upstream image hosting до smoke run; operator judgement при partially_confirmed.

## Findings (заполнить после smoke test)

| ID | Verdict | Notes |
|----|---------|-------|
| A1 | | |
| A2 | | |
| A3 | | |
| A4 | | |
| A5 | | |

## Ссылки

- [ADR-0005](ADR-0005-kling-kie-adapter-contract.md)
- [Smoke Test Runbook](../runbooks/first-smoke-test.md)
- [Smoke Run Checklist](../../templates/runs/examples/orbit_micro_default.smoke-run-checklist.md)
- [Provider Task Observation](../../templates/adapters/examples/orbit_micro_default.provider-task-observation.md)
