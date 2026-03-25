# Prompt Pack Format — food-video-template-lab

## Статус

Proposed (Phase 2, batch 1)

## Дата

2026-03-25

## Назначение

Определяет формат YAML-файлов prompt pack для V1.

Prompt pack — набор prompt-конфигураций, привязанный к конкретному template.
Содержит всё необходимое для формирования prompt к Kling 3.0 API.

Canonical sources: ADR-0002 (п.13), ADR-0003, domain-model.md (сущность prompt_pack).

---

## Принципы

1. **1:1 с template в V1.** Один prompt pack — ровно один template.
2. **Декларативный.** Prompt pack описывает _что_ подавать в API, а не _как_ вызывать.
3. **Модульный.** Базовый prompt + условные модификаторы по angle и aspect_ratio.
4. **Стабильность важнее креативности.** Формулировки оптимизируются на повторяемость.

---

## Структура файла

Файл: `templates/prompt-packs/{prompt_pack_id}.prompt-pack.yaml`

### Обязательные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `prompt_pack_id` | string | Уникальный идентификатор. Формат: `pp_{template_id}` |
| `template_id` | string | Обратная ссылка на template |
| `version` | integer | Версия prompt pack (≥ 1) |
| `motion_prompt` | string | Основной prompt движения — базовый текст, подаваемый в Kling |
| `negative_prompt` | string | Anti-drift guidance — что Kling НЕ должен делать |

### Опциональные поля

| Поле | Тип | Описание |
|------|-----|----------|
| `prompt_modifiers` | object | Условные модификации prompt (см. ниже) |
| `safe_patterns` | list[string] | Формулировки, подтверждённые как стабильные |
| `unsafe_patterns` | list[string] | Формулировки, дающие нестабильный результат |
| `stability_notes` | string | Свободные заметки о стабильности |
| `created_at` | string | ISO 8601 |
| `updated_at` | string | ISO 8601 |

---

## Prompt modifiers

Модификаторы позволяют изменять prompt в зависимости от условий генерации.

### Структура

```yaml
prompt_modifiers:
  by_angle:
    top_down:
      append: "viewed from directly above"
    three_quarter:
      append: "viewed from a three-quarter angle"
    hero_side:
      append: "viewed from a low side angle"
  by_aspect_ratio:
    "9:16":
      append: "vertical composition"
    "16:9":
      append: "horizontal wide composition"
    "1:1":
      append: ""  # no modification needed
```

### Правила применения

1. `motion_prompt` — всегда используется как базовый текст.
2. Если для текущих `angle_family` и `aspect_ratio` есть modifier —
   его `append` добавляется к motion_prompt через разделитель `, `.
3. `negative_prompt` подаётся в API as-is, без модификаций.
4. Порядок применения: base → angle modifier → aspect_ratio modifier.

### Пример сборки

Входные данные:
- template: `orbit_micro_default`
- angle: `three_quarter`
- aspect_ratio: `9:16`

Prompt pack:
- motion_prompt: `"Subtle smooth orbital camera rotation around the dish"`
- angle modifier (three_quarter): `"viewed from a three-quarter angle"`
- aspect_ratio modifier (9:16): `"vertical composition"`

Результат:
```
Subtle smooth orbital camera rotation around the dish, viewed from a three-quarter angle, vertical composition
```

---

## Связь с template

- Template spec содержит `prompt_pack_id` → указывает на файл prompt pack.
- Prompt pack содержит `template_id` → обратная ссылка.
- Версии template и prompt pack инкрементируются независимо.
  Изменение prompt pack может требовать bump версии template,
  если оно влияет на template как единицу тестирования.
  Окончательная versioning policy определяется отдельно.

---

## Что prompt pack НЕ содержит

| Исключённое | Почему |
|---|---|
| API endpoint / credentials | Это adapter layer (Phase 4) |
| Параметры генерации (mode, duration, resolution) | Это template spec + run params |
| Precompose instructions | Это template.precompose_rules (Phase 3) |
| Scoring criteria | Это test_suite (Phase 5) |
| Результаты генерации | Это run → output_artifact |

---

## Рекомендации по написанию prompt

Эти рекомендации — рабочие гипотезы. Подлежат валидации в Phase 6.

1. **Конкретность.** Описывай движение конкретно: «slow 10-degree orbital rotation»,
   а не «interesting camera movement».
2. **Ограничение scope.** Описывай только движение камеры/объекта.
   Не описывай содержимое блюда, сервировку, фон.
3. **Negative prompt.** Явно исключай нежелательное: morphing, deformation,
   new objects appearing, background changes.
4. **Стабильные конструкции.** Предпочитай формулировки из `safe_patterns`.
   Избегай формулировок из `unsafe_patterns`.
5. **Минимализм.** Короткий prompt обычно стабильнее длинного.
