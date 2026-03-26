# Task Status Mapping — food-video-template-lab

## Статус

Proposed (Phase 4, batch 2)

## Дата

2026-03-26

## Назначение

Определяет маппинг провайдерских статусов (Kie AI) в нормализованные внутренние статусы V1.

Canonical sources: ADR-0005, kling-response-format.md, run-record-format.

---

## Важно

> Точные статусы провайдера **должны верифицироваться по актуальной Kie AI документации**.
> Значения ниже — proposed, основаны на официальных примерах и community knowledge.

---

## Internal Status Model

V1 использует минимальный набор из 5 нормализованных статусов:

| Internal status | Описание |
|-----------------|----------|
| `pending` | Задача отправлена, ожидает начала обработки |
| `running` | Задача обрабатывается провайдером |
| `success` | Видео успешно сгенерировано, `video_url` доступен |
| `failed` | Провайдер вернул ошибку (content policy, model error, etc.) |
| `timeout` | Polling превысил max_wait без терминального статуса |

> `timeout` — внутренний концепт. Провайдер не возвращает такой статус.
> Фиксируется на нашей стороне, когда polling loop завершается по max_wait.

---

## Provider → Internal Mapping

| Provider status | Internal status | Терминальный? | Действие |
|-----------------|-----------------|---------------|----------|
| `pending` | `pending` | Нет | Продолжить ожидание |
| `processing` | `running` | Нет | Продолжить ожидание |
| `completed` | `success` | **Да** | Скачать видео, обновить run record |
| `failed` | `failed` | **Да** | Сохранить `error_message`, обновить run record |
| _(polling timeout)_ | `timeout` | **Да** | Зафиксировать `internal_error`, пометить run record |
| _(неизвестный статус)_ | — | — | Логировать, не обновлять статус, оповестить оператора |

---

## Поведение при каждом статусе

### `pending` / `running`

- Ничего не обновлять в run record
- Продолжить polling или ждать callback
- Polling strategy — см. callback-and-polling-workflow.md

### `success`

1. Установить `generation_status = success`
2. Сохранить `video_url`
3. Сохранить `duration_actual`, `resolution` (если есть в response)
4. Установить `api_completed_at`
5. Скачать видео → `runs/{template_id}/{run_id}/output.mp4`
6. Перейти к scoring

### `failed`

1. Установить `generation_status = failed`
2. Сохранить `provider_error` (из `error_message`)
3. Установить `api_completed_at`
4. **Не удалять** run record
5. **Не выставлять** score
6. Записать заметку оператора, если причина известна

### `timeout`

1. Установить `generation_status = timeout`
2. Установить `internal_error = "polling_timeout: exceeded {max_wait}s without terminal status"`
3. **Не удалять** run record
4. Рекомендация: проверить статус задачи вручную позже — задача может всё ещё выполняться
5. Если задача позже завершилась — обновить run record вручную

---

## Как generation_status отражается в run record

| Internal status | `generation_status` в run record |
|-----------------|----------------------------------|
| `success` | `success` |
| `failed` | `failed` |
| `timeout` | `timeout` |

---

## Non-recoverable vs Potentially recoverable

| Тип | Примеры | Действие |
|-----|---------|----------|
| Non-recoverable (`failed`) | Content policy, invalid input, model error | Зафиксировать, не повторять автоматически |
| Potentially recoverable (`timeout`) | API перегрузка, задержка провайдера | Проверить вручную, при необходимости — новый run |

> В V1 нет автоматического retry. Повторный run = новый `run_id`, предыдущий record не изменяется.

---

## Partial result

Kling API не документирует partial completion для V1 сценария.
В V1 partial case не предусмотрен.
Если API вернёт статус, не входящий в mapping — логировать как неизвестный, не трактовать как success.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Полный список provider statuses (есть ли `queued`, `cancelled`, иные?) | Верифицировать по Kie docs |
| 2 | При `failed`: API возвращает HTTP error или 200 с error в body? | Верифицировать |
| 3 | Нужен ли статус `cancelled` для V1? | Отложено — V1 без отмены задач |
