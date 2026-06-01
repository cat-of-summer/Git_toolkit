<!-- DOCGEN:START -->
# workflows

## Папки

- [ci_cd](ci_cd/)

## Файлы

- [docgen.yml](docgen.yml.md)
- [minifier.yml](minifier.yml.md)
- [python-build.yml](python-build.yml.md)

<!-- DOCGEN:END -->

Все автоматизированные процессы репозитория. Запускаются GitHub Actions при пуше ветки.

| Workflow | Триггер | Назначение |
|----------|--------|------------|
| [cd.yml](cd.yml.md) | `push`, `pull_request`, `workflow_dispatch` | CI-проверки и деплой проекта на сервер (job `ci` → `cd`) |
| [minifier.yml](minifier.yml.md) | `push` | Автоминификация CSS и JS файлов |
| [docgen.yml](docgen.yml.md) | `push`, `workflow_dispatch` | Генерация структуры документации в `docs/` |

> Все три workflow работают независимо друг от друга. Автоматически создаваемые коммиты содержат `[skip ci]` в сообщении, чтобы не триггерить рекурсивные запуски.

