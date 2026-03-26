# Run Execution Workflow — food-video-template-lab

## Статус

Proposed (Phase 4, batch 1)

## Дата

2026-03-25

## Назначение

Пошаговый runbook для выполнения одного run в V1.
V1: все шаги выполняются оператором вручную. Автоматизация — за рамками V1.

> **Примечание по API.** Конкретные endpoint paths, field names, model identifiers
> должны сверяться с актуальной Kie AI документацией перед первым run.
> Этот runbook описывает workflow — не точные CLI команды или HTTP запросы.

Canonical sources: ADR-0003, ADR-0004, ADR-0005, run-record-format, prompt-pack-format, precompose-format.

---

## Обзор workflow

```
[1]  Определить параметры run
[2]  Валидировать параметры
[3]  Подготовить start frame (precompose)
[4]  Обеспечить доступность start frame по URL
[5]  Собрать prompt
[6]  Создать run record (draft)
[7]  Отправить запрос в Kling API
[8]  Дождаться результата (callBackUrl или polling)
[9]  Скачать и сохранить видео
[10] Заполнить run record (output)
[11] Оценить результат (scoring)
[12] Финализировать run record
```

---

## Шаг 1: Определить параметры run

**Из test case (structured run):**

| Параметр | Источник |
|----------|----------|
| `template_id` | test_suite.template_id |
| `source_image_ref` | test_case.source_image_ref |
| `angle_family` | test_case.angle_family |
| `aspect_ratio` | test_case.aspect_ratio |
| `duration` | test_case.duration |
| `test_suite_id` | test_suite.test_suite_id |
| `case_id` | test_case.case_id |

**Ad-hoc run:** параметры выбираются оператором вручную. `test_suite_id` и `case_id` = null.

---

## Шаг 2: Валидировать параметры

- [ ] `angle_family` ∈ template.supported_angles
- [ ] `aspect_ratio` ∈ template.supported_aspect_ratios
- [ ] `duration` ∈ template.supported_durations
- [ ] Source image существует: `fixtures/raw-inputs/{...}`
- [ ] template.status ≠ `archived`

Если любая проверка не пройдена — run не выполняется.

---

## Шаг 3: Подготовить start frame (precompose)

**Проверить кэш:**

```
fixtures/prepped-images/{source_name}_{aspect_shortcode}.png
```

Если файл существует и precompose spec не менялся с момента создания — переиспользовать (ADR-0004, П-8).

**Если нужен новый precompose:**

1. Открыть: `templates/precompose/{template_id}.precompose.yaml`
2. Найти canvas spec для целевого aspect_ratio
3. Создать canvas целевого разрешения (ADR-0004, П-5):

   | Aspect ratio | Canvas |
   |---|---|
   | 1:1 | 1080×1080 |
   | 9:16 | 1080×1920 |
   | 16:9 | 1920×1080 |

4. Применить safe_area_margin (из canvas spec или defaults), разместить блюдо в центре
5. Заполнить фон: `source_bg` (edge sampling) или `solid_color`
6. Сохранить: `fixtures/prepped-images/{source_name}_{aspect_shortcode}.png`

`aspect_shortcode`: `1x1`, `9x16`, `16x9`

---

## Шаг 4: Обеспечить доступность start frame по URL

Kling API принимает изображение через `image_urls` (список URL).
Precomposed файл должен быть доступен по HTTP URL до отправки запроса.

**V1: upstream upload workflow TBD.** Допустимые варианты (на усмотрение оператора):
- Временный upload в cloud storage (S3, GCS и т.п.)
- Kie AI upload endpoint (если поддерживается — проверить docs)
- Локальный HTTP сервер (только для dev/local runs)

Зафиксировать полученный `start_frame_url`.

---

## Шаг 5: Собрать prompt

1. Открыть: `templates/prompt-packs/{prompt_pack_id}.prompt-pack.yaml`
2. Взять `motion_prompt` (base)
3. Добавить `prompt_modifiers.by_angle.{angle_family}.append` (если непустой)
4. Добавить `prompt_modifiers.by_aspect_ratio.{aspect_ratio}.append` (если непустой)
5. Взять `negative_prompt` as-is (без modifiers)

Результат: `assembled_prompt` и `negative_prompt`.

---

## Шаг 6: Создать run record (draft)

1. Определить `run_id`: `run_{template_id}_{YYYYMMDD}_{seq}`
2. Создать директорию: `runs/{template_id}/`
3. Создать файл: `runs/{template_id}/{run_id}.run-record.yaml`
4. Заполнить все секции, кроме output и score (API-поля пока null)
5. `generation_status` = pending
6. `created_at` = now

**Важно:** run record создаётся **до** отправки запроса.
Это гарантирует, что параметры зафиксированы даже при сбое API call.

---

## Шаг 7: Отправить запрос в Kling API

Сформировать request согласно `templates/adapters/kling-request-format.md`:

```
POST {kie_api_base_url}/v1/videos/image2video   ← endpoint — верифицировать по Kie docs
Authorization: Bearer <KIEAI_API_TOKEN>
Content-Type: application/json

{
  "model_name":      "kling-3.0",              ← proposed, верифицировать
  "image_urls":      ["<start_frame_url>"],
  "prompt":          "<assembled_prompt>",
  "negative_prompt": "<negative_prompt>",
  "duration":        "<duration>",              ← строка: "5" или "10"
  "mode":            "pro",
  "aspect_ratio":    "<aspect_ratio>",          ← "1:1", "9:16", "16:9" (proposed format)
  "callback_url":    "<callback_url>"           ← production; опустить для polling mode
}
```

Зафиксировать в run record: `api_task_id` (из response), `api_submitted_at` (timestamp).

---

## Шаг 8: Дождаться результата

### Production — callBackUrl (preferred)

Kling API отправит POST на `callback_url` при завершении задачи.
Обработать входящий callback, извлечь `video_url` и `status`.

### Dev / local / manual — polling (допустимо)

```
GET {kie_api_base_url}/v1/videos/tasks/{task_id}
Authorization: Bearer <KIEAI_API_TOKEN>
```

**Polling schedule (proposed):**

| Параметр | Значение |
|----------|----------|
| initial_delay | 2–3s |
| backoff | exponential: 3s → 6s → 12s → 24s → 30s cap |
| max_wait | 600s (10 мин) → `timeout` |

**Статусы:**
- `pending` / `processing` → продолжить
- `completed` → перейти к шагу 9
- `failed` → зафиксировать error, перейти к шагу 10

---

## Шаг 9: Скачать и сохранить видео

При `generation_status = success`:

1. Скачать видео из `video_url`
2. Сохранить: `runs/{template_id}/{run_id}/output.mp4`
3. (Опционально) thumbnail: `runs/{template_id}/{run_id}/thumb.jpg`
4. Зафиксировать: `duration_actual`, `resolution`, `file_size_bytes`

---

## Шаг 10: Заполнить run record (output)

Обновить run record:

**Секция API:**
- `api_completed_at` — timestamp финального статуса
- `api_error` — текст ошибки или null

**Секция Output:**
- `generation_status` — success / failed / timeout
- `video_url`, `video_file_path`, `thumbnail_path`
- `duration_actual`, `resolution`, `file_size_bytes`

**Timestamps:**
- `updated_at` = now

---

## Шаг 11: Оценить результат (scoring)

Human scoring (V1):

1. Открыть: `templates/scoring/scoring-rubric-v1.md`
2. Просмотреть видео
3. Оценить 5 критериев (1–5):
   - dish_integrity (weight: critical)
   - motion_stability (weight: critical)
   - motion_intent_match (weight: high)
   - background_stability (weight: high)
   - no_artifacts (weight: normal)
4. Определить overall_verdict:
   - **fail**: любой critical ≤ 2 ИЛИ любой high = 1
   - **pass**: все critical ≥ 3, все high ≥ 3, weighted_avg ≥ 3.5
   - **marginal**: все critical ≥ 3, все ≥ 2, weighted_avg ≥ 2.5
   - **fail** (default): всё остальное

---

## Шаг 12: Финализировать run record

Добавить секцию `score`:

```yaml
score:
  evaluator: human
  criteria_results:
    - criterion_id: dish_integrity
      score: <1-5>
      notes: "<заметки>"
    - criterion_id: motion_stability
      score: <1-5>
      notes: "<заметки>"
    - criterion_id: motion_intent_match
      score: <1-5>
      notes: "<заметки>"
    - criterion_id: background_stability
      score: <1-5>
      notes: "<заметки>"
    - criterion_id: no_artifacts
      score: <1-5>
      notes: "<заметки>"
  overall_verdict: <pass|fail|marginal>
  score_notes: "<общие наблюдения>"
  scored_at: "<ISO 8601>"
```

Обновить `updated_at`.

---

## Полный чеклист run

```
Pre-run:
  [ ] Параметры определены (template, image, angle, ratio, duration)
  [ ] Параметры валидированы (∈ supported_*)
  [ ] Start frame precomposed
  [ ] Start frame доступен по URL
  [ ] Prompt собран (base + modifiers)
  [ ] Run record создан (draft)

Execution:
  [ ] Запрос отправлен, task_id получен и записан
  [ ] Результат получен (callBackUrl или polling)
  [ ] Видео скачано (если success)

Post-run:
  [ ] Run record обновлён (output секция)
  [ ] Scoring выполнен
  [ ] Run record финализирован
```

---

## Пример trace: orbit_micro_default, tc_01

| Шаг | Значение |
|-----|----------|
| template_id | orbit_micro_default |
| source_image_ref | fixtures/raw-inputs/plate_pasta_three_quarter.png |
| angle_family | three_quarter |
| aspect_ratio | 1:1 |
| duration | 5 |
| start_frame (precomposed) | fixtures/prepped-images/plate_pasta_three_quarter_1x1.png |
| canvas | 1080×1080, margin 10%, source_bg |
| assembled_prompt | base + three_quarter modifier |
| API aspect_ratio | "1:1" |
| run_id | run_orbit_micro_default_20260325_001 |
| result_retrieval | callBackUrl (production) или polling (dev) |
| request example | templates/adapters/examples/orbit_micro_default.kling-request.json |

---

## Особые случаи

**Failed run:** не удалять record, заполнить `api_error`, score не выставляется.

**Timeout:** проверить task статус вручную позже. Если задача завершилась — обновить record вручную.

**Повторный run с теми же параметрами:** новый `run_id` (следующий seq).
Предыдущий run record не модифицируется (ADR-0003, И-3).

---

## Открытые вопросы

| # | Вопрос | Когда |
|---|--------|-------|
| 1 | Upstream image upload workflow | До первого smoke test |
| 2 | Автоматизация шагов 3–9 | Phase 6+ |
| 3 | Batch execution (весь test suite за один запуск) | Phase 6+ |
| 4 | Pre-flight check (API quota, availability) перед submit | Phase 6 |
| 5 | Хранить video files рядом с run record или в отдельной media dir? | Phase 5 |
