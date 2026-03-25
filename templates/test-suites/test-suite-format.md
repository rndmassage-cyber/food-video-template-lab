# Test Suite Format — food-video-template-lab

## Статус

Proposed (Phase 2, batch 2)

## Дата

2026-03-25

## Назначение

Определяет формат YAML-файлов test suite для V1.

Test suite — набор тест-кейсов для систематической проверки template.
Определяет, на каких входах и при каких условиях template должен быть протестирован,
и по каким критериям оцениваются результаты.

Canonical sources: ADR-0003 (И-4, scoring_criteria в test_suite), domain-model.md (сущность test_suite).

---

## Принципы

1. **Привязан к template.** Каждый test suite тестирует ровно один template.
2. **Явный список test cases.** Каждый кейс — конкретная комбинация параметров. Нет неявной генерации.
3. **Scoring criteria — здесь, не в template.** Template содержит scoring_hints, test suite — формальные критерии оценки (ADR-0003, И-4).
4. **Representative, не exhaustive.** Не обязательно покрывать все комбинации. Достаточно representative subset для принятия решения о статусе template.

---

## Структура файла

Файл: `templates/test-suites/{template_id}.test-suite.yaml`

### Обязательные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `test_suite_id` | string | Уникальный идентификатор. Формат: `ts_{template_id}` |
| `template_id` | string | Ссылка на template |
| `version` | integer | Версия test suite (≥ 1) |
| `test_cases` | list[object] | Список тест-кейсов (см. ниже) |
| `scoring_criteria` | list[object] | Критерии оценки (см. ниже) |

### Опциональные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `min_pass_rate` | number (0.0–1.0) | Минимальная доля passed runs для принятия template. Proposed default: 0.7 |
| `notes` | string | Свободные заметки |
| `created_at` | string | ISO 8601 |
| `updated_at` | string | ISO 8601 |

---

## Структура test case

Каждый test case описывает один планируемый run.

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `case_id` | string | yes | Уникальный в рамках suite (например `tc_01`) |
| `source_image_ref` | string | yes | Ссылка на исходное изображение. Формат: `fixture:{name}` (до Phase 3 — placeholder) |
| `angle_family` | string | yes | Ракурс: `top_down`, `three_quarter`, `hero_side` |
| `aspect_ratio` | string | yes | `1:1`, `9:16`, `16:9` |
| `duration` | number | yes | Длительность в секундах |
| `notes` | string | no | Зачем этот кейс включён |

### Валидация test case

- `angle_family` ∈ `template.supported_angles`
- `aspect_ratio` ∈ `template.supported_aspect_ratios`
- `duration` ∈ `template.supported_durations`

Если test case содержит значение за пределами supported-списков template,
это ошибка в test suite, а не расширение template.

---

## Структура scoring criteria

Scoring criteria определяют, по каким осям оценивается каждый run.
Полная scoring rubric (шкалы, пороги, методы оценки) определяется в Phase 5.
Здесь фиксируется только структура и набор критериев.

| Поле | Тип | Required | Описание |
|------|-----|----------|----------|
| `criterion_id` | string | yes | Уникальный идентификатор критерия |
| `name` | string | yes | Человекочитаемое название |
| `description` | string | yes | Что именно оценивается |
| `weight` | string | no | Relative importance: `critical`, `high`, `normal`. Default: `normal` |

### Стандартные критерии V1 (proposed)

Эти критерии — рабочий набор, предложенный на основе domain-model.md.
Конкретный test suite может использовать подмножество или добавлять template-specific критерии.

| criterion_id | Название | Описание |
|---|---|---|
| `dish_integrity` | Сохранность формы блюда | Блюдо сохраняет форму, пропорции и детали на протяжении всего видео |
| `motion_stability` | Стабильность движения | Движение плавное, без рывков, скачков и внезапных изменений |
| `motion_intent_match` | Соответствие motion intent | Фактическое движение совпадает с описанным в template.motion_intent |
| `background_stability` | Чистота фона | Фон однотонный, без появления новых объектов, текстур, теней |
| `no_artifacts` | Отсутствие артефактов | Нет визуальных глитчей, размытий, дублирования элементов |

---

## Связь с другими сущностями

```
template
  ├── prompt_pack (1:1, определяет prompt)
  └── test_suite (1:N, определяет программу испытаний)
        └── test_case → при выполнении порождает run
              run → output_artifact (результат)
              run → score (оценка по scoring_criteria)
```

- **test_suite → template:** каждый suite привязан к одному template.
- **test_case → run:** каждый выполненный test case порождает один run.
- **scoring_criteria → score:** score оценивает run по критериям из его test_suite.
- **min_pass_rate:** доля runs с verdict=pass, необходимая для положительного решения о template.

---

## Что test suite НЕ содержит

| Исключённое | Почему |
|---|---|
| Prompt-тексты | Это prompt_pack |
| Precompose-параметры | Это template.precompose_rules |
| API-конфигурация | Это adapter layer (Phase 4) |
| Результаты runs | Это run → output_artifact (в runs/) |
| Score values | Это score (привязан к run, не к test suite) |
| Автоматизация запуска | Phase 6+ |

---

## Рекомендации по составлению test suite

1. **Покрывай ключевые оси.** Каждый supported_angle и supported_aspect_ratio
   должен присутствовать хотя бы в одном test case.
2. **Начинай с минимума.** 6–10 test cases для стартового suite.
   Расширяй по результатам первых runs.
3. **Разные блюда.** Используй fixture-изображения с разной геометрией:
   плоское блюдо (паста, салат) и объёмное (бургер, стопка).
4. **Помечай «зачем».** Поле `notes` в test case помогает понять,
   что именно проверяет этот кейс.
