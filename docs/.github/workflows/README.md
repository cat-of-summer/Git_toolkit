<!-- DOCGEN:START -->
# workflows

## Файлы

- [ci-cd.yml](ci-cd.yml.md)
- [docgen.yml](docgen.yml.md)
- [grabber.yml](grabber.yml.md)
- [minifier.yml](minifier.yml.md)
- [python-build.yml](python-build.yml.md)
- [release.yml](release.yml.md)

<!-- DOCGEN:END -->

Все автоматизированные процессы репозитория. Запускаются GitHub Actions при пуше ветки.

| Workflow | Триггер | Назначение |
|----------|--------|------------|
| [ci-cd.yml](ci-cd.yml.md) | `push`, `pull_request`, `workflow_dispatch`, `create` | CI-проверки и деплой проекта на сервер (job `get-branch` → `gate` → `ci` → `cd`) |
| [minifier.yml](minifier.yml.md) | `push` | Автоминификация CSS и JS файлов |
| [docgen.yml](docgen.yml.md) | `push`, `workflow_dispatch` | Генерация структуры документации в `docs/` |

> Все три workflow работают независимо друг от друга. Автоматически создаваемые коммиты содержат `[skip ci]` в сообщении, чтобы не триггерить рекурсивные запуски.

