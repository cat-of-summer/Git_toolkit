# Reusable workflows — Git_toolkit

Здесь лежат **тонкие workflow-шаблоны** для реальных проектов. Вся логика живёт в
`cat-of-summer/Git_toolkit/.github/workflows/*.yml` и подключается как
[reusable workflow](https://docs.github.com/actions/using-workflows/reusing-workflows).

## Как подключить

1. Скопируй нужный файл из этой папки в `.github/workflows/` своего проекта.
2. Задай в своём репозитории нужные **Variables** и **Secrets**
   (Settings → Secrets and variables → Actions), см. таблицы ниже.
3. Готово. Шаблон вызывает логику отсюда:
   ```yaml
   jobs:
     ci-cd:
       uses: cat-of-summer/Git_toolkit/.github/workflows/ci-cd.yml@main
       secrets: inherit
   ```

**`secrets: inherit`** передаёт все секреты вызывающего репозитория в reusable workflow —
объявлять их по отдельности не нужно.

**`vars` и `secrets`** внутри логики резолвятся из **твоего** репозитория и его окружений
(environments), а не из Git_toolkit. То есть каждый проект настраивает себя сам.

**Закрепление версии.** Шаблоны ссылаются на `@main`. Для стабильности можно закрепиться на
теге или коммите: `...@v1` или `...@<sha>`.

**Permissions.** Права `GITHUB_TOKEN` у reusable workflow не могут превышать права
вызывающего, поэтому нужные `permissions:` уже прописаны в каждом шаблоне — не удаляй их.

---

## minifier

Минифицирует `.css`/`.js` при каждом `push` и коммитит `*.min.*` обратно.

- **Variables:** не требуются.
- **Secrets:** не требуются (используется автоматический `GITHUB_TOKEN`).
- **Permissions:** `contents: write`.

## docgen

Генерирует зеркальную структуру документации в `docs/` при `push` / вручную.

- **Variables:** не требуются.
- **Secrets:** не требуются.
- **Permissions:** `contents: write`.

## grabber

Забирает файлы из указанных коммитов и деплоит на сервер (только `workflow_dispatch`).
Использует **environment = имя ветки** — создай в проекте environment с именем ветки и
положи туда переменные/секреты ниже.

| Тип     | Имя                | Назначение                                              |
|---------|--------------------|---------------------------------------------------------|
| var     | `DEPLOY_HOST`      | Хост сервера                                             |
| var     | `DEPLOY_USER`      | SSH-пользователь                                         |
| var     | `DEPLOY_PATH`      | Путь на сервере                                          |
| var     | `DEPLOY_METHOD`    | Метод деплоя                                             |
| var     | `DEPLOY_PORT`      | SSH-порт (по умолчанию `22`)                             |
| var     | `DEPLOY_LOCAL_DIR` | Локальная папка-источник (по умолчанию `./`)            |
| secret  | `DEPLOY_KEY`       | Приватный SSH-ключ                                       |

- **Inputs:** `commits`, `deploy_mirror` (прокидываются шаблоном).
- **Permissions:** `contents: write`.

## ci-cd

Полный CI/CD: сборка/проверка, Release + публикация в реестры (на теге), деплой.
Триггеры: `push` (ветки и теги `v*`), `pull_request`, `workflow_dispatch`.

Окружение определяется автоматически (обычно по имени ветки) — заведи соответствующие
environments в проекте и распредели переменные/секреты по ним.

**Общие / CI:**

| Тип | Имя                | Назначение                                                     |
|-----|--------------------|----------------------------------------------------------------|
| var | `ACTION_TRIGGER`   | `WORKFLOW_DISPATCH` \| `PUSH` \| `RELEASE`                      |
| var | `PUBLISH_METHOD`   | `npm` \| `docker` \| `packagist`                               |
| var | `RUNS_ON`          | Раннеры, напр. `ubuntu-latest` (можно `a,b` через запятую)      |
| var | `TOOLCHAIN`        | Тулчейн сборки (поддерживает разделители `:` и `@`)            |
| var | `BUILD_COMMAND`    | Команда сборки                                                  |
| var | `CI_COMMAND`       | Команда проверки/тестов                                         |
| var | `MULTIPLE_PACKAGES`| Суффикс имени пакета по ветке (npm/docker)                      |
| var | `RELEASE_FILES`    | Файлы, прикладываемые к GitHub Release                          |

**Публикация — Docker:**

| Тип    | Имя                 | Назначение                          |
|--------|---------------------|-------------------------------------|
| var    | `DOCKER_REGISTRY`   | Реестр                              |
| var    | `DOCKER_IMAGE`      | Имя образа                          |
| var    | `DOCKERFILE_PATH`   | Путь к Dockerfile                   |
| var    | `BUILD_CONTEXT`     | Контекст сборки                     |
| var    | `DOCKER_USERNAME`   | Пользователь реестра                |
| var    | `DOCKER_BUILD_ARGS` | build-args                          |
| secret | `DOCKER_TOKEN`      | Токен реестра                       |
| secret | `PAT_TOKEN`         | PAT (напр. для GitHub Packages)     |

**Публикация — Packagist:**

| Тип    | Имя                   | Назначение              |
|--------|-----------------------|-------------------------|
| var    | `PACKAGIST_USERNAME`  | Пользователь Packagist  |
| secret | `PACKAGIST_API_TOKEN` | API-токен Packagist     |

**Деплой (job cd):**

| Тип    | Имя                     | Назначение                          |
|--------|-------------------------|-------------------------------------|
| var    | `DEPLOY_HOST`           | Хост                                |
| var    | `DEPLOY_USER`           | SSH-пользователь                    |
| var    | `DEPLOY_PATH`           | Путь на сервере                     |
| var    | `DEPLOY_METHOD`         | Метод деплоя                        |
| var    | `DEPLOY_PORT`           | SSH-порт                            |
| var    | `DEPLOY_LOCAL_DIR`      | Локальная папка-источник            |
| var    | `DEPLOY_MIRROR`         | Режим зеркалирования по умолчанию   |
| var    | `DEPLOY_LAST_COMMITS`   | Кол-во последних коммитов           |
| var    | `BEFORE_DEPLOY_COMMAND` | Команда до деплоя                   |
| var    | `AFTER_DEPLOY_COMMAND`  | Команда после деплоя                |
| secret | `DEPLOY_KEY`            | Приватный SSH-ключ                  |

- **Inputs:** `run_ci`, `run_release`, `run_cd`, `environment`, `runs_on`,
  `publish_method`, `commits`, `deploy_mirror` (прокидываются шаблоном при
  ручном запуске).
- **Permissions:** `contents: write`, `packages: write`.
