# Kling Response Format — food-video-template-lab

## Статус

Proposed (Phase 4, batch 2) — обновлён по Kie AI docs (Phase 4 correction)

## Дата

2026-03-26

## Назначение

Описывает response format от Kie AI API и нормализованный internal task record для V1.

Canonical sources: ADR-0005, kling-request-format.md, run-record-format.

---

## Важно

> Структура верхнего уровня зафиксирована по официальной Kie AI документации.
> Значения, помеченные _(proposed)_, требуют верификации smoke test (ADR-0006).

---

## 1. Task Creation Response

Возвращается при `POST /api/v1/jobs/createTask`.

Ожидаемое (proposed — верифицировать точную структуру):

```json
{ "taskId": "task_xxxxxxxxxxxxxxxx" }
```

**Что делаем:**
- Сохранить `taskId` → `api_task_id` в run record
- Сохранить timestamp → `api_submitted_at`
- Перейти к ожиданию результата (polling или callBackUrl)

---

## 2. Task Status Response

Используется при `GET /api/v1/jobs/recordInfo?taskId={taskId}` (polling)
и как payload callBackUrl (при production webhook).

### Documented terminal state — success

```json
{
  "taskId": "task_xxxxxxxxxxxxxxxx",
  "state": "success",
  "resultJson": {
    "<video_url_field>": "https://...",
    "<duration_field>": 5.04,
    "<resolution_field>": "1080x1080"
  },
  "failCode": null,
  "failMsg": null,
  "costTime": 222,
  "timestamps": { "...": "..." }
}
```

> Точные поля внутри `resultJson` (имена видео URL, длительности, разрешения) — proposed.
> Верифицировать в smoke test (A5).

### Failure state (to observe)

```json
{
  "taskId": "task_xxxxxxxxxxxxxxxx",
  "state": "<failure_state_value>",
  "resultJson": null,
  "failCode": "<code>",
  "failMsg": "<message>",
  "costTime": null,
  "timestamps": { "...": "..." }
}
```

> Точный string value failure state — к наблюдению в smoke test.

### Non-terminal states (documented)

```json
{
  "taskId": "task_xxxxxxxxxxxxxxxx",
  "state": "waiting"
}
```

State sequence: `"waiting"` → `"queuing"` → `"generating"` → terminal

---

## 3. Top-level Response Fields

| Поле | Тип | Описание |
|------|-----|----------|
| `taskId` | string | ID задачи в системе Kie AI |
| `state` | string | Текущий state. Значения — см. task-status-mapping.md |
| `resultJson` | object \| null | Данные результата при `success`. Структура — proposed, верифицировать |
| `failCode` | string \| null | Код ошибки при failure state |
| `failMsg` | string \| null | Сообщение ошибки при failure state |
| `costTime` | number \| null | Время генерации при success |
| `timestamps` | object \| null | Временны́е метки (структура — к наблюдению) |

---

## 4. resultJson (при success)

Внутри `resultJson` ожидаются видео-данные.
Точные имена полей — proposed, выводятся из smoke test.

| Ожидаемое содержимое | Proposed field name | Верифицировать |
|---------------------|--------------------|----|
| URL видео | неизвестно | В smoke test (A5) |
| Длительность | неизвестно | В smoke test (A5) |
| Разрешение | неизвестно | В smoke test (A5) |

---

## 5. Нормализованный Internal Task Record

### Поля

| Поле | Тип | Источник | Описание |
|------|-----|----------|----------|
| `task_id` | string | provider (`taskId`) | Идентификатор задачи |
| `run_record_ref` | string | internal | Ссылка на run_id |
| `status` | string | mapped | Нормализованный статус — см. task-status-mapping.md |
| `result_source` | string | internal | `"callback"` или `"polling"` |
| `submitted_at` | ISO 8601 | internal | Timestamp отправки |
| `completed_at` | ISO 8601 \| null | internal | Timestamp терминального state |
| `video_url` | string \| null | provider (из resultJson) | URL видео |
| `duration_actual` | number \| null | provider (из resultJson) | Фактическая длительность |
| `resolution` | string \| null | provider (из resultJson) | Разрешение |
| `provider_error` | string \| null | provider (`failMsg`) | Ошибка провайдера |
| `internal_error` | string \| null | internal | Внутренняя ошибка (timeout и т.п.) |

### V1 решение

> Task record встроен в run record (секции `api_*` + `output`).
> Отдельный `.task-record.json` — только illustrative example артефакт.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Структура submission response (точные поля) | Proposed — верифицировать |
| 2 | Поля внутри `resultJson` (видео URL, длительность, разрешение) | Верифицировать smoke test (A5) |
| 3 | Failure state string value | Верифицировать smoke test |
| 4 | Структура `timestamps` объекта | Верифицировать |
| 5 | TTL video URL из `resultJson` | Уточнить у Kie AI |
| 6 | callBackUrl payload: идентичен polling response? | Proposed: да. Верифицировать |
