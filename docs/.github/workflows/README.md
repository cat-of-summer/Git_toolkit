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
| [deploy.yml](deploy.yml.md) | `push` | Сборка и деплой проекта на сервер |
| [minifier.yml](minifier.yml.md) | `push` | Автоминификация CSS и JS файлов |
| [docgen.yml](docgen.yml.md) | `push`, `workflow_dispatch` | Генерация структуры документации в `docs/` |

> Все три workflow работают независимо друг от друга. Автоматически создаваемые коммиты содержат `[skip ci]` в сообщении, чтобы не триггерить рекурсивные запуски.

