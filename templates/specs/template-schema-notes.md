# Template Schema — Design Notes

## Статус

Proposed (Phase 2, batch 1)

## Дата

2026-03-25

## Назначение

Пояснения к `template.schema.yaml`: обоснования ключевых решений,
trade-offs, и что остаётся открытым.

---

## Решение 1: Версионирование через поле `version`

**Выбор:** `template_id` стабилен, `version` — инкрементируемое целое число.

**Альтернатива:** Новый `template_id` для каждой версии
(например `orbit_micro_v1`, `orbit_micro_v2`).

**Почему выбрано `version` поле:**
- `template_id` остаётся стабильной точкой ссылки (test suites, runs, scoring —
  все ссылаются на template_id, а не на конкретную версию);
- при анализе истории тестирования можно видеть эволюцию одного template
  через версии, а не собирать по разрозненным ID;
- проще для навигации: один файл `orbit_micro_default.template.yaml`,
  а не множество файлов.

**Следствия:**
- При изменении template spec файл обновляется in-place, `version` инкрементируется.
- Предыдущие версии сохраняются в git history.

**Открытый вопрос — versioning policy для runs:**
Как runs ссылаются на версию template — через сохранённый `template_version`,
через snapshot конфигурации, или достаточно git history — пока не определено.
Это proposed-решение, которое будет зафиксировано при проектировании run format
(Phase 4–5). Текущий domain-model.md не фиксирует этот механизм.

**Статус:** proposed. Требует подтверждения при первом реальном use case
версионирования (Phase 6).

---

## Решение 2: Нет template-level defaults для aspect_ratio и duration

**Выбор:** Template определяет только `supported_*` списки.
Пользователь всегда явно выбирает `aspect_ratio` и `duration`.

**Почему:**
- Один template может хорошо работать в разных aspect ratios —
  нет объективного «default»;
- явный выбор пользователя = меньше неожиданностей;
- в V1 нет UI, где default сэкономил бы пользователю время;
- при необходимости defaults можно добавить позже без ломающих изменений.

**Статус:** proposed.

---

## Решение 3: `template_id` формат

**Формат:** `{family_id}_{variant}`.

**Примеры:**
- `orbit_micro_default` — основной вариант orbit_micro
- `dolly_in_clean_slow` — вариант dolly_in_clean с акцентом на медленное движение

**Почему:**
- Читаемо для человека;
- family_id встроен в template_id → можно определить family без чтения файла;
- `variant` даёт пространство для нескольких templates внутри одного family.

**Примечание:** В V1 скорее всего будет по одному template на family
(все с variant = `default`). Формат оставляет пространство для расширения.

---

## Решение 4: `precompose_rules` — placeholder-структура

Детали precompose определяются в Phase 3. В Phase 2 фиксируется только:
- что `precompose_rules` — обязательная секция template spec;
- что `aspect_ratio_strategy: precompose` — единственный вариант в V1;
- что есть поля для `safe_area_margin`, `background_fill` и `notes`.

Точные значения, форматы и правила — Phase 3.

---

## Решение 5: `scoring_hints` вместо полной scoring rubric

Template spec содержит только `scoring_hints` — подсказки для оценки.
Полная scoring rubric определяется на уровне `test_suite` (Phase 5),
в соответствии с ADR-0003 (И-4: score → run, не score → template).

`scoring_hints` позволяют зафиксировать template-specific акценты
(например: «у orbit_micro особенно важна стабильность краёв блюда при вращении»),
не дублируя систему scoring.

---

## Что осознанно НЕ включено в schema V1

| Исключённое | Почему |
|---|---|
| `end_frame_rules` | V1 = start frame only (ADR-0002, п.9) |
| `audio_config` | V1 = sound=false (ADR-0002, п.9) |
| `multi_shot_config` | V1 = single-shot only (ADR-0002, п.9) |
| `ab_test_config` | V1 = prompt_pack 1:1 (ADR-0002, п.13) |
| `auto_angle_detection` | V1 избегает тяжёлого CV (ADR-0002, exclusions) |
| `priority` / `weight` | Нет selection layer в V1 |
| `owner` / `assignee` | Нет workflow management в V1 |

---

## Открытые вопросы

| # | Вопрос | Когда решается |
|---|--------|----------------|
| 1 | Точный формат `safe_area_margin` (px, %, ratio?) | Phase 3 |
| 2 | Нужен ли `background_fill` в template или он определяется только в precompose pipeline? | Phase 3 |
| 3 | Как runs ссылаются на конкретную версию template: хранят `template_version` или snapshot полной конфигурации? | Phase 4–5 |
