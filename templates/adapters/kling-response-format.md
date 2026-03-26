# Kling Response Format — food-video-template-lab

## Статус

Proposed (Phase 4, batch 2)

## Дата

2026-03-26

## Назначение

Описывает raw response format от Kie AI API и нормализованный internal task record для V1.

Canonical sources: ADR-0005, kling-request-format.md, run-record-format.

---

## Важно

> Точные имена полей, структура вложенности и типы данных **должны верифицироваться
> по актуальной Kie AI документации** перед первым реальным запросом.
> Всё в этом документе — proposed, основано на официальных примерах и community knowledge.

---

## 1. Task Creation Response

Возвращается синхронно при POST на endpoint image2video (endpoint — верифицировать по Kie docs).

### Raw provider response

```json
{
  "task_id": "task_xxxxxxxxxxxxxxxx",
  "status": "pending"
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| `task_id` | string | Уникальный ID задачи в системе Kie AI. Сохраняется как `api_task_id` в run record |
| `status` | string | Начальный статус. Ожидаемое значение: `"pending"` |

**Что делаем:**
- Сохранить `task_id` → `api_task_id` в run record
- Сохранить timestamp → `api_submitted_at`
- Перейти к ожиданию результата (callback или polling)

---

## 2. Task Status Response

Используется в двух сценариях:
- **Polling:** тело ответа на `GET /v1/videos/tasks/{task_id}`
- **Callback:** тело POST-запроса, отправленного Kling на `callback_url`

### Raw provider response (terminal — completed)

```json
{
  "task_id": "task_xxxxxxxxxxxxxxxx",
  "status": "completed",
  "video_url": "https://cdn.kieai.com/videos/task_xxxxxxxxxxxxxxxx.mp4",
  "duration_actual": 5.04,
  "resolution": "1080x1080",
  "error_message": null
}
```

### Raw provider response (terminal — failed)

```json
{
  "task_id": "task_xxxxxxxxxxxxxxxx",
  "status": "failed",
  "video_url": null,
  "duration_actual": null,
  "resolution": null,
  "error_message": "Content policy violation: ..."
}
```

### Raw provider response (non-terminal)

```json
{
  "task_id": "task_xxxxxxxxxxxxxxxx",
  "status": "processing"
}
```

### Response fields

| Поле | Тип | Терминальный? | Описание |
|------|-----|--------------|----------|
| `task_id` | string | — | ID задачи |
| `status` | string | — | Текущий статус. Значения — см. task-status-mapping.md |
| `video_url` | string \| null | при completed | CDN URL готового видео. TTL неизвестен — уточнить |
| `duration_actual` | number \| null | при completed | Фактическая длительность (секунды) _(proposed — верифицировать)_ |
| `resolution` | string \| null | при completed | Фактическое разрешение: `"1080x1080"` _(proposed — верифицировать)_ |
| `error_message` | string \| null | при failed | Описание ошибки от провайдера |

---

## 3. Нормализованный Internal Task Record

### Зачем отдельная нормализация

Raw provider response:
- Имена полей могут меняться при смене API версии
- Статусы провайдера ≠ статусы внутренней модели (`timeout` — внутренний концепт)
- Нужно зафиксировать контекст: timestamp получения ответа, откуда пришёл результат

Нормализованная запись изолирует run record от деталей провайдера.

### Поля normalized task record

| Поле | Тип | Источник | Описание |
|------|-----|----------|----------|
| `task_id` | string | provider | Идентификатор задачи на стороне Kie AI |
| `run_record_ref` | string | internal | Ссылка на run record (`run_id`) |
| `status` | string | mapped | Нормализованный статус — см. task-status-mapping.md |
| `result_source` | string | internal | `"callback"` или `"polling"` |
| `submitted_at` | ISO 8601 | internal | Timestamp отправки запроса |
| `completed_at` | ISO 8601 \| null | internal | Timestamp получения терминального статуса |
| `video_url` | string \| null | provider | CDN URL готового видео |
| `duration_actual` | number \| null | provider | Фактическая длительность |
| `resolution` | string \| null | provider | Разрешение выходного видео |
| `provider_error` | string \| null | provider | `error_message` от провайдера |
| `internal_error` | string \| null | internal | Ошибка на нашей стороне (timeout, network, etc.) |

### Связь task record с run record

Task record — детализированное состояние API-слоя.
Run record — полный lifecycle run'а: input, output, score.

Run record содержит ключевые поля из task record (`api_task_id`, `api_submitted_at`,
`api_completed_at`, `api_error`, `generation_status`).

> **V1 решение (proposed):** task record встроен в run record как API-секция.
> Отдельный `.task-record.json` файл — только illustrative example артефакт.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Точная структура completion response (поля, вложенность) | Верифицировать по Kie docs |
| 2 | TTL для `video_url` — как долго действителен CDN link? | Уточнить у Kie AI |
| 3 | Наличие `duration_actual` и `resolution` в response | Proposed — верифицировать |
| 4 | Callback payload: идентичен polling response по структуре? | Proposed: да. Верифицировать |
| 5 | Есть ли webhook secret для верификации callback? | Уточнить по Kie docs |
