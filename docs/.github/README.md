# How to work with workflows and docs

Краткое руководство по работе с workflow и системой документации в репозитории.

1) Где находятся workflow
- `.github/workflows/` — исходные YAML-файлы, которые выполняются GitHub Actions.
- Детализированные версии workflow для чтения находятся в `docs/.github/workflows/` (MD-файлы).

2) Обновление документации
- Для авто-генерируемых записей используйте `docgen.yml`. При первом запуске он создаёт `docs/.docignore` и начальную структуру `docs/`.

3) Как вносить изменения в workflow
- Рекомендуется:
  - создать ветку `feature/docs-update` или `feature/ci-tweak`;
  - внести изменения в `.github/workflows/*.yml`;
  - обновить соответствующий `docs/.github/workflows/*.md` вручную (если нужно) или запустить `docgen.yml`.

4) Локальная проверка и отладка
- Локально вы можете просмотреть YAML и проверить команды shell.
- Для тестов workflow используйте тестовые ветки и ручные `workflow_dispatch` запуски.

5) Полезные ссылки
- Подробные страницы workflow: `docs/.github/workflows/`.
- Правила автогенерации: `docs/.docignore`.
<!-- DOCGEN:START -->
# .github

## Папки

- [workflows](workflows/)

<!-- DOCGEN:END -->

Стандартная папка GitHub. Содержит настройки GitHub Actions для автоматизации CI/CD процессов.

## Содержимое

| Папка | Назначение |
|--------|------------|
| `workflows/` | GitHub Actions workflow-файлы. Каждый `.yml` — отдельный автоматизированный процесс |
