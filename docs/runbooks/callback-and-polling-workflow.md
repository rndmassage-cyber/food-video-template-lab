# Callback and Polling Workflow — food-video-template-lab

## Статус

Proposed (Phase 4, batch 2)

## Дата

2026-03-26

## Назначение

Описывает, как получать результат от Kling API после отправки задачи.
Два поддерживаемых метода: **callBackUrl (production preferred)** и **polling (dev/manual допустим)**.

Canonical sources: ADR-0005, run-execution-workflow.md, task-status-mapping.md, kling-response-format.md.

---

## Обзор

После `POST` запроса Kling API возвращает `task_id` и `status: "pending"`.
Результат появляется асинхронно. V1 поддерживает два способа его получения:

| Метод | Когда использовать | Требования |
|-------|-------------------|------------|
| **callBackUrl** | Production, автоматизированные runs | Публично доступный webhook endpoint |
| **Polling** | Dev / local / manual runs | HTTP-клиент + API token |

---

## Метод 1: callBackUrl (Production Preferred)

### Принцип

При завершении задачи (success или failed) Kling API отправляет `POST` на `callback_url`,
переданный в исходном запросе. Payload — структура ответа из kling-response-format.md.

### Когда передавать callback_url

Поле `callback_url` включается в Kling request, если есть публичный endpoint.
Если поле опущено — задача выполнится, но уведомления не будет (потребуется polling).

### Что делать при получении callback

1. Верифицировать подлинность запроса (webhook secret — если поддерживается Kie AI; уточнить)
2. Извлечь `task_id` и `status` из payload
3. Найти соответствующий run record по `task_id`
4. Применить task-status-mapping (см. task-status-mapping.md):
   - `completed` → `success`: скачать видео, обновить run record output
   - `failed` → `failed`: сохранить `error_message`, обновить run record
5. Установить `api_completed_at` = timestamp получения callback
6. Обновить `updated_at` в run record

### Ошибки при обработке callback

| Ситуация | Действие |
|----------|----------|
| `task_id` не найден в run records | Логировать аномалию, ничего не обновлять |
| Не удалось скачать `video_url` | Сохранить `video_url`, `video_file_path = null`, повторить скачивание вручную |
| Повторный callback (дубль) | Если `generation_status` уже терминальный — игнорировать |

> **Открытый вопрос:** поддерживает ли Kie AI webhook secret?
> Верифицировать по документации перед production-деплоем.

---

## Метод 2: Polling (Dev / Local / Manual)

### Принцип

Оператор (или скрипт) периодически запрашивает статус задачи через GET endpoint.
Polling продолжается до получения терминального статуса или истечения max_wait.

### Endpoint _(proposed — верифицировать по Kie docs)_

```
GET {kie_api_base_url}/v1/videos/tasks/{task_id}
Authorization: Bearer <KIEAI_API_TOKEN>
```

### Polling schedule

| Параметр | Значение |
|----------|----------|
| `initial_delay` | 2–3s после отправки запроса |
| `backoff` | Exponential: 3s → 6s → 12s → 24s → 30s (cap) |
| `max_wait` | 600s (10 минут) |
| `timeout_action` | `generation_status = timeout`, зафиксировать `internal_error` |

### Алгоритм polling loop

```
submit request → получить task_id
wait initial_delay (2–3s)

loop:
  GET /tasks/{task_id}
  match provider_status:
    "pending" | "processing" → wait next_interval (exponential); continue
    "completed"              → goto SUCCESS
    "failed"                 → goto FAILED
    unknown                  → log warning; continue (если повторяется — break)

  if elapsed > max_wait → goto TIMEOUT

SUCCESS:
  применить task-status-mapping: completed → success
  download video_url → runs/{template_id}/{run_id}/output.mp4
  обновить run record (generation_status=success, output fields)
  перейти к scoring

FAILED:
  применить task-status-mapping: failed → failed
  обновить run record (generation_status=failed, api_error)
  не выставлять score

TIMEOUT:
  установить generation_status = timeout
  установить internal_error = "polling_timeout: exceeded 600s"
  обновить run record
  проверить статус задачи вручную позже
```

### Почему exponential backoff

Kling generation обычно занимает 2–5 минут для 5s видео в pro mode.
Агрессивный polling может исчерпать rate limits. Exponential backoff снижает нагрузку
и укладывается в типичное время генерации.

---

## Что сохранять при каждом исходе

### Success

| Поле в run record | Значение |
|-------------------|----------|
| `generation_status` | `success` |
| `api_completed_at` | timestamp терминального статуса |
| `video_url` | из provider response |
| `video_file_path` | `runs/{template_id}/{run_id}/output.mp4` |
| `duration_actual` | из provider response (если есть) |
| `resolution` | из provider response (если есть) |
| `api_error` | `null` |

### Failed

| Поле в run record | Значение |
|-------------------|----------|
| `generation_status` | `failed` |
| `api_completed_at` | timestamp терминального статуса |
| `video_url` | `null` |
| `video_file_path` | `null` |
| `api_error` | текст из `error_message` провайдера |

### Timeout

| Поле в run record | Значение |
|-------------------|----------|
| `generation_status` | `timeout` |
| `api_completed_at` | `null` |
| `video_url` | `null` |
| `video_file_path` | `null` |
| `api_error` | `"polling_timeout: exceeded 600s"` |

---

## Связь с run record

Оба метода завершаются одинаковым результатом: обновлённый run record с заполненной output-секцией.
Единственное отличие — `result_source` в task record: `"callback"` или `"polling"`.

Workflow после получения результата (шаги 9–12 из run-execution-workflow.md) одинаков для обоих методов.

---

## Рекомендации для V1

1. **Production / staging:** всегда передавать `callback_url`. Polling — только fallback.
2. **Local dev:** polling допустим. Использовать ручной polling (не автоматизированный loop).
3. **После timeout:** не удалять run record. Задача может всё ещё выполняться у провайдера.
4. **video_url TTL:** скачивать видео немедленно после success. Срок жизни ссылки неизвестен.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Webhook secret для верификации callback | Уточнить по Kie docs перед production |
| 2 | TTL video_url (CDN link expiration) | Уточнить у Kie AI |
| 3 | Rate limits на polling endpoint | Уточнить по Kie docs |
| 4 | Callback payload идентичен polling response по структуре? | Proposed: да. Верифицировать |
| 5 | Retry со стороны Kie при недоступном callback endpoint? | Уточнить по Kie docs |
