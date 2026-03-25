# Precompose Spec Format — food-video-template-lab

## Статус

Proposed (Phase 3, batch 1)

## Дата

2026-03-25

## Назначение

Определяет формат YAML-файлов precompose spec для V1.

Precompose spec — детальная спецификация подготовки start frame для конкретного template.
Описывает canvas-параметры per aspect_ratio, background fill, safe area, placement rules.

Canonical sources: ADR-0004 (precompose policy), ADR-0003 (template-first architecture),
domain-model.md (сущность precompose_input).

---

## Принципы

1. **1:1 с template.** Каждый template имеет ровно один precompose spec.
   Discoverable by convention: `{template_id}.precompose.yaml`.
2. **Декларативный.** Описывает *что* нужно получить, а не *как* это реализовать.
   Конкретная implementation (скрипт, ручной Photoshop, pipeline) — вне формата.
3. **Per-aspect-ratio.** Основные параметры задаются для каждого aspect_ratio отдельно,
   потому что canvas dimensions и placement логика различаются.
4. **Совместим с template.** Precompose spec не дублирует template spec.
   Template содержит summary (`precompose_rules`), precompose spec — детали.

---

## Расположение файлов

- Формат и документация: `templates/precompose/precompose-format.md`
- Precompose specs: `templates/precompose/{template_id}.precompose.yaml`

---

## Структура файла

### Секция: Идентификация

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `template_id` | string | yes | Ссылка на template. Должен совпадать с именем файла. |
| `version` | integer | yes | Версия precompose spec. Начинается с 1. |
| `status` | string | yes | `proposed`, `active`, `deprecated` |

### Секция: Defaults

Параметры по умолчанию, применяемые ко всем aspect ratios, если не переопределены.

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `safe_area_margin` | string | yes | Минимальный отступ, % от меньшей стороны canvas. Формат: `"N%"` |
| `background_fill_strategy` | string | yes | `source_bg` или `solid_color` |
| `background_fill_color` | string | no | Hex-цвет, используется только при `solid_color`. Пример: `"#F5F5F5"` |
| `dish_placement` | string | yes | Стратегия размещения блюда: `center` (V1: единственный вариант) |

### Секция: Canvas specs (per aspect_ratio)

Список `canvas_specs`, по одному элементу на каждый supported aspect_ratio из template.

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `aspect_ratio` | string | yes | `1:1`, `9:16`, `16:9` |
| `resolution` | string | yes | Целевое разрешение. Формат: `"WxH"` |
| `safe_area_margin` | string | no | Override default margin для этого aspect_ratio |
| `notes` | string | no | Специфические заметки для этого aspect_ratio |

### Секция: Template-specific notes

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `motion_considerations` | string | no | Как motion тип template влияет на precompose (например, нужен ли extra margin для rotation) |
| `angle_notes` | list[object] | no | Per-angle заметки для precompose |
| `general_notes` | string | no | Свободные заметки |

Структура `angle_notes`:

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `angle_family` | string | yes | `top_down`, `three_quarter`, `hero_side` |
| `note` | string | yes | Заметка по precompose для этого ракурса |

---

## Связь с template spec

Template spec (`precompose_rules` секция) содержит:
- `aspect_ratio_strategy: precompose` — подтверждает использование precompose
- `safe_area_margin` — default значение (дублируется из precompose spec для быстрого обзора)
- `background_fill` — hint (source_bg / solid_color)
- `notes` — краткие заметки

Precompose spec расширяет эту информацию:
- per-aspect-ratio resolution и canvas specs
- per-angle notes
- motion-specific considerations

**При расхождении** между template.precompose_rules и precompose spec —
source of truth **precompose spec** (более детальный документ).

---

## Связь с precompose_input (domain model)

Precompose spec определяет **правила**. `precompose_input` — конкретный **результат**
применения этих правил к конкретному source image.

```
precompose spec (rules)
  + source image (input)
  + target aspect_ratio (parameter)
  = precompose_input (result = prepared start frame)
```

Атрибуты `precompose_input` из domain model заполняются так:

| Атрибут | Источник |
|---------|---------|
| precompose_id | Генерируется (или совпадает с file path) |
| source_image_ref | Входной параметр |
| template_id | Из precompose spec |
| target_aspect_ratio | Входной параметр |
| angle_family | Из source image metadata / входного параметра |
| output_resolution | Из canvas_specs precompose spec |
| canvas_params | Из defaults + canvas_specs precompose spec |
| file_path | По конвенции: `fixtures/prepped-images/{source_name}_{aspect_shortcode}.png` |

---

## Связь с run record

Run record ссылается на результат precompose через:
- `start_frame_ref` — путь к precomposed image (ADR-0004, П-7)
- `precompose_notes` — заметки, если были отклонения от spec

---

## Что precompose spec НЕ содержит

| Исключённое | Почему |
|---|---|
| Код или скрипты precompose | Формат декларативный, implementation отдельно |
| Ссылки на конкретные source images | Spec описывает правила, не конкретные inputs |
| Scoring criteria | Это test_suite / scoring rubric |
| Prompt modifiers | Это prompt_pack |

---

## Открытые вопросы

| # | Вопрос | Когда решается |
|---|--------|----------------|
| 1 | Нужен ли precompose_id как отдельное поле или достаточно file path? | Phase 4 (при реализации pipeline) |
| 2 | Нужна ли автоматическая валидация precompose output (размеры, safe area)? | Phase 4–5 |
| 3 | Как обрабатывать source images, которые уже имеют целевой aspect ratio? | Phase 4 (всё равно проходят precompose — ADR-0004, П-1) |
