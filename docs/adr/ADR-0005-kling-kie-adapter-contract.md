# ADR-0005: Kling / Kie AI Adapter Contract

## Status

Proposed

## Date

2026-03-25

## Context

ADR-0002 (п. 8–9) зафиксировал: генерация видео выполняется через Kling 3.0 via Kie AI API
в режиме single-shot, start frame only, mode=pro, sound=false, multi_shots=false.

ADR-0003 определил: template не вызывает API напрямую — это делает adapter layer через run.
ADR-0004 зафиксировал: precompose обязателен, обеспечивает framing/canvas/safe area.

Phase 2–3 подготовили все входные артефакты: template spec, prompt_pack, precompose spec.
Данный ADR определяет **contract** adapter layer, а не implementation.

> **Предупреждение о стабильности API.**
> Точные имена параметров, endpoint paths и model identifiers должны сверяться
> с актуальной официальной документацией Kie AI перед каждым первым use.
> Этот ADR фиксирует архитектурный контракт — не конкретные API string literals.

## Decision

### А-1. Adapter = тонкий mapping layer

Adapter отвечает **только** за:

1. **Request assembly** — сборку API-запроса из domain-данных.
2. **Request submission** — отправку запроса в Kie AI API.
3. **Result retrieval** — через callBackUrl (production) или polling (dev/manual).
4. **Response capture** — фиксацию метаданных результата.
5. **Asset download** — скачивание видеофайла (при success).

### А-2. Adapter НЕ отвечает за

| Исключённая ответственность | Кто отвечает |
|---|---|
| Precompose start frame (framing, canvas, safe area) | precompose pipeline |
| Image hosting / upload | upstream workflow (TBD, за рамками V1) |
| Prompt assembly (modifiers) | prompt assembly logic |
| Validation входных параметров | validation layer (template.supported_*) |
| Scoring результата | scoring system |
| Orchestration нескольких runs | за рамками V1 |
| Retry policy (бизнес-уровень) | за рамками V1 |

### А-3. Input contract

Adapter принимает **prepared run request** — структуру, в которой:

| Поле | Тип | Источник | Описание |
|------|-----|----------|----------|
| `start_frame_url` | string (URL) | upstream upload | URL precomposed start frame |
| `prompt` | string | prompt assembly | Собранный motion prompt |
| `negative_prompt` | string | prompt_pack | Negative prompt as-is |
| `duration` | string | run params | Длительность в секундах, строка (`"5"`, `"10"`) |
| `aspect_ratio` | string | run params | Целевое соотношение сторон (`"1:1"`, `"9:16"`, `"16:9"`) |
| `mode` | string | fixed: `"pro"` | Режим генерации |
| `callback_url` | string \| null | environment config | URL для webhook (null = polling mode) |

**Роль precompose и aspect_ratio:**

Precompose (ADR-0004) отвечает за корректный framing, safe area и canvas preparation.
Однако `aspect_ratio` **передаётся явно** в API request — для детерминированного output contract.
Это гарантирует, что выходной видеофайл будет иметь ожидаемое соотношение сторон
независимо от внутреннего поведения Kling.

**Примечание по формату aspect_ratio:** точный формат значений (`"1:1"` vs другой вариант)
должен быть верифицирован по актуальной Kie AI документации перед первым run.

### А-4. Fixed adapter config (V1)

| Параметр | Значение V1 | Примечание |
|----------|-------------|-----------|
| `api_provider` | `kie_ai` | Единственный провайдер в V1 |
| `api_model` | `kling-3.0` | Proposed — из официального примера Kie AI. **Верифицировать по docs** |
| `mode` | `pro` | Зафиксирован ADR-0002, п. 9 |
| `auth_header` | `Authorization: Bearer <token>` | Token из env variable, не хранится в коде |
| `api_base_url` | — | **Определяется по актуальным Kie AI docs**, не цементируется здесь |

### А-5. Output contract

Adapter возвращает **run result** для секции API run record:

| Поле | Тип | Описание |
|------|-----|----------|
| `api_task_id` | string | Task ID от Kie AI API |
| `api_submitted_at` | string (ISO 8601) | Время отправки |
| `api_completed_at` | string (ISO 8601) | Время финального статуса |
| `generation_status` | string | `success`, `failed`, `timeout` |
| `video_url` | string \| null | URL видео от API (при success) |
| `video_file_path` | string \| null | Локальный путь к скачанному видео |
| `duration_actual` | number \| null | Фактическая длительность |
| `resolution` | string \| null | Разрешение output видео |
| `file_size_bytes` | integer \| null | Размер файла |
| `api_error` | string \| null | Текст ошибки |

### А-6. Result retrieval: callBackUrl vs polling

Два режима получения результата:

**Production — callBackUrl (preferred):**
- При submit передаётся `callBackUrl` (webhook endpoint).
- Kling API отправляет POST на URL при завершении задачи.
- Не требует периодических опросов, снижает latency и нагрузку.
- Требует доступного HTTP endpoint на стороне клиента.

**Dev / manual / local — polling (допустимо):**
- `callback_url = null` → adapter переходит в polling mode.
- Периодические GET-запросы статуса задачи.

**Polling schedule (V1, proposed):**

| Параметр | Значение | Примечание |
|----------|----------|-----------|
| `initial_delay` | 2–3s | Минимальная пауза до первого poll |
| `backoff_strategy` | exponential | 3s → 6s → 12s → 24s → 30s cap |
| `max_interval` | 30s | Верхняя граница интервала |
| `max_wait_time` | 600s (10 мин) | Таймаут |

Значения — proposed. Подлежат калибровке на первых test runs.

### А-7. Error handling (V1)

| Тип ошибки | Поведение | generation_status |
|-----------|-----------|-------------------|
| API reject (4xx) | Сохранить error, не повторять | `failed` |
| Network error | Сохранить error, не повторять в V1 | `failed` |
| Timeout (max_wait_time) | Сохранить последний known status | `timeout` |
| API server error (5xx) | Сохранить error, не повторять в V1 | `failed` |
| Generation failed (API status=failed) | Сохранить error details | `failed` |

**V1 policy: нет автоматических retries.** Решение о повторном запуске принимает оператор.

### А-8. File storage convention

```
runs/{template_id}/{run_id}/output.mp4
runs/{template_id}/{run_id}/thumb.jpg  (опционально)
runs/{template_id}/{run_id}.run-record.yaml
```

## Consequences

**Что получаем:**
- Формальный контракт между domain model и Kling API.
- Детерминированный output contract: aspect_ratio передаётся явно + precompose обеспечивает framing.
- callBackUrl для production снижает polling overhead.
- Чистое разделение: adapter не знает о templates, prompt packs, precompose.
- Все данные для воспроизводимости сохраняются в run record.

**Чем платим:**
- `start_frame_url` требует upstream upload workflow (TBD).
- Нет auto-retries — ручное повторение при transient errors.
- Polling values — эвристика, требует калибровки.
- Exact API contract (field names, model identifiers) требует верификации по docs.

## Открытые вопросы

| # | Вопрос | Статус | Когда |
|---|--------|--------|-------|
| 1 | Точный model_name в Kie request body | Proposed: `kling-3.0` (из офиц. примера). Верифицировать | До первого smoke test |
| 2 | Формат aspect_ratio (`"1:1"` vs другой?) | Proposed: `"1:1"`, `"9:16"`, `"16:9"`. Верифицировать | До первого smoke test |
| 3 | Upstream image upload workflow | Открыт — TBD | До первого smoke test |
| 4 | Точные endpoint paths Kie AI | Открыт — по docs | До первого smoke test |
| 5 | Оптимальные polling intervals | Proposed. Калибруется | Phase 6 |
| 6 | Rate limits / throttling needs | Открыт | До серийных runs |

## Условия пересмотра

- Kie AI изменит API contract, endpoint paths или model identifiers.
- Понадобятся automatic retries или batch orchestration.
- Upstream image upload workflow будет определён — возможна корректировка input contract.
- Kling изменит поведение aspect_ratio output.

## Ссылки

- [ADR-0002 — Границы и ограничения V1](ADR-0002-v1-scope.md)
- [ADR-0003 — Template-first architecture](ADR-0003-template-first-architecture.md)
- [ADR-0004 — Precompose first frame policy](ADR-0004-precompose-first-frame-policy.md)
- [Domain Model](../domain-model.md)
- [Run Record Format](../../templates/runs/run-record-format.md)
- [Kling Request Format](../../templates/adapters/kling-request-format.md)
