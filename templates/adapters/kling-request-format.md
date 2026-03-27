# Kling Request Format — food-video-template-lab

## Статус

Proposed (Phase 4, batch 1) — частично обновлён по Kie AI docs (Phase 4 correction)

## Дата

2026-03-26

## Назначение

Определяет формат запроса к Kling 3.0 via Kie AI API для V1.

> **Важно:** endpoint paths и request shape зафиксированы по официальной Kie AI документации.
> Значения, помеченные _(proposed)_, требуют верификации первым smoke test (ADR-0006).

Canonical sources: ADR-0002, ADR-0005, Run record format.

---

## Принципы

1. **Минимальный request.** Только обязательные поля.
2. **Собран из domain-данных.** Каждое поле имеет чёткий source.
3. **aspect_ratio передаётся явно** — для детерминированного output contract
   в дополнение к precompose (ADR-0005, А-3).
4. **V1 = фиксированная конфигурация.** Вариативны: prompt, image URL, duration, aspect_ratio.

---

## Endpoint

```
POST {kie_api_base_url}/api/v1/jobs/createTask
Authorization: Bearer <KIEAI_API_TOKEN>
Content-Type: application/json
```

---

## Request Shape

Request body использует nested структуру `model` + `input`:

```json
{
  "model": "<model_identifier>",
  "input": {
    "image_url": "<start_frame_url>",
    "prompt": "<assembled_prompt>",
    "negative_prompt": "<negative_prompt>",
    "duration": "<duration>",
    "mode": "pro",
    "aspect_ratio": "<aspect_ratio>"
  }
}
```

---

## Request Fields (V1)

### Верхний уровень

| Поле | Тип | V1 значение / источник | Описание |
|------|-----|----------------------|----------|
| `model` | string | `"kling-3.0"` _(proposed — верифицировать по Kie docs)_ | Идентификатор модели |

### Объект `input`

| Поле | Тип | V1 значение / источник | Описание |
|------|-----|----------------------|----------|
| `image_url` | string | `start_frame_url` | URL precomposed start frame _(поле vs `image_urls` — верифицировать)_ |
| `prompt` | string | assembled_prompt | Motion prompt с applied modifiers |
| `negative_prompt` | string | prompt_pack.negative_prompt | Anti-drift guidance |
| `duration` | string | `"5"` или `"10"` | Длительность _(строка vs число — верифицировать)_ |
| `mode` | string | `"pro"` (fixed) | Режим генерации |
| `aspect_ratio` | string | run params | `"1:1"`, `"9:16"`, `"16:9"` _(формат — верифицировать)_ |

### callBackUrl (polling mode = опустить)

| Поле | Тип | Описание |
|------|-----|----------|
| `callBackUrl` | string \| omitted | Webhook URL (production). Опустить → polling mode |

> Точное название поля (`callBackUrl` vs `callback_url`) — верифицировать по docs.

### Поля, не используемые в V1

| Поле | V1 поведение | Причина |
|------|-------------|---------|
| End frame / tail image | не передаётся | V1: start frame only (ADR-0002) |
| `camera_control` | не передаётся | Управление через prompt |

---

## Mapping: Domain → API Request

```
template
  └── prompt_pack
        ├── motion_prompt ──────────────┐
        ├── negative_prompt ─────────────┼──► input fields
        └── modifiers                    │
              ├── by_angle ──────────────┘
              └── by_ratio ──────────────┘

precompose_input
  └── file_path → (upstream upload) → URL ──► input.image_url

run params
  ├── duration ────────────────────────────► input.duration
  └── aspect_ratio ────────────────────────► input.aspect_ratio

adapter config (fixed)
  ├── model ───────────────────────────────► model
  └── mode ────────────────────────────────► input.mode
```

### Prompt assembly (выполняется до вызова adapter)

```
assembled_prompt = motion_prompt
                 + ", " + by_angle[angle_family].append      (если непустой)
                 + ", " + by_aspect_ratio[aspect_ratio].append   (если непустой)
```

---

## Image delivery

Adapter получает готовый URL. Upstream upload workflow — TBD (за рамками V1 adapter contract).

---

## Authentication

```
Authorization: Bearer <KIEAI_API_TOKEN>
```

Token хранится в env variable, не в коде.

---

## Response Model (при submission)

При успешной отправке API возвращает `taskId` задачи.
Точная структура submission response — верифицировать по docs.

Ожидаемое (proposed):
```json
{ "taskId": "..." }
```

Результат задачи получается через `GET /api/v1/jobs/recordInfo?taskId={taskId}`.
Детали response — см. `kling-response-format.md`.

---

## Пример Request Body (JSON)

```json
{
  "model": "kling-3.0",
  "input": {
    "image_url": "https://<image-host>/prepped-images/plate_pasta_three_quarter_1x1.png",
    "prompt": "Subtle smooth slow orbital camera rotation around the food dish, approximately 10 degrees of arc. The dish stays centered in frame. Minimal movement, emphasizing volume and texture of the food. Stable consistent background throughout, three-quarter elevated angle, orbiting around the dish",
    "negative_prompt": "morphing, deformation, melting, shape change, new objects appearing, hands, people, utensils appearing, background change, color shift, fast movement, jerky motion, zoom, camera shake, blurry",
    "duration": "5",
    "mode": "pro",
    "aspect_ratio": "1:1"
  }
}
```

Полный аннотированный пример — в `examples/orbit_micro_default.kling-request.json`.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Точный model identifier для Kling 3.0 | Proposed: `"kling-3.0"`. Верифицировать smoke test |
| 2 | `image_url` vs `image_urls` внутри `input` | Proposed: `image_url`. Верифицировать smoke test |
| 3 | `duration`: строка `"5"` или число `5` | Proposed: строка. Верифицировать smoke test |
| 4 | Формат `aspect_ratio` | Proposed: `"1:1"`. Верифицировать smoke test |
| 5 | Точное название поля для webhook: `callBackUrl` vs иное | Верифицировать по docs |
| 6 | Upstream image upload workflow | Открыт — TBD |
