# Task Status Mapping — food-video-template-lab

## Статус

Proposed (Phase 4, batch 2) — обновлён по Kie AI docs (Phase 4 correction)

## Дата

2026-03-26

## Назначение

Маппинг провайдерских task states (Kie AI) в нормализованные внутренние статусы V1.

Canonical sources: ADR-0005, kling-response-format.md, run-record-format.

---

## Важно

> Task state values (`waiting`, `queuing`, `generating`, `success`) зафиксированы
> по официальной Kie AI документации.
> Failure state и полный список states требуют верификации smoke test.

---

## Internal Status Model

| Internal status | Описание |
|-----------------|----------|
| `pending` | Задача принята, ожидает начала или стоит в очереди |
| `running` | Задача активно генерируется |
| `success` | Видео сгенерировано, видео-данные доступны в `resultJson` |
| `failed` | Провайдер вернул ошибку |
| `timeout` | Polling превысил max_wait без терминального state |

---

## Provider → Internal Mapping

| Provider state | Internal status | Терминальный? | Действие |
|---------------|-----------------|---------------|----------|
| `waiting` | `pending` | Нет | Продолжить ожидание |
| `queuing` | `pending` | Нет | Продолжить ожидание |
| `generating` | `running` | Нет | Продолжить ожидание |
| `success` | `success` | **Да** | Извлечь видео из `resultJson`, обновить run record |
| _(failure state — к наблюдению)_ | `failed` | **Да** | Сохранить `failCode`/`failMsg`, обновить run record |
| _(polling timeout)_ | `timeout` | **Да** | Зафиксировать `internal_error`, обновить run record |
| _(неизвестный state)_ | — | — | Логировать, не обновлять статус, оповестить оператора |

> Точный string value failure state — к наблюдению в smoke test.
> Может быть `"failed"`, `"error"` или иное значение.

---

## Поведение при каждом статусе

### `pending` (waiting / queuing)

- Ничего не обновлять в run record
- Продолжить polling или ждать callBackUrl

### `running` (generating)

- Ничего не обновлять в run record
- Продолжить polling или ждать callBackUrl

### `success`

1. Установить `generation_status = success`
2. Извлечь видео-данные из `resultJson` (поля — верифицировать smoke test)
3. Сохранить `video_url`, `duration_actual`, `resolution`
4. Установить `api_completed_at`
5. Скачать видео → `runs/{template_id}/{run_id}/output.mp4`
6. Перейти к scoring

### `failed`

1. Установить `generation_status = failed`
2. Сохранить `provider_error` (из `failMsg`)
3. Сохранить `failCode` (если полезен для диагностики)
4. Установить `api_completed_at`
5. **Не удалять** run record
6. **Не выставлять** score

### `timeout`

1. Установить `generation_status = timeout`
2. Установить `internal_error = "polling_timeout: exceeded {max_wait}s without terminal state"`
3. **Не удалять** run record
4. Проверить state задачи вручную позже

---

## generation_status в run record

| Internal status | `generation_status` |
|-----------------|---------------------|
| `success` | `success` |
| `failed` | `failed` |
| `timeout` | `timeout` |

---

## Non-recoverable vs Potentially recoverable

| Тип | Примеры | Действие |
|-----|---------|----------|
| Non-recoverable (`failed`) | Content policy, invalid input, model error | Зафиксировать, не повторять автоматически |
| Potentially recoverable (`timeout`) | API перегрузка, задержка провайдера | Проверить вручную, при необходимости новый run |

> V1: нет автоматического retry. Повторный run = новый `run_id`.

---

## Открытые вопросы

| # | Вопрос | Статус |
|---|--------|--------|
| 1 | Точный string value failure state (`"failed"`, `"error"`, иное?) | Верифицировать smoke test |
| 2 | Есть ли промежуточные states кроме waiting/queuing/generating? | Верифицировать |
| 3 | Behaviour при failure: `state` = failure value или отдельное поле? | Верифицировать |
