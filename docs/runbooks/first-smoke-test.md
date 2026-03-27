# First Smoke Test — food-video-template-lab

## Статус

Proposed (Phase 4.5)

## Дата

2026-03-26

## Назначение

Пошаговый runbook для первого реального ручного run через Kie AI / Kling 3.0.
Цель: верифицировать ключевые adapter assumptions против реального API.
Это не production run и не scoring run. Это controlled verification exercise.

Canonical sources: ADR-0005, ADR-0006, kling-request-format.md, kling-response-format.md, task-status-mapping.md.

---

## Что проверяем

| ID | Assumption | Источник |
|----|-----------|---------|
| A1 | Поле `model` принимает значение `"kling-3.0"` | ADR-0005, kling-request-format.md |
| A2 | `input.duration = "5"` (строка, не число) принимается | ADR-0005, kling-request-format.md |
| A3 | `input.aspect_ratio = "1:1"` принимается | ADR-0005, kling-request-format.md |
| A4 | Документированные task states = `waiting` / `queuing` / `generating` / `success` | Kie AI docs, task-status-mapping.md |
| A5 | Terminal response = `{ taskId, state, resultJson, failCode, failMsg, costTime, timestamps }`, видео-данные внутри `resultJson` | Kie AI docs, kling-response-format.md |

---

## Тест-кейс

`orbit_micro_default`, `tc_01` — plate_pasta, three_quarter, 1:1, 5s.

---

## Предварительные условия

| Условие | Проверка |
|---------|---------|
| Kie AI API token активен | curl с Bearer token → не 401 |
| Start frame precomposed | `fixtures/prepped-images/plate_pasta_three_quarter_1x1.png` существует |
| Start frame доступен по URL | URL возвращает 200 с image/png |
| Run record создан (draft) | `runs/orbit_micro_default/run_orbit_micro_default_smoke_001.run-record.yaml` |

### Upstream image hosting

Допустим любой временный hosting: ngrok, presigned S3/GCS, Kie AI upload endpoint.
Зафиксировать метод в run record (`precompose_notes`).

---

## Шаг 1: Подготовить запрос

Smoke test использует **polling mode** (без callBackUrl).

```
POST {kie_api_base_url}/api/v1/jobs/createTask
Authorization: Bearer <KIEAI_API_TOKEN>
Content-Type: application/json
```

```json
{
  "model": "kling-3.0",
  "input": {
    "image_url": "<start_frame_url>",
    "prompt": "<assembled_prompt из tc_01>",
    "negative_prompt": "<negative_prompt из pp_orbit_micro_default>",
    "duration": "5",
    "mode": "pro",
    "aspect_ratio": "1:1"
  }
}
```

> Точное поле изображения (`image_url` vs `image_urls`) и model identifier — верифицировать по docs.

Зафиксировать точный JSON в observation file.

---

## Шаг 2: Отправить запрос

Записать: HTTP status code, полный response body (raw), timestamp.

**При 4xx/5xx:** записать error, не продолжать polling, `generation_status = failed`.

---

## Шаг 3: Polling

```
GET {kie_api_base_url}/api/v1/jobs/recordInfo?taskId={taskId}
Authorization: Bearer <KIEAI_API_TOKEN>
```

**Schedule:**

| Параметр | Значение |
|----------|---------|
| initial_delay | 3s |
| интервалы | 5s → 10s → 20s → 30s cap |
| max_wait | 600s |

Для каждого poll записывать: timestamp, elapsed, HTTP status, точное значение `state`, все поля.

Non-terminal: `waiting`, `queuing`, `generating` → продолжать.
Terminal success: `success` → шаг 4.
Failure state: зафиксировать, перейти к шагу 4.

---

## Шаг 4: Зафиксировать результат

**При `success`:**
- Записать полный terminal response
- Развернуть и записать всю структуру `resultJson`
- Найти URL видео, скачать немедленно (TTL неизвестен)
- Сохранить: `runs/orbit_micro_default/run_orbit_micro_default_smoke_001/output.mp4`
- Проверить разрешение и длительность видео

**При failure:**
- Записать полный response, `failCode`, `failMsg`
- Записать точный string value `state`

---

## Шаг 5: Заполнить артефакты

1. `templates/adapters/examples/orbit_micro_default.provider-task-observation.md`
2. `templates/runs/examples/orbit_micro_default.smoke-run-checklist.md`
3. `runs/orbit_micro_default/run_orbit_micro_default_smoke_001.run-record.yaml` (output секция)

---

## Шаг 6: Оценить assumptions

Verdict для каждого A1–A5: `confirmed` / `partially_confirmed` / `contradicted` / `unclear`

**При `contradicted`:**
1. Обновить spec-файл реальным значением
2. Пометить: `Corrected (smoke test, {дата}): was "{old}", actual "{new}"`

---

## Критерии smoke test success

| Критерий | Условие |
|---------|---------|
| API принял запрос | HTTP 2xx на createTask |
| Получен `taskId` | Присутствует в response |
| Наблюдены промежуточные states | Хотя бы один non-terminal |
| Получен терминальный state | `success` или failure (не timeout) |
| Все 5 assumptions оценены | Каждый имеет verdict |

> Цель — данные, а не видео. `failed` результат = valid smoke test.

---

## Что НЕ делать

- Не запускать несколько runs параллельно
- Не использовать callBackUrl в первом smoke test
- Не начинать scoring до терминального state
