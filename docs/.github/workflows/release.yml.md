<!-- DOCGEN:START -->
# release.yml — создание GitHub Release

Workflow `.github/workflows/release.yml` автоматически создаёт GitHub Release при пуше тега вида `v*` и при необходимости публикует артефакт выбранным способом (`PUBLISH_METHOD`): npm-пакет или Docker-образ. Перед этим он дожидается зелёного CI (`ci-cd.yml`), а тег можно также опубликовать/переопубликовать вручную.

Назначение:
- Дождаться успешного завершения `ci` из `ci-cd.yml` для этого тега (job `await-ci`)
- Определить и провалидировать тег, определить ветку, из которой он был создан (job `prepare`)
- Загрузить настройки деплой-окружения для этой ветки
- Опционально собрать проект
- Подготовить и переименовать файлы релиза (flatten)
- Создать GitHub Release с автоматически сгенерированным описанием и прикреплёнными файлами
- Опционально опубликовать артефакт: npm-пакет (в GitHub Packages всегда, и дополнительно в npmjs.org при наличии `NPM_TOKEN`) **или** Docker-образ в реестр

Триггеры:
- `push` с `tags: ['v*']` — обычный релиз при пуше тега
- `workflow_dispatch` с обязательным input `tag` — ручной запуск/повторная публикация по конкретному уже существующему тегу (например, если публикация упала или нужно задним числом опубликовать в npmjs.org). На этом триггере job `await-ci` пропускается — ожидания CI нет

Права (permissions):
- `contents: write` — необходимо для создания Release
- `packages: write` — необходимо для публикации npm-пакета в GitHub Packages и пуша Docker-образа в `ghcr.io`

---

## Джобы

### await-ci

Дожидается зелёного CI из `ci-cd.yml`, прежде чем что-либо публиковать.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `if` | `github.event_name == 'push'` — на `workflow_dispatch` job пропускается (ожидания нет) |

Шаг **Wait for CI (ci-cd workflow)** — `lewagon/wait-on-check-action@v1.8.0`, ждёт на этом `ref`
чек `^ci$` (job `ci` из `ci-cd.yml`), `allowed-conclusions: success,skipped`,
`fail-on-no-checks: false` (если `ci-cd.yml` не настроен или чека нет — не блокирует, релиз идёт
сразу).

### prepare

Резолвит и валидирует тег, определяет ветку, из которой он был создан.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `await-ci` |
| `if` | `!failure() && !cancelled()` — не запускается, если `await-ci` упал/отменён |
| `outputs` | `tag` — резолвленный и провалидированный тег; `branch` — имя ветки |

Шаги:
1. **Resolve & validate tag** — берёт тег из `inputs.tag` (на `workflow_dispatch`) или из
   `github.ref_name` (на `push`); проверяет формат `^v[0-9]+(\.[0-9]+)*$` и падает с понятной
   ошибкой, если тег не подходит; кладёт результат в output `tag`
2. Checkout с полной историей (`fetch-depth: 0`) и `ref: <резолвленный тег>` — важно для
   `workflow_dispatch`, где тег может отличаться от текущего `github.ref`
3. **find-branch** — поиск ветки через `git branch -r --contains $(git rev-parse HEAD)` — берётся
   первая удалённая ветка (не HEAD), из неё отрезается префикс `origin/`

Результат (`tag`, `branch`) передаётся во все нижестоящие джобы через `needs.prepare.outputs.*`.

### release

Выполняет сборку, подготовку файлов и создаёт GitHub Release.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `prepare` |
| `environment` | `${{ needs.prepare.outputs.branch }}` — имя ветки используется как Environment |
| `outputs` | `publish_method` — способ публикации для джоб `npm-publish` / `docker-publish` |

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки. Если пустая — шаг Build пропускается |
| `RELEASE_FILES` | Glob-паттерны файлов для прикрепления к Release, через запятую (например, `dist/*.zip`) |
| `PUBLISH_METHOD` | Способ публикации после релиза: `npm` → джоба `npm-publish`, `docker` → джоба `docker-publish`. Пусто — публикация не выполняется |

Шаги:
1. Checkout репозитория с `ref: needs.prepare.outputs.tag`
2. **Build** — выполняется только если `BUILD_COMMAND` не пустой; запускается через `eval` в bash с `set -euo pipefail`
3. **Flatten release asset names** — выполняется только если `RELEASE_FILES` не пустой:
   - Создаёт временную папку `.release-staging/`
   - Разбивает `RELEASE_FILES` по запятой на отдельные паттерны
   - Для каждого совпадающего файла: вычисляет относительный путь и заменяет `/` на `__` в имени файла (flatten)
   - Копирует файлы в `.release-staging/` с плоскими именами
   - Если ни один файл не совпал — завершается с ошибкой
   - Список staged-файлов передаётся в output `files` (multiline, через EOF-блок)
4. **Create GitHub Release** — используется `softprops/action-gh-release@v2`:
   - Тег: `needs.prepare.outputs.tag`
   - Название: `Release <tag>`
   - Описание: автоматически генерируется из коммитов (`generate_release_notes: true`)
   - Прикреплённые файлы: staged-файлы из шага `stage` (`steps.stage.outputs.files`)

### npm-publish

Публикует npm-пакет **всегда** в GitHub Packages и **дополнительно** в npmjs.org, если задан секрет `NPM_TOKEN`. Это не выбор одного реестра из двух — при наличии токена публикация идёт в оба параллельно и независимо.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `prepare`, `release` |
| `if` | `needs.release.outputs.publish_method == 'npm'` |
| `environment` | `${{ needs.prepare.outputs.branch }}` |

Переменные окружения:

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки (та же, что и в джобе `release`) |

Шаги:
1. Checkout репозитория с `ref: needs.prepare.outputs.tag`
2. Setup Node.js (`actions/setup-node@v4`, без выбора реестра)
3. **Build** — аналогично джобе `release`, пропускается если `BUILD_COMMAND` пустой. Выполняется **до** записи публикационных токенов, поэтому установка зависимостей (`npm ci`) использует закоммиченный в репозитории `.npmrc`, если он там есть (например, read-токен для приватных пакетов из GitHub Packages)
4. **Write .npmrc auth** — дописывает (`>>`) в project-level `.npmrc`:
   - `//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}` — всегда
   - `//registry.npmjs.org/:_authToken=${NPM_TOKEN}` — только если секрет `NPM_TOKEN` задан
   - `always-auth=true`

   Файл `.npmrc` может одновременно содержать авторизацию для обоих реестров; работает и если `.npmrc` в репозитории не было (создаётся с нуля)
5. **Set version from tag** — берёт тег из `needs.prepare.outputs.tag`; проверяет, что он соответствует формату `v#`, `v#.#` или `v#.#.#`; извлекает числа и устанавливает версию пакета через `npm version` (без создания git-тега)
6. **Publish to GitHub Packages** — выполняется всегда. Уровень доступа определяется по приватности репозитория (`github.event.repository.private`): приватный → `restricted`, публичный → `public`. Перед публикацией проверяет `npm view <pkg>@<version> --registry https://npm.pkg.github.com` — если версия там уже есть, публикация пропускается (идемпотентно, важно для повторного запуска через `workflow_dispatch` на том же теге)
7. **Publish to npmjs.org** — выполняется только если задан `NPM_TOKEN` (иначе шаг логирует пропуск и завершается успешно, `exit 0`). Та же idempotent-проверка через `npm view ... --registry https://registry.npmjs.org` и та же логика `--access`

> Имя пакета в `package.json` для GitHub Packages обязано быть scoped (`@owner/name`). Для npmjs допускается и unscoped-имя.

### docker-publish

Собирает Docker-образ из `Dockerfile` (код копируется внутрь самим Dockerfile) и пушит его в реестр. Образ можно подключить готовым в `docker-compose.yml`.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `prepare`, `release` |
| `if` | `needs.release.outputs.publish_method == 'docker'` |
| `environment` | `${{ needs.prepare.outputs.branch }}` |

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
1. Checkout репозитория с `ref: needs.prepare.outputs.tag`
2. **Validate Dockerfile** — если файла по `DOCKERFILE_PATH` нет, джоба падает с понятной ошибкой (docker-метод требует Dockerfile)
3. **Build** — опциональная пред-сборка, пропускается если `BUILD_COMMAND` пустой
4. **Resolve image ref & version** — формирует полное имя образа `<registry>/<image>` (lowercase) и версию из тега `needs.prepare.outputs.tag` (`v1.2.3` → `1.2.3`, валидация формата)
5. **Login** — `docker/login-action@v3`, единый шаг: `username` = `DOCKER_USERNAME` (vars) с фолбэком на `github.actor`, `password` = `DOCKER_TOKEN` (secret) с фолбэком на `GITHUB_TOKEN`. Для `ghcr.io` секреты не нужны; для другого реестра задай `DOCKER_USERNAME` + `DOCKER_TOKEN`
6. **Build & push** — `docker/build-push-action@v6` для платформы `linux/amd64`, теги `<version>` и `latest`

---

## Как работает определение окружения

Тег может быть запушен из любой ветки (`push`) либо указан явно при ручном ре-паблише (`workflow_dispatch` с input `tag`). Workflow определяет исходную ветку и использует её имя как GitHub Environment. Это позволяет иметь разные настройки сборки и разные файлы релиза для разных веток (например, отдельные конфиги для `main` и `develop`).

---

## Связка с `ci-cd.yml` (деплой при релизе)

`release.yml` только публикует артефакт (npm-пакет / Docker-образ). Деплой на сервер делает
`ci-cd.yml`, когда в окружении ветки задана переменная `ACTION_TRIGGER=RELEASE`. Связь между
workflow двусторонняя.

Порядок при пуше тега:

1. Запускаются оба workflow. `ci-cd.yml` выполняет job `ci`.
2. `release.yml` (job `await-ci`) дожидается зелёного чека `ci` из `ci-cd.yml`, прежде чем
   резолвить тег (`prepare`) и запускать релиз.
3. `release.yml` собирает и пушит артефакт (`PUBLISH_METHOD=npm`/`docker`).
4. Job `cd` в `ci-cd.yml` дожидается успешного завершения `release.yml`
   (через `lewagon/wait-on-check-action`, чеки `release` / `npm-publish` / `docker-publish`)
   и только потом деплоит — например, `DEPLOY_METHOD=COMMAND` с
   `AFTER_DEPLOY_COMMAND=docker compose pull && docker compose up -d`.

На `workflow_dispatch` (ручной ре-паблиш конкретного тега) `await-ci` пропускается — CI заново
не ждётся.

Поведение при провале: если `ci` упал — `release.yml` не публикует ничего. Если `release.yml`
упал — `cd` тоже падает (🔴), деплой не выполняется. Если одна из сторон (`ci-cd.yml`/`release.yml`)
не настроена или нужного чека нет — вторая сторона не ждёт (`fail-on-no-checks: false`) и работает
сама по себе. Подробности — в [ci-cd.yml.md](ci-cd.yml.md).

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
   - npm-пакет всегда публикуется в GitHub Packages; чтобы дополнительно публиковать его и в npmjs.org, добавь секрет `NPM_TOKEN`
3. Создай и запушь тег:
   ```
   git tag v1.0.0
   git push origin v1.0.0
   ```
4. Повторная публикация того же тега (например, публикация упала или `NPM_TOKEN` добавили уже
   после релиза) — запусти workflow вручную (`workflow_dispatch`) с input `tag=v1.0.0`; `await-ci`
   на этом триггере пропускается, публикация идёт сразу

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
| `GITHUB_TOKEN` | Встроенный токен GitHub Actions, используется для создания Release, публикации npm-пакета в **GitHub Packages** (всегда) и пуша Docker-образа в `ghcr.io` |
| `NPM_TOKEN` | Токен npmjs.org (npm automation / granular token). Не заменяет публикацию в GitHub Packages, а **дополняет** её: если задан и не пустой — пакет публикуется ещё и в `registry.npmjs.org` |
| `DOCKER_TOKEN` | Пароль/токен для входа в нестандартный реестр (когда `DOCKER_REGISTRY` ≠ `ghcr.io`); для `ghcr.io` не нужен |

Отладка:
- Если релиз не стартует или зависает — проверь job `await-ci`: она ждёт чек `ci` из `ci-cd.yml`
  на этом теге; если `ci-cd.yml` не настроен, ожидание не должно блокировать (`fail-on-no-checks: false`)
- Если Release не создаётся — проверь тег: джоба `prepare` валидирует формат `v#`/`v#.#`/`v#.#.#`
  и падает с понятной ошибкой, если он не подходит
- Если шаг Build падает — проверь `BUILD_COMMAND` в Environment
- Если файлы не прикрепляются — проверь паттерн `RELEASE_FILES` и результаты сборки; убедись, что хотя бы один файл совпадает
- Если npm-публикация не запускается — проверь, что `PUBLISH_METHOD=npm` задана в Environment и тег соответствует формату `v#.#.#`
- Если пакет не появился в npmjs.org — проверь секрет `NPM_TOKEN` (без него публикуется только в GitHub Packages); в GitHub Packages пакет публикуется всегда
- Если публикация не происходит, а в логе шага сообщение вида "уже есть — пропускаем" — версия
  из `package.json`/тега уже была опубликована в этом реестре; это не ошибка (idempotent-пропуск).
  Для принудительной переопубликации нужен новый тег/версия
- Уровень доступа (`--access public`/`restricted`) выбирается автоматически по приватности репозитория; для публичного scoped-пакета на npmjs репозиторий должен быть публичным
- Если docker-публикация не запускается — проверь, что `PUBLISH_METHOD=docker`; если падает на шаге Validate — в репозитории нет Dockerfile по пути `DOCKERFILE_PATH` (по умолчанию `./Dockerfile`)
- Логи: Actions → выбрать run → шаг `Create GitHub Release`, `Publish to GitHub Packages`/`Publish to npmjs.org` или `Build & push image`

<!-- DOCGEN:END -->
