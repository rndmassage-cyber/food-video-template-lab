# Runbook: Bootstrap проекта

## Назначение

Пошаговая инструкция для первоначальной настройки рабочего окружения проекта food-video-template-lab.

## Предусловия

- Git установлен
- Доступ к репозиторию
- Claude Code установлен и настроен

## Шаги

### Шаг 1. Клонировать репозиторий

```bash
git clone <repo-url>
cd food-video-template-lab
```

### Шаг 2. Проверить структуру проекта

Убедиться, что присутствуют основные папки:

- `docs/` — документация, ADR, runbooks
- `templates/` — specs, prompt-packs, test-suites
- `fixtures/` — raw-inputs, prepped-images, reference-outputs
- `runs/` — результаты запусков
- `scripts/` — служебные скрипты

### Шаг 3. Ознакомиться с контекстом проекта

При начале работы с проектом ориентироваться на следующие документы (в этом порядке):

1. `README.md` — общее описание проекта
2. `CLAUDE.md` — проектные инструкции и ограничения V1
3. `docs/roadmap.md` — дорожная карта
4. `docs/project-map.md` — карта структуры проекта

### Шаг 4. Проверить настройки Claude Code

Permissions и hooks для Claude Code настроены в `.claude/settings.json`. Локальные overrides при необходимости добавляются в `.claude/settings.local.json` (не коммитится).

### Шаг 5. Проверить текущий статус проекта

Текущая фаза проекта указана в `docs/roadmap.md` (раздел «Текущий статус»).
