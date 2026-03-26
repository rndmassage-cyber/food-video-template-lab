# Kling Request Format — food-video-template-lab

## Статус

Proposed (Phase 4, batch 1)

## Дата

2026-03-25

## Назначение

Определяет формат запроса к Kling 3.0 via Kie AI API для V1.

> **Важно:** Этот документ описывает архитектурный request model.
> Точные имена полей, допустимые значения и endpoint paths **должны сверяться
> с актуальной официальной Kie AI документацией** перед первым реальным запросом.
> Везде, где значение помечено _(proposed)_ — оно основано на официальных примерах
> или community knowledge и требует верификации.

Canonical sources: ADR-0002, ADR-0005, Run record format.

---

## Принципы

1. **Минимальный request.** Только обязательные поля.
2. **Собран из domain-данных.** Каждое поле имеет чёткий source.
3. **aspect_ratio передаётся явно** — для детерминированного output contract
   в дополнение к precompose (ADR-0005, А-3).
4. **V1 = фиксированная конфигурация.** Вариативны: prompt, image URL, duration, aspect_ratio.

---

## Request Fields (V1)

### Основные поля

| API поле | Тип | V1 значение / источник | Описание |
|----------|-----|----------------------|----------|
| `model_name` | string | `"kling-3.0"` _(proposed)_ | Идентификатор модели. **Верифицировать по Kie docs** |
| `image_urls` | array[string] | `[start_frame_url]` | URL precomposed start frame (1 элемент в V1) |
| `prompt` | string | assembled_prompt | Motion prompt с applied modifiers |
| `negative_prompt` | string | prompt_pack.negative_prompt | Anti-drift guidance |
| `duration` | string | `"5"` или `"10"` | Длительность, строковый формат _(proposed)_ |
| `mode` | string | `"pro"` (fixed) | Режим генерации |
| `aspect_ratio` | string | run params | `"1:1"`, `"9:16"`, `"16:9"` _(proposed format)_ |
| `callback_url` | string \| omitted | environment config | Webhook URL (production). Omit → polling mode |

### Поля, не используемые в V1

| API поле | V1 поведение | Причина |
|----------|-------------|---------|
| `tail_image_urls` / end frame | не передаётся | V1: start frame only (ADR-0002, п. 9) |
| `camera_control` | не передаётся | Управление камерой — через prompt |
| `cfg_scale` | не передаётся (API default) | V1: полагаемся на default модели |

---

## Mapping: Domain → API Request

```
template
  └── prompt_pack
        ├── motion_prompt ──────────────┐
        ├── negative_prompt ─────────────┼──► API fields
        └── modifiers                    │
              ├── by_angle ──────────────┘
              └── by_ratio ──────────────┘

precompose_input
  └── file_path → (upstream upload) → URL ──► image_urls[0]

run params
  ├── duration ────────────────────────────► duration
  └── aspect_ratio ────────────────────────► aspect_ratio

adapter config (fixed)
  ├── model_name ──────────────────────────► model_name
  └── mode ────────────────────────────────► mode
```

### Prompt assembly (выполняется до вызова adapter)

```
assembled_prompt = motion_prompt
                 + ", " + by_angle[angle_family].append      (если непустой)
                 + ", " + by_aspect_ratio[aspect_ratio].append   (если непустой)
```

Порядок: base → angle modifier → aspect_ratio modifier.

---

## Image delivery

В Kling request изображение передаётся как **URL** через поле `image_urls`.
Adapter не выполняет binary upload — он получает уже готовый URL.

**Upstream workflow (TBD):**
Перед вызовом adapter precomposed start frame должен быть доступен по URL.
Как именно он туда попадает (upload в cloud storage, Kie upload endpoint, иное)
— определяется отдельно, за рамками V1 adapter contract.

---

## Authentication

```
Authorization: Bearer <KIEAI_API_TOKEN>
```

Token хранится в env variable, не в коде.

---

## Response Model

### Task creation response

| Поле | Тип | Описание |
|------|-----|----------|
| `task_id` | string | Уникальный ID задачи |
| `status` | string | Начальный статус |

### Task status response / callBack payload

| Поле | Тип | Описание |
|------|-----|----------|
| `task_id` | string | ID задачи |
| `status` | string | `pending`, `processing`, `completed`, `failed` |
| `video_url` | string \| null | URL видео (при completed) |
| `error_message` | string \| null | Описание ошибки (при failed) |

> Точные имена полей response — proposed. Верифицировать по актуальной Kie AI документации.

---

## Пример Request Body (JSON)

```json
{
  "model_name": "kling-3.0",
  "image_urls": ["https://<image-host>/prepped-images/plate_pasta_three_quarter_1x1.png"],
  "prompt": "Subtle smooth slow orbital camera rotation around the food dish, approximately 10 degrees of arc. The dish stays centered in frame. Minimal movement, emphasizing volume and texture of the food. Stable consistent background throughout, three-quarter elevated angle, orbiting around the dish",
  "negative_prompt": "morphing, deformation, melting, shape change, new objects appearing, hands, people, utensils appearing, background change, color shift, fast movement, jerky motion, zoom, camera shake, blurry",
  "duration": "5",
  "mode": "pro",
  "aspect_ratio": "1:1",
  "callback_url": "https://<your-endpoint>/webhooks/kling-result"
}
```

Полный аннотированный пример — в `examples/orbit_micro_default.kling-request.json`.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Точный `model_name` для Kling 3.0 | Proposed: `kling-3.0`. Верифицировать по docs |
| 2 | Формат `aspect_ratio` в API | Proposed: `"1:1"`, `"9:16"`, `"16:9"`. Верифицировать |
| 3 | Upstream image upload workflow | Открыт — TBD |
| 4 | `duration`: строка `"5"` или число `5`? | Proposed: строка. Из официального примера |
| 5 | `cfg_scale` default — подходит ли для pro mode? | Опционально, Phase 6 |
