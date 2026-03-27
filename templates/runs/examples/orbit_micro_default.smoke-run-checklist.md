# Smoke Run Checklist: orbit_micro_default — tc_01

## Статус

Template (Phase 4.5) — заполняется оператором в ходе первого real run

- Template: `orbit_micro_default` / Test case: `tc_01` — plate_pasta, three_quarter, 1:1, 5s
- Run ID: `run_orbit_micro_default_smoke_001`
- Дата: ________________

---

## Блок 1: Pre-run

| Проверка | Результат |
|---------|----------|
| API token активен | [ ] да / [ ] нет |
| Start frame существует | [ ] да / [ ] нет |
| Start frame доступен по URL | [ ] да / [ ] нет |
| Start frame URL | `___________________________` |
| Image hosting метод | `___________________________` |
| Prompt собран | [ ] да / [ ] нет |
| Run record создан (draft) | [ ] да / [ ] нет |

---

## Блок 2: Request

**Endpoint:** `POST {base_url}/api/v1/jobs/createTask`

```json
{
  "model": "_______________",
  "input": {
    "image_url": "_______________",
    "prompt": "...",
    "negative_prompt": "...",
    "duration": "_______________",
    "mode": "_______________",
    "aspect_ratio": "_______________"
  }
}
```

- Timestamp: `___________________________`

---

## Блок 3: Submission Response

- HTTP: `___` / Timestamp: `___________________________`

```json

```

| Поле | Значение |
|------|---------|
| `taskId` (или иное) | `___________________________` |
| Прочие поля | `___________________________` |

- [ ] `taskId` записан в run record

---

## Блок 4: Polling Log

**Endpoint:** `GET {base_url}/api/v1/jobs/recordInfo?taskId={taskId}`

| # | Timestamp | Elapsed (s) | HTTP | `state` | `resultJson` есть? | Прочее |
|---|-----------|------------|------|---------|--------------------|--------|
| 1 | | | | | | |
| 2 | | | | | | |
| 3 | | | | | | |
| 4 | | | | | | |
| 5 | | | | | | |
| 6 | | | | | | |

- Sequence: `___ → ___ → ___`
- Терминальный state: `___________________________`
- Общее время: ___ с

---

## Блок 5: Terminal Response

- HTTP: `___` / Timestamp: `___________________________`

```json

```

### Top-level field check

| Поле | Присутствует? | Значение |
|------|--------------|---------|
| `taskId` | | |
| `state` | | |
| `resultJson` | [ ] object / [ ] null | |
| `failCode` | | |
| `failMsg` | | |
| `costTime` | | |
| `timestamps` | | |

### resultJson contents (при success)

```json

```

| Поле | Значение | Примечание |
|------|---------|-----------|
| (видео URL поле — имя: ___) | | |
| (длительность — имя: ___) | | |
| (разрешение — имя: ___) | | |

---

## Блок 6: Video (при success)

| Проверка | Результат |
|---------|----------|
| Видео URL найден (имя поля: ___) | [ ] да / [ ] нет |
| Видео доступен (HTTP 200) | [ ] да / [ ] нет |
| Скачан | [ ] да / [ ] нет |
| Размер (bytes) | `___________________________` |
| Разрешение | `___________________________` |
| Длительность | `___________________________` |
| Aspect ratio 1:1? | [ ] да / [ ] нет (реальное: ___) |

---

## Блок 7: Assumption Verification

| ID | Assumption | Verdict | Комментарий |
|----|-----------|---------|------------|
| A1 | `model = "kling-3.0"` принят | [ ] confirmed / [ ] contradicted / [ ] unclear | model id: ___ |
| A2 | `input.duration = "5"` (строка) принят | [ ] confirmed / [ ] contradicted / [ ] unclear | |
| A3 | `input.aspect_ratio = "1:1"` принят | [ ] confirmed / [ ] contradicted / [ ] unclear | |
| A4 | States = waiting/queuing/generating/success | [ ] confirmed / [ ] partially / [ ] contradicted | Observed: ___ |
| A5 | Response shape + resultJson structure | [ ] confirmed / [ ] partially / [ ] contradicted | Расхождения: ___ |

**Дополнительно наблюдённое:**

```
___________________________________________________________
```

---

## Блок 8: Post-run

- [ ] observation file заполнен
- [ ] run record обновлён (output)
- [ ] ADR-0006 Findings заполнены
- [ ] Spec-файлы обновлены при необходимости

---

## Итог

- Smoke test success? [ ] да / [ ] нет
- Критические расхождения? [ ] нет / [ ] да: `___________________________`
