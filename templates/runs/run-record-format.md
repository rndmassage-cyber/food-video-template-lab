# Run Record Format — food-video-template-lab

## Статус

Proposed (Phase 2, batch 3)

## Дата

2026-03-25

## Назначение

Определяет формат YAML-файлов run record для V1.

Run record — полная запись единичного запуска генерации видео.
Фиксирует входные параметры, собранный prompt, API-метаданные,
результат генерации и оценку (score).

Canonical sources: ADR-0003 (И-3: run = atomic execution unit, И-4: score → run),
domain-model.md (сущности run, output_artifact, score).

---

## Принципы

1. **Атомарность.** Один run record = один вызов Kling API. Неделим.
2. **Полнота входов.** Все параметры, использованные для генерации, сохраняются в record.
   Run record должен содержать достаточно информации для воспроизведения запроса.
3. **Score встроен.** В V1 score не выносится в отдельный файл, а хранится как секция
   внутри run record. Это упрощает работу при ручном scoring.
   Концептуально score остаётся отдельной сущностью (domain-model.md, И-4).
4. **Failed runs сохраняются.** Run record создаётся даже при ошибке генерации.
   Request metadata и error details сохраняются (ADR-0003, И-3).

---

## Расположение файлов

- Формат и примеры: `templates/runs/`
- Фактические run records: `runs/{template_id}/{run_id}.run-record.yaml`

Директория `runs/` на корневом уровне проекта содержит фактические данные.
Видеофайлы и превью хранятся рядом с run record или в поддиректории.

---

## Именование

### run_id

Формат: `run_{template_id}_{YYYYMMDD}_{seq}`

- `template_id` — ID шаблона
- `YYYYMMDD` — дата запуска
- `seq` — 3-значный порядковый номер в рамках дня (001, 002, ...)

Пример: `run_orbit_micro_default_20260325_001`

### Имя файла

`{run_id}.run-record.yaml`

---

## Структура файла

### Секция: Идентификация

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `run_id` | string | yes | Уникальный идентификатор run |
| `template_id` | string | yes | Какой template использовался |
| `template_version` | integer | yes | Версия template на момент run (proposed, см. примечание) |
| `prompt_pack_id` | string | yes | Какой prompt pack использовался |
| `prompt_pack_version` | integer | yes | Версия prompt pack на момент run |
| `test_suite_id` | string | no | Ссылка на test suite (null для ad-hoc runs) |
| `case_id` | string | no | Ссылка на test case внутри suite (null для ad-hoc runs) |

**Примечание по template_version / prompt_pack_version:**
Открытый вопрос из template-schema-notes.md (#3): как runs ссылаются на версию template.
Proposed решение для V1: хранить номер версии. При расхождении — git history.
Полная versioning policy определяется отдельно.

### Секция: Входные параметры

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `source_image_ref` | string | yes | Ссылка на исходное изображение |
| `angle_family` | string | yes | Ракурс: top_down, three_quarter, hero_side |
| `aspect_ratio` | string | yes | 1:1, 9:16, 16:9 |
| `duration` | number | yes | Запрошенная длительность в секундах |

### Секция: Собранный prompt

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `assembled_prompt` | string | yes | Финальный motion prompt после применения модификаторов |
| `negative_prompt` | string | yes | Negative prompt as-is |

Хранение собранного prompt обеспечивает воспроизводимость:
даже если prompt pack изменится, run record содержит точный prompt, использованный при генерации.

### Секция: Precompose

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `start_frame_ref` | string | yes | Ссылка на подготовленный start frame (path или ID) |
| `precompose_notes` | string | no | Заметки о precompose (если были отклонения от rules) |

Формат start_frame_ref уточняется в Phase 3.
До Phase 3 допустим placeholder (например, path к файлу).

### Секция: API

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `api_provider` | string | yes | Провайдер: `kie_ai` (V1) |
| `api_model` | string | yes | Модель: `kling_3.0` (V1) |
| `api_mode` | string | yes | Режим: `pro` (V1) |
| `api_task_id` | string | no | Task ID от Kling API (null если запрос не отправлен) |
| `api_submitted_at` | string | no | ISO 8601 время отправки запроса |
| `api_completed_at` | string | no | ISO 8601 время получения результата |
| `api_error` | string | no | Текст ошибки (если генерация не удалась) |

### Секция: Output

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `generation_status` | string | yes | `success`, `failed`, `timeout` |
| `video_url` | string | no | URL видео от Kling API (если success) |
| `video_file_path` | string | no | Локальный путь к скачанному видео |
| `thumbnail_path` | string | no | Путь к превью |
| `duration_actual` | number | no | Фактическая длительность видео в секундах |
| `resolution` | string | no | Разрешение (например "1080x1080") |
| `file_size_bytes` | integer | no | Размер видеофайла |

### Секция: Score (embedded)

Score — опциональная секция. Отсутствует, если run ещё не оценён.

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `evaluator` | string | yes* | Кто оценивал: `human`, `auto_rule`, `llm_judge` |
| `criteria_results` | list[object] | yes* | Результаты по критериям (см. ниже) |
| `overall_verdict` | string | yes* | `pass`, `fail`, `marginal` |
| `score_notes` | string | no | Свободные комментарии оценщика |
| `scored_at` | string | yes* | ISO 8601 время оценки |

*Required если секция score присутствует.

**Структура criteria_result:**

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `criterion_id` | string | yes | ID критерия из test_suite.scoring_criteria |
| `score` | integer (1–5) | yes | Оценка по шкале scoring rubric |
| `notes` | string | no | Комментарий к оценке |

Методика оценки и шкала определяются в `scoring-rubric-v1.md`.

### Секция: Timestamps

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `created_at` | string | yes | ISO 8601 дата создания записи |
| `updated_at` | string | yes | ISO 8601 дата последнего обновления |

---

## Связь с другими сущностями

```
test_suite
  └── test_case
        └── run record (один test case → один run)
              ├── assembled_prompt (из prompt_pack + modifiers)
              ├── start_frame_ref (из precompose pipeline)
              ├── output (video, metadata)
              └── score (embedded, оценка по scoring_criteria из test_suite)
```

- **test_suite → run:** test case определяет параметры run.
- **run → score:** score оценивает конкретный run (ADR-0003, И-4).
- **run → output_artifact:** успешный run порождает видео + метаданные.
- **ad-hoc runs:** run без test_suite_id / case_id — допустим для exploratory runs.

---

## Что run record НЕ содержит

| Исключённое | Почему |
|---|---|
| Полный template spec | Избыточно — ссылка через template_id + template_version |
| Полный prompt pack | Избыточно — assembled_prompt + negative_prompt достаточно |
| Scoring rubric | Это scoring-rubric-v1.md |
| Сравнение с другими runs | Это аналитическая операция, не данные |
| Billing / cost data | За рамками V1 |

---

## Рекомендации

1. **Создавать record сразу.** Run record создаётся при отправке запроса,
   не после получения результата. Output и score дозаполняются позже.
2. **Сохранять failed runs.** Ошибки — ценные данные для анализа стабильности.
3. **Фиксировать assembled_prompt.** Не полагаться на пересборку из prompt pack —
   prompt pack мог измениться.
4. **Scoring — отдельный шаг.** Score заполняется после просмотра видео.
   Между генерацией и scoring может пройти время.

---

## Открытые вопросы

| # | Вопрос | Когда решается |
|---|--------|----------------|
| 1 | Точный формат start_frame_ref | Phase 3 |
| 2 | Нужен ли snapshot полного template spec в run record или достаточно version? | Phase 4–5 |
| 3 | Хранить ли video файлы рядом с run record или в отдельной директории? | Phase 4 |
| 4 | Формат api_task_id и маппинг на конкретные поля Kie AI API response | Phase 4 |
| 5 | Автоматическое создание run records из test suite | Phase 6+ |
