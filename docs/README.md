<!-- DOCGEN:START -->
# GitHub_workflows

## Папки

- [.github](.github/)
<!-- DOCGEN:START -->
# GitHub Workflows — обзор документации

Этот раздел содержит краткий обзор доступных workflow и инструкции по работе с ними. Подробная документация каждого workflow находится в `docs/.github/workflows/`.

Содержание:

- `ci-cd.yml` — универсальный деплой (FTP / RSYNC / GIT): [ci-cd.yml](.github/workflows/ci-cd.yml.md)
- `docgen.yml` — автогенерация структуры `docs/` и индексов: [docgen.yml](.github/workflows/docgen.yml.md)
- `minifier.yml` — автоминификация CSS/JS: [minifier.yml](.github/workflows/minifier.yml.md)
- `separated-delpoy.yml` — выборочный деплой по списку коммитов: [separated-delpoy.yml](.github/workflows/separated-delpoy.yml.md)

Как читать и редактировать документацию:

1. Авто-генерация документации создаёт `docs/` и поддерживает `docs/.docignore`. Автогенератор не перезаписывает содержимое после маркера `<!-- DOCGEN:END -->`.
2. Чтобы добавить ручную документацию к автогенерируемому файлу, открой соответствующий `docs/.../file.md` и допиши материалы после `<!-- DOCGEN:END -->`.
3. README в корне `docs/` служит как общий справочник — здесь перечислены ключевые workflow и ссылки на их подробные страницы.

Контакты и процессы:
- Автоматические коммиты (docgen, minifier) выполняются от имени `github-actions[bot]`.
- Перед пушем в `main` рекомендовано тестировать workflow на отдельной ветке (например, `ci-test`).

<!-- DOCGEN:END -->

