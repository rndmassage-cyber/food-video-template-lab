# ADR-0004: Precompose First Frame Policy

## Status

Proposed

## Date

2026-03-25

## Context

ADR-0002 (п. 12) зафиксировал принцип: aspect ratio обеспечивается через precompose start frame,
а не через внутреннюю адаптацию Kling. Domain model определил сущность `precompose_input`
с атрибутами, но без формализованных правил.

Phase 2 оставила открытыми вопросы:
- Точный формат safe_area_margin (template-schema-notes, Q#1)
- background_fill: в template или в precompose pipeline? (template-schema-notes, Q#2)
- Формат source_image_ref (domain-model, open Q#1)
- Политика кэширования precompose_input (domain-model, open Q#2)
- Формат start_frame_ref в run records (run-record-format, open Q#1)

Данный ADR формализует policy-level решения для precompose layer в V1.

### Почему precompose, а не внутренняя адаптация Kling

Kling API принимает параметр `aspect_ratio` при генерации, но его внутреннее поведение
при несовпадении aspect ratio входного изображения и целевого — непредсказуемо:
возможны автоматический crop, scaling с искажениями, добавление артефактов по краям.

Это противоречит главному приоритету проекта: **стабильность, предсказуемость, повторяемость**.

Precompose даёт полный контроль над start frame до отправки в Kling.

## Decision

### П-1. Precompose обязателен

Каждый V1 run использует precomposed start frame. Bypass невозможен.
Даже если source image уже имеет целевой aspect ratio, оно проходит через precompose
(как минимум — проверка safe area и resolution).

### П-2. Canvas-based, no crop

Source image **никогда не кропается**. Precompose работает по принципу:

1. Создаётся canvas целевого размера.
2. Source image размещается на canvas с учётом safe area margin.
3. Пустые области заполняются background fill.

Это гарантирует, что блюдо всегда видно целиком.

### П-3. Safe area margin — percentage

`safe_area_margin` задаётся как **процент от меньшей стороны canvas**.

Пример: при canvas 1080×1920 (9:16) и margin 10% →
минимальный отступ = 1080 × 0.10 = 108 px с каждой стороны.

Значение margin определяется в precompose spec каждого template.
Рекомендуемый диапазон V1: 8%–15%.

**Почему процент, а не пиксели:**
- Масштабируется при изменении resolution targets.
- Единообразен между aspect ratios.
- Достаточно точен для V1 (не нужна попиксельная точность).

### П-4. Background fill

Background fill определяется в **precompose spec**, а не в template spec.

Template spec содержит только `background_fill` hint (source_bg / solid_color).
Конкретные параметры (цвет, метод сэмплирования) — в precompose spec.

**V1 стратегии:**

| Стратегия | Описание | Когда использовать |
|-----------|----------|-------------------|
| `source_bg` | Цвет фона определяется из source image (edge sampling) | Когда фон однотонный и чистый (основной случай V1) |
| `solid_color` | Явно заданный цвет | Когда edge sampling ненадёжен или нужен конкретный цвет |

**Примечание:** V1 работает только с однотонным фоном (ADR-0002, п.4),
поэтому `source_bg` через edge sampling — основной и достаточный метод.

### П-5. Resolution targets

Фиксированные resolution targets для V1:

| Aspect ratio | Resolution | Примечание |
|-------------|-----------|-----------|
| 1:1 | 1080 × 1080 | |
| 9:16 | 1080 × 1920 | |
| 16:9 | 1920 × 1080 | |

**Статус:** proposed. Подлежит валидации на совместимость с Kling 3.0 API
(некоторые модели могут требовать конкретные resolution). При несовместимости —
amendment к этому ADR.

### П-6. Source image ref format

`source_image_ref` в V1 — **относительный путь от корня проекта**.

Формат: `fixtures/raw-inputs/{image_name}.{ext}`

Пример: `fixtures/raw-inputs/plate_pasta_three_quarter.png`

**Почему путь, а не ID:**
- В V1 нет image registry и не планируется.
- Путь однозначен и проверяем (файл существует или нет).
- При необходимости миграция на ID возможна без ломающих изменений.

### П-7. Start frame ref format

`start_frame_ref` в run records — **относительный путь от корня проекта**.

Формат: `fixtures/prepped-images/{source_name}_{aspect_shortcode}.png`

Где `aspect_shortcode`: `1x1`, `9x16`, `16x9`.

Пример: `fixtures/prepped-images/plate_pasta_three_quarter_1x1.png`

### П-8. Кэширование precompose outputs

Precompose output **можно переиспользовать** между runs, если совпадают:
- `source_image_ref` (тот же файл)
- `template_id` (те же precompose rules)
- `target_aspect_ratio`

В V1 кэширование — ручное (оператор проверяет, есть ли уже подготовленный frame).
Автоматическое кэширование — за рамками V1.

**Когда НЕ переиспользовать:**
- Если template.precompose_rules изменились (новая версия template).
- Если source image обновлён (даже если имя файла то же).

### П-9. Precompose spec — отдельный файл

Детальная precompose-спецификация для каждого template хранится в отдельном файле:

`templates/precompose/{template_id}.precompose.yaml`

Связь template → precompose spec = **1:1**, discoverable by convention (совпадение template_id).

Template spec содержит `precompose_rules` как summary + hints.
Precompose spec содержит полные параметры: per-aspect-ratio canvas specs,
margin overrides, background fill details.

## Consequences

**Что получаем:**
- Полный контроль над start frame, не зависящий от поведения Kling.
- Формализованные ответы на 5 открытых вопросов из Phase 2.
- Единообразный формат ссылок на изображения (source и precomposed).
- Возможность кэширования подготовленных frames между runs.
- Отдельный precompose spec per template — чистое разделение ответственности.

**Чем платим:**
- Обязательный шаг precompose добавляет latency в workflow.
- Новый тип файла (.precompose.yaml) — ещё один артефакт для поддержки.
- Resolution targets фиксированы — если Kling потребует другие, нужен amendment.

## Условия пересмотра

- Kling API потребует другие resolution или изменит поведение с aspect ratio.
- Появятся source images с неоднотонным фоном (выход за V1 scope).
- Потребуется динамическая подгонка margin на основе содержимого изображения (CV-анализ).

## Ссылки

- [ADR-0002 — Границы и ограничения V1](ADR-0002-v1-scope.md)
- [ADR-0003 — Template-first architecture](ADR-0003-template-first-architecture.md)
- [Domain Model](../domain-model.md)
- [Template Schema Notes](../../templates/specs/template-schema-notes.md)
