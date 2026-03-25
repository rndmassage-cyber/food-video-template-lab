# Template Registry Format — food-video-template-lab

## Статус

Proposed (Phase 2, batch 3)

## Дата

2026-03-25

## Назначение

Определяет формат единого индексного файла (registry) для всех templates проекта.

Registry — централизованный обзор: какие templates существуют, в каком статусе,
с какими агрегатными метриками. Позволяет видеть картину целиком
без необходимости открывать каждый template spec.

Canonical sources: ADR-0003 (template = central entity), domain-model.md (статусная модель).

---

## Принципы

1. **Единый файл.** Registry — один YAML-файл, не набор файлов.
2. **Индекс, не дублирование.** Ключевые поля (status, version, supported_*) дублируются
   из template specs для быстрого обзора. Source of truth — template spec.
3. **Агрегатные метрики.** Registry добавляет информацию, которой нет в template specs:
   total_runs, pass_rate, last_tested. Это главная добавленная ценность.
4. **Ручное обновление в V1.** Синхронизация registry с template specs и run results
   выполняется вручную. Автоматизация — за рамками V1.

---

## Расположение файла

Файл: `templates/registry.yaml`

---

## Структура файла

### Верхнеуровневые поля

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `registry_version` | integer | yes | Версия формата registry (начинается с 1) |
| `updated_at` | string | yes | ISO 8601 дата последнего обновления registry |
| `templates` | list[object] | yes | Список записей о templates |

### Структура записи template

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `template_id` | string | yes | Ссылка на template spec |
| `family_id` | string | yes | Семейство шаблона |
| `version` | integer | yes | Текущая версия template spec |
| `status` | string | yes | Текущий статус lifecycle |
| `supported_angles` | list[string] | yes | Поддерживаемые ракурсы |
| `supported_aspect_ratios` | list[string] | yes | Поддерживаемые соотношения сторон |
| `supported_durations` | list[number] | yes | Подтверждённые длительности |
| `prompt_pack_id` | string | yes | Ссылка на связанный prompt pack |
| `test_suite_ids` | list[string] | no | Ссылки на связанные test suites |
| `stats` | object | no | Агрегатные метрики по runs (см. ниже) |
| `notes` | string | no | Свободные заметки |

### Структура stats

| Поле | Тип | Description |
|------|-----|-------------|
| `total_runs` | integer | Общее количество выполненных runs |
| `scored_runs` | integer | Количество runs с выставленной оценкой |
| `pass_rate` | number (0.0–1.0) или null | Доля runs с verdict=pass среди scored |
| `last_tested` | string (ISO 8601) или null | Дата последнего run |

---

## Связь с другими артефактами

```
registry.yaml (индекс)
  └── template_id → templates/specs/{template_id}.template.yaml (source of truth)
       ├── prompt_pack_id → templates/prompt-packs/{pp_id}.prompt-pack.yaml
       ├── test_suite_ids → templates/test-suites/{ts_id}.test-suite.yaml
       └── runs → runs/{template_id}/*.run-record.yaml
```

Registry **ссылается на** template specs, но не заменяет их.
При расхождении source of truth — template spec.

---

## Что registry НЕ содержит

| Исключённое | Почему |
|---|---|
| motion_intent, precompose_rules | Слишком детально для индекса — см. template spec |
| Prompt-тексты | Это prompt_pack |
| Отдельные run results | Это run records |
| Scoring criteria | Это test_suite + scoring rubric |

---

## Рекомендации по ведению registry

1. **Обновлять при изменении статуса.** Когда template меняет status — обновить registry.
2. **Обновлять stats после scoring.** После оценки пакета runs — обновить stats.
3. **Синхронизировать version.** Если template spec обновился — обновить version в registry.
4. **Не ленитесь обновлять.** В V1 registry — ручной инструмент. Его ценность
   пропорциональна актуальности.

---

## Открытые вопросы

| # | Вопрос | Когда решается |
|---|--------|----------------|
| 1 | Нужна ли автоматическая генерация/валидация registry из template specs? | Phase 7 |
| 2 | Нужны ли дополнительные агрегатные метрики (per-angle pass rate, per-duration stats)? | Phase 7 |
| 3 | Нужен ли history log изменений статусов в registry? | Phase 7 |
