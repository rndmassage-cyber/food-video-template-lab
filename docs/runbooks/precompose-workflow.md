# Runbook: Precompose Workflow

## Статус

Proposed (Phase 3, batch 1)

## Дата

2026-03-25

## Назначение

Пошаговое руководство по подготовке start frame (precompose)
для video generation run. V1 — ручной workflow.

Canonical sources: ADR-0004 (precompose policy),
precompose-format.md (формат spec), template spec (precompose_rules).

---

## Когда выполнять

Precompose выполняется **перед каждым run**, если подходящий precomposed frame
ещё не существует. Проверка кэша — шаг 1.

---

## Входные данные

| Данные | Источник | Пример |
|--------|---------|--------|
| source image | `fixtures/raw-inputs/` | `plate_pasta_three_quarter.png` |
| template_id | Выбран пользователем | `orbit_micro_default` |
| target aspect_ratio | Выбран пользователем | `9:16` |

---

## Шаги

### Шаг 1. Проверить кэш

Проверить, существует ли уже подготовленный frame:

```
fixtures/prepped-images/{source_name}_{aspect_shortcode}.png
```

Пример: `fixtures/prepped-images/plate_pasta_three_quarter_9x16.png`

**Если файл существует** и выполняются условия:
- Precompose spec (version) не изменился с момента создания файла
- Source image не обновлялся

→ Переиспользовать. Перейти к «Результат».

**Если файла нет или условия не выполняются** → продолжить со шага 2.

### Шаг 2. Загрузить precompose spec

Открыть precompose spec template:

```
templates/precompose/{template_id}.precompose.yaml
```

Найти `canvas_specs` для целевого `aspect_ratio`.
Определить:
- `resolution` (например `1080x1920`)
- `safe_area_margin` (override из canvas_specs или default)
- `background_fill_strategy`

### Шаг 3. Определить параметры canvas

1. **Canvas size** = resolution из spec (например 1080 × 1920).
2. **Margin в пикселях** = safe_area_margin (%) × min(canvas_width, canvas_height).
   - Пример: 12% × 1080 = 130 px.
3. **Available area** = canvas размер минус margin с каждой стороны.
   - Пример: (1080 − 2×130) × (1920 − 2×130) = 820 × 1660.

### Шаг 4. Определить размер блюда на canvas

1. Взять source image dimensions.
2. Вписать source image в available area с сохранением пропорций (fit, не fill).
3. Итоговый размер = максимальный, при котором image целиком помещается в available area.

**Важно:** source image **не кропается** (ADR-0004, П-2).

### Шаг 5. Определить background

**При `source_bg`:**
1. Сэмплировать цвет с краёв source image (углы или edge strip).
2. Визуально проверить однотонность.
3. Если фон неоднороден — переключиться на `solid_color` с ручным подбором.

**При `solid_color`:**
1. Использовать цвет из `background_fill_color` в precompose spec.
2. Если не задан — подобрать вручную, максимально близко к фону source image.

### Шаг 6. Собрать start frame

1. Создать canvas заданного resolution.
2. Залить canvas background цветом.
3. Разместить source image по центру (dish_placement: center).
4. Сохранить в PNG формате.

### Шаг 7. Проверить результат

Визуальная проверка:

| Проверка | Критерий |
|----------|---------|
| Aspect ratio | Совпадает с target |
| Resolution | Совпадает со spec |
| Блюдо видно целиком | Нет обрезки |
| Margins | Блюдо не ближе safe_area_margin к любому краю |
| Фон | Однотонный, без видимых границ между source background и canvas fill |
| Артефакты | Нет видимых швов, полос, градиентов на фоне |

Если проверка не пройдена → вернуться к шагу 5 (скорректировать background)
или шагу 4 (скорректировать размер).

### Шаг 8. Сохранить результат

Сохранить файл по конвенции:

```
fixtures/prepped-images/{source_name}_{aspect_shortcode}.png
```

---

## Результат

После precompose доступен start frame, который:
- Используется как `start_frame_ref` в run record
- Передаётся в Kling API как start frame для генерации видео

---

## Quick reference: aspect_shortcode

| aspect_ratio | shortcode |
|-------------|-----------|
| 1:1 | `1x1` |
| 9:16 | `9x16` |
| 16:9 | `16x9` |

---

## Quick reference: resolution targets (V1)

| aspect_ratio | resolution |
|-------------|-----------|
| 1:1 | 1080 × 1080 |
| 9:16 | 1080 × 1920 |
| 16:9 | 1920 × 1080 |

---

## Troubleshooting

### Фон source image неоднотонный

В V1 source images должны иметь однотонный фон (ADR-0002, п.4).
Если фон неоднородный — это проблема source image, а не precompose.
Варианты:
1. Переснять / перегенерировать source image.
2. Если отклонения минимальны — использовать `solid_color` с подобранным цветом.

### Source image слишком маленькое

Если source image не может заполнить available area без upscale:
1. Разместить as-is (с большими margins) — если блюдо выглядит приемлемо.
2. Upscale source image до precompose — если блюдо слишком мелкое.

Рекомендуемый минимум source image: 800px по меньшей стороне.

### Блюдо не по центру в source image

Если блюдо смещено в source image:
1. Учесть смещение при placement (сдвинуть вручную).
2. Зафиксировать в `precompose_notes` run record.

В V1 source images из собственного pipeline, где блюдо должно быть в safe area.
Если это не так — проблема pipeline, не precompose.

---

## Ссылки

- [ADR-0004 — Precompose First Frame Policy](../adr/ADR-0004-precompose-first-frame-policy.md)
- [Precompose Format](../../templates/precompose/precompose-format.md)
- [Run Record Format](../../templates/runs/run-record-format.md)
