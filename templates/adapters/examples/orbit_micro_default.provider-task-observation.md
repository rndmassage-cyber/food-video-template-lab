# Provider Task Observation: orbit_micro_default — Smoke Run

## Статус

Template (Phase 4.5) — заполняется оператором в ходе первого real run

Связанные файлы:
- `templates/adapters/kling-request-format.md`
- `templates/adapters/kling-response-format.md`
- `templates/adapters/task-status-mapping.md`
- `runs/orbit_micro_default/run_orbit_micro_default_smoke_001.run-record.yaml`

---

## Контекст

| Поле | Значение |
|------|---------|
| Дата | |
| Оператор | |
| API token (masked) | `kie_***...***` |
| API base URL | |
| Image hosting | |
| Start frame URL | |

---

## 1. Raw Request

```
POST _______________________________________________
Authorization: Bearer <masked>
Content-Type: application/json
```

```json

```

---

## 2. Submission Response

```
HTTP: ___     Timestamp: ___
```

```json

```

| Аспект | Наблюдение | Notes |
|--------|-----------|-------|
| HTTP status | | |
| taskId field name (exact) | | Ожидалось: `taskId` |
| taskId value | | |
| Прочие поля | | |

---

## 3. Polling Log

**Endpoint:** `GET {base_url}/api/v1/jobs/recordInfo?taskId={taskId}`

### Poll 1

```
Timestamp: ___     Elapsed: ___s     HTTP: ___
state: "___"
Body:
```

### Poll 2

```
Timestamp: ___     Elapsed: ___s     HTTP: ___
state: "___"
Body:
```

### Poll 3

```
Timestamp: ___     Elapsed: ___s     HTTP: ___
state: "___"
Body:
```

_(добавить по необходимости)_

### Summary

| Параметр | Наблюдение |
|---------|-----------|
| Observed sequence | `___ → ___ → ___` |
| Expected (from docs) | `waiting → queuing → generating → success` |
| Time to first non-waiting | ___s |
| Time to terminal | ___s |
| Total polls | ___ |
| Неожиданные states | |

---

## 4. Terminal Response

```
HTTP: ___     Timestamp: ___
```

```json

```

### Top-level: spec vs actual

| Spec field | В response? | Actual name | Actual value | Match? |
|-----------|------------|------------|-------------|-------|
| `taskId` | | | | |
| `state` | | | | |
| `resultJson` | | | | |
| `failCode` | | | | |
| `failMsg` | | | | |
| `costTime` | | | | |
| `timestamps` | | | | |

**Поля не из spec:**

| Field | Value |
|-------|-------|
| | |

### resultJson (при success)

```json

```

| Field name | Value | Type |
|-----------|-------|------|
| (video url) | | |
| (duration) | | |
| (resolution) | | |
| (прочие) | | |

---

## 5. Assumption Verdicts

| ID | Assumption | Verdict | Evidence |
|----|-----------|---------|---------|
| A1 | `model` field + value | `confirmed` / `contradicted` / `unclear` | |
| A2 | `input.duration = "5"` (string) | `confirmed` / `contradicted` / `unclear` | |
| A3 | `input.aspect_ratio = "1:1"` | `confirmed` / `contradicted` / `unclear` | |
| A4 | States = waiting/queuing/generating/success | `confirmed` / `partially_confirmed` / `contradicted` | |
| A5 | Terminal shape + resultJson | `confirmed` / `partially_confirmed` / `contradicted` | |

---

## 6. Spec Updates Required

| Файл | Изменение | Приоритет |
|------|----------|---------|
| | | |

---

## 7. Free-form Observations

```
___________________________________________________________
```
