<!-- DOCGEN:START -->
# release.yml — создание GitHub Release

Workflow `.github/workflows/release.yml` автоматически создаёт GitHub Release при создании тега и при необходимости публикует артефакт выбранным способом (`PUBLISH_METHOD`): npm-пакет или Docker-образ.

Назначение:
- Определить ветку, из которой был создан тег
- Загрузить настройки деплой-окружения для этой ветки
- Опционально собрать проект
- Подготовить и переименовать файлы релиза (flatten)
- Создать GitHub Release с автоматически сгенерированным описанием и прикреплёнными файлами
- Опционально опубликовать артефакт: npm-пакет в GitHub Packages **или** Docker-образ в реестр

Триггеры:
- `create` — срабатывает при создании любого тега или ветки; джоба `get-branch` дополнительно проверяет `github.ref_type == 'tag'`

Права (permissions):
- `contents: write` — необходимо для создания Release
- `packages: write` — необходимо для публикации npm-пакета в GitHub Packages и пуша Docker-образа в `ghcr.io`

---

## Джобы

### get-branch

Определяет ветку, содержащую коммит тега.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `if` | `github.ref_type == 'tag'` — выполняется только для тегов |
| `outputs` | `branch` — имя ветки |

Шаги:
1. Checkout с полной историей (`fetch-depth: 0`)
2. Поиск ветки через `git branch -r --contains <sha>` — берётся первая удалённая ветка (не HEAD), из неё отрезается префикс `origin/`

Результат передаётся в джобы `release`, `npm-publish` и `docker-publish` через `needs.get-branch.outputs.branch`.

### release

Выполняет сборку, подготовку файлов и создаёт GitHub Release.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `get-branch` |
| `environment` | `${{ needs.get-branch.outputs.branch }}` — имя ветки используется как Environment |
| `outputs` | `publish_method` — способ публикации для джоб `npm-publish` / `docker-publish` |

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки. Если пустая — шаг Build пропускается |
| `RELEASE_FILES` | Glob-паттерны файлов для прикрепления к Release, через запятую (например, `dist/*.zip`) |
| `PUBLISH_METHOD` | Способ публикации после релиза: `npm` → джоба `npm-publish`, `docker` → джоба `docker-publish`. Пусто — публикация не выполняется |

Шаги:
1. Checkout репозитория
2. **Build** — выполняется только если `BUILD_COMMAND` не пустой; запускается через `eval` в bash с `set -euo pipefail`
3. **Flatten release asset names** — выполняется только если `RELEASE_FILES` не пустой:
   - Создаёт временную папку `.release-staging/`
   - Разбивает `RELEASE_FILES` по запятой на отдельные паттерны
   - Для каждого совпадающего файла: вычисляет относительный путь и заменяет `/` на `__` в имени файла (flatten)
   - Копирует файлы в `.release-staging/` с плоскими именами
   - Если ни один файл не совпал — завершается с ошибкой
   - Список staged-файлов передаётся в output `files` (multiline, через EOF-блок)
4. **Create GitHub Release** — используется `softprops/action-gh-release@v2`:
   - Тег: `github.ref_name`
   - Название: `Release <tag>`
   - Описание: автоматически генерируется из коммитов (`generate_release_notes: true`)
   - Прикреплённые файлы: staged-файлы из шага `stage` (`steps.stage.outputs.files`)

### npm-publish

Публикует npm-пакет в GitHub Packages.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `get-branch`, `release` |
| `if` | `needs.release.outputs.publish_method == 'npm'` |
| `environment` | `${{ needs.get-branch.outputs.branch }}` |

Переменные окружения:

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки (та же, что и в джобе `release`) |

Шаги:
1. Checkout репозитория
2. Setup Node.js с реестром `https://npm.pkg.github.com`
3. **Build** — аналогично джобе `release`, пропускается если `BUILD_COMMAND` пустой
4. **Override npm auth for publish** — патчит `.npmrc`, подставляя `GITHUB_TOKEN` в качестве токена аутентификации
5. **Set version from tag** — проверяет, что тег соответствует формату `v#`, `v#.#` или `v#.#.#`; извлекает числа и устанавливает версию пакета через `npm version` (без создания git-тега)
6. `npm publish` — публикует пакет

### docker-publish

Собирает Docker-образ из `Dockerfile` (код копируется внутрь самим Dockerfile) и пушит его в реестр. Образ можно подключить готовым в `docker-compose.yml`.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `get-branch`, `release` |
| `if` | `needs.release.outputs.publish_method == 'docker'` |
| `environment` | `${{ needs.get-branch.outputs.branch }}` |

Переменные окружения:

| Переменная | Default | Описание |
|------------|---------|----------|
| `DOCKER_REGISTRY` | `ghcr.io` | Целевой реестр |
| `DOCKER_IMAGE` | `${{ github.repository }}` | Имя образа без реестра (`owner/repo`); приводится к нижнему регистру |
| `DOCKERFILE_PATH` | `./Dockerfile` | Путь к Dockerfile |
| `DOCKER_CONTEXT` | `.` | Контекст сборки |
| `DOCKER_USERNAME` | — | Логин для нестандартного реестра (для `ghcr.io` не нужен) |
| `BUILD_COMMAND` | — | Опциональная пред-сборка перед сборкой образа |

Шаги:
1. Checkout репозитория
2. **Validate Dockerfile** — если файла по `DOCKERFILE_PATH` нет, джоба падает с понятной ошибкой (docker-метод требует Dockerfile)
3. **Build** — опциональная пред-сборка, пропускается если `BUILD_COMMAND` пустой
4. **Resolve image ref & version** — формирует полное имя образа `<registry>/<image>` (lowercase) и версию из тега (`v1.2.3` → `1.2.3`, валидация формата)
5. **Login** — `docker/login-action@v3`, единый шаг: `username` = `DOCKER_USERNAME` (vars) с фолбэком на `github.actor`, `password` = `DOCKER_TOKEN` (secret) с фолбэком на `GITHUB_TOKEN`. Для `ghcr.io` секреты не нужны; для другого реестра задай `DOCKER_USERNAME` + `DOCKER_TOKEN`
6. **Build & push** — `docker/build-push-action@v6` для платформы `linux/amd64`, теги `<version>` и `latest`

---

## Как работает определение окружения

Тег может быть создан из любой ветки. Workflow определяет исходную ветку и использует её имя как GitHub Environment. Это позволяет иметь разные настройки сборки и разные файлы релиза для разных веток (например, отдельные конфиги для `main` и `develop`).

---

## Связка с `ci-cd.yml` (деплой при релизе)

`release.yml` только публикует артефакт (npm-пакет / Docker-образ). Деплой на сервер делает
`ci-cd.yml`, когда в окружении ветки задана переменная `ACTION_TRIGGER=RELEASE`.

Порядок при создании тега:

1. Запускаются оба workflow (оба на `create` + `get-branch`).
2. `release.yml` собирает и пушит артефакт (`PUBLISH_METHOD=npm`/`docker`).
3. Job `cd` в `ci-cd.yml` дожидается успешного завершения `release.yml`
   (через `lewagon/wait-on-check-action`, чеки `release` / `npm-publish` / `docker-publish`)
   и только потом деплоит — например, `DEPLOY_METHOD=COMMAND` с
   `AFTER_DEPLOY_COMMAND=docker compose pull && docker compose up -d`.

Поведение при провале: если `release.yml` упал — `cd` тоже падает (🔴), деплой не выполняется.
Если `ci-cd.yml`/`ACTION_TRIGGER=RELEASE` не настроен — `release.yml` работает сам по себе.
Подробности — в [ci-cd.yml.md](ci-cd.yml.md).

---

## Как работает flatten файлов

Шаг "Flatten release asset names" решает проблему коллизий имён при прикреплении файлов из разных директорий. Символ `/` в относительном пути заменяется на `__`:

```
dist/linux/app   →  linux__app
dist/windows/app.exe  →  windows__app.exe
```

---

## Настройка

1. Создай GitHub Environment с именем ветки (например, `main`) в Settings → Environments
2. Задай переменные в этом Environment:
   - `BUILD_COMMAND` — команда сборки (необязательно)
   - `RELEASE_FILES` — glob-паттерны файлов через запятую (необязательно)
   - `PUBLISH_METHOD` — `npm` или `docker` для публикации (необязательно)
3. Создай и запушь тег:
   ```
   git tag v1.0.0
   git push origin v1.0.0
   ```

Примеры значений переменных:

```
BUILD_COMMAND=npm ci && npm run build
RELEASE_FILES=dist/*.zip
PUBLISH_METHOD=npm
```

```
BUILD_COMMAND=make release
RELEASE_FILES=bin/app-linux,bin/app-windows.exe
```

Docker (образ в `ghcr.io`, подключаемый в `docker-compose.yml`):

```
PUBLISH_METHOD=docker
# опционально:
DOCKER_IMAGE=myorg/myapp
DOCKERFILE_PATH=./docker/Dockerfile
```
```yaml
# docker-compose.yml
services:
  app:
    image: ghcr.io/<owner>/<repo>:latest
```

---

Секреты:

| Секрет | Описание |
|--------|----------|
| `GITHUB_TOKEN` | Встроенный токен GitHub Actions, используется для создания Release, публикации npm-пакета и пуша Docker-образа в `ghcr.io` |
| `DOCKER_TOKEN` | Пароль/токен для входа в нестандартный реестр (когда `DOCKER_REGISTRY` ≠ `ghcr.io`); для `ghcr.io` не нужен |

Отладка:
- Если Release не создаётся — проверь, что создан именно **тег** (не ветка); джоба `get-branch` проверяет `ref_type == 'tag'`
- Если шаг Build падает — проверь `BUILD_COMMAND` в Environment
- Если файлы не прикрепляются — проверь паттерн `RELEASE_FILES` и результаты сборки; убедись, что хотя бы один файл совпадает
- Если npm-публикация не запускается — проверь, что `PUBLISH_METHOD=npm` задана в Environment и тег соответствует формату `v#.#.#`
- Если docker-публикация не запускается — проверь, что `PUBLISH_METHOD=docker`; если падает на шаге Validate — в репозитории нет Dockerfile по пути `DOCKERFILE_PATH` (по умолчанию `./Dockerfile`)
- Логи: Actions → выбрать run → шаг `Create GitHub Release`, `npm publish` или `Build & push image`

<!-- DOCGEN:END -->
