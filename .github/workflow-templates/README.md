# Reusable workflows — git_toolkit

Здесь лежат **тонкие workflow-шаблоны** для реальных проектов. Вся логика живёт в
`cat-of-summer/git_toolkit/.github/workflows/*.yml` и подключается как
[reusable workflow](https://docs.github.com/actions/using-workflows/reusing-workflows).

## Как подключить

1. Скопируй нужный файл из этой папки в `.github/workflows/` своего проекта.
2. Задай в своём репозитории нужные **Variables** и **Secrets**
   (Settings → Secrets and variables → Actions, либо Settings → Environments) —
   полные таблицы см. в документации конкретного workflow, ссылки ниже.
3. Готово. Шаблон вызывает логику отсюда:
   ```yaml
   jobs:
     ci-cd:
       uses: cat-of-summer/git_toolkit/.github/workflows/ci-cd.yml@main
       secrets: inherit
   ```

**`secrets: inherit`** передаёт все секреты вызывающего репозитория в reusable workflow —
объявлять их по отдельности не нужно.

**`vars` и `secrets`** внутри логики резолвятся из **твоего** репозитория и его окружений
(environments), а не из git_toolkit. То есть каждый проект настраивает себя сам.

**Закрепление версии.** Шаблоны ссылаются на `@main`. Для стабильности можно закрепиться на
теге или коммите: `...@v1` или `...@<sha>`.

**Permissions.** Права `GITHUB_TOKEN` у reusable workflow не могут превышать права
вызывающего, поэтому нужные `permissions:` уже прописаны в каждом шаблоне — не удаляй их.

---

## Шаблоны

| Шаблон | Триггеры | Назначение | Настройка |
|---|---|---|---|
| `ci-cd.yml` | `push` (ветки и теги `v*`), `pull_request`, `workflow_dispatch` | Сборка и тесты, GitHub Release, публикация в npm / Docker / Packagist, деплой | Переменные и секреты в Environment — [документация](../../docs/.github/workflows/ci-cd.yml.md) |
| `grabber.yml` | `workflow_dispatch` | Забирает файлы с сервера в репозиторий, складывает в ветку `sync/...` | `DEPLOY_*` + секрет `DEPLOY_KEY` — [документация](../../docs/.github/workflows/grabber.yml.md) |
| `docgen.yml` | `push` (только ветки), `workflow_dispatch` | Держит структуру `docs/` в соответствии с деревом репозитория | Только `docs/.docignore` — [документация](../../docs/.github/workflows/docgen.yml.md) |
| `minifier.yml` | `push` (только ветки) | Минифицирует `.css` и `.js`, коммитит `*.min.*` | Не настраивается — [документация](../../docs/.github/workflows/minifier.yml.md) |

Полный индекс документации — [`docs/README.md`](../../docs/README.md).
