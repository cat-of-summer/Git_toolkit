<!-- DOCGEN:START -->
# release.yml — создание GitHub Release

Workflow `.github/workflows/release.yml` автоматически создаёт GitHub Release при пуше тега вида `v*` и при необходимости публикует артефакт выбранным способом (`PUBLISH_METHOD`): npm-пакет или Docker-образ. Перед этим он дожидается зелёного CI (`ci-cd.yml`), а тег можно также опубликовать/переопубликовать вручную.

Назначение:
- Дождаться успешного завершения `ci` из `ci-cd.yml` для этого тега (job `await-ci`)
- Определить и провалидировать тег, определить целевую ветку/окружение — автоматически по коммиту либо явно из префикса тега `{branch}/{vX.Y.Z}` (job `prepare`)
- Загрузить настройки деплой-окружения для этой ветки
- Опционально собрать проект
- Подготовить и переименовать файлы релиза (flatten)
- Создать GitHub Release с автоматически сгенерированным описанием и прикреплёнными файлами
- Опционально опубликовать артефакт: npm-пакет (в GitHub Packages **и** npmjs.org — оба реестра) **или** Docker-образ в реестр. Публикация в npmjs.org идёт через OIDC trusted publishing (без токена), в GitHub Packages — по встроенному `GITHUB_TOKEN`

Триггеры:
- `push` с `tags: ['v*', '**/v*']` — релиз при пуше тега. Первый паттерн ловит обычные теги (`v1.2.3`), второй — теги с явной веткой (`{branch}/v1.2.3`, в т.ч. со слешами: `release/1.x/v1.2.3`). Важно: glob `*` не матчит `/`, поэтому без паттерна `**/v*` теги с префиксом ветки workflow бы **не запускали**
- `workflow_dispatch` (без inputs) — ручной запуск/повторная публикация. Тег выбирается в дропдауне **«Use workflow from»** при запуске (Run workflow) — нужно указать именно **тег**, а не ветку (иначе `prepare` упадёт с «Запускать нужно на теге»). На этом триггере job `await-ci` пропускается — ожидания CI нет, публикация идёт сразу. Идемпотентные проверки делают повтор на том же теге безопасным

Формат тега — **жёстко диктуется** репозиторной переменной `MULTIPLE_PACKAGES` (проверка в самом начале, job `prepare`):
- `MULTIPLE_PACKAGES` выключен (по умолчанию) → тег обязан быть **плоским** `vX.Y.Z`; целевая ветка (Environment) определяется автоматически по коммиту. Тег с префиксом ветки в этом режиме **отклоняется** с ошибкой.
- `MULTIPLE_PACKAGES=true` → тег обязан быть с **явной веткой** `{branch}/vX.Y.Z` (например, `main/v1.2.3`). Плоский `vX.Y.Z` в этом режиме **отклоняется**. Ветка берётся из префикса однозначно; её наличие на remote проверяется (`find-branch`), если ветки нет — workflow падает. Префикс `{branch}/` вырезается из версии, поэтому в заголовке релиза и в версии пакета/образа используется только `vX.Y.Z`, а имя артефакта получает суффикс ветки: npm-пакет `@owner/name-<branch>`, Docker-образ `owner/repo-<branch>`.

> `MULTIPLE_PACKAGES` должна быть **repository-level** переменной (Settings → Secrets and variables → Actions → Variables), т.к. проверка идёт в `prepare` **до** выбора Environment — environment-scoped значение там не видно.

Права (permissions):
- `contents: write` — необходимо для создания Release
- `packages: write` — необходимо для публикации npm-пакета в GitHub Packages и пуша Docker-образа в `ghcr.io`

> Джоба `npm-publish` дополнительно поднимает права до `id-token: write` — это нужно для OIDC trusted publishing в npmjs.org (обмен короткоживущего OIDC-токена GitHub на публикационный токен npm). Без этого права публикация в npmjs.org работать не будет.

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
| `outputs` | `ref` — полный реальный тег в git (`main/v1.2.3` или `v1.2.3`), используется для checkout и `tag_name` релиза; `tag` — очищенная версия (`v1.2.3`) для заголовка и версии пакета/образа; `branch` — имя окружения |

Шаги:
1. **Resolve & validate tag** — берёт тег из `github.ref_name`; если он содержит `/`, делит по
   **последнему** `/` на префикс-ветку (`{branch}`, может содержать `/`, напр. `release/1.x`) и
   версию (`{vX.Y.Z}`), иначе явной ветки нет и весь тег — это версия; проверяет версию по
   `^v[0-9]+(\.[0-9]+)*$`. Затем **обеспечивает формат тега по `vars.MULTIPLE_PACKAGES`**
   (repository-level): при `true` требует префикс ветки (плоский тег → ошибка), при выключенном —
   требует плоский тег (тег с префиксом → ошибка). Кладёт в outputs `ref` (полный тег), `tag`
   (версия) и `branch_explicit` (префикс-ветка или пусто)
2. Checkout с полной историей (`fetch-depth: 0`) и `ref: <полный тег>` — важно для
   `workflow_dispatch`, где тег может отличаться от текущего `github.ref`
3. **find-branch** — определяет окружение:
   - если задан `branch_explicit` — проверяет наличие ветки на remote через
     `git ls-remote --exit-code --heads origin <branch>` (падает, если ветки нет) и берёт её;
   - иначе — автоопределение через `git branch -r --contains <sha>`: первая удалённая ветка
     (не HEAD), из неё отрезается префикс `origin/`;
   - итоговое имя санитизируется для использования как Environment / Docker-тег / заголовок:
     `/` заменяется на `-` (`release/1.x` → `release-1.x`); проверка наличия ветки при этом идёт
     по **реальному** имени со слешами

Результат (`ref`, `tag`, `branch`) передаётся во все нижестоящие джобы через `needs.prepare.outputs.*`.

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
1. Checkout репозитория с `ref: needs.prepare.outputs.ref` (полный реальный тег)
2. **Build** — выполняется только если `BUILD_COMMAND` не пустой; запускается через `eval` в bash с `set -euo pipefail`
3. **Flatten release asset names** — выполняется только если `RELEASE_FILES` не пустой:
   - Создаёт временную папку `.release-staging/`
   - Разбивает `RELEASE_FILES` по запятой на отдельные паттерны
   - Для каждого совпадающего файла: вычисляет относительный путь и заменяет `/` на `__` в имени файла (flatten)
   - Копирует файлы в `.release-staging/` с плоскими именами
   - Если ни один файл не совпал — завершается с ошибкой
   - Список staged-файлов передаётся в output `files` (multiline, через EOF-блок)
4. **Create GitHub Release** — используется `softprops/action-gh-release@v2`:
   - Тег (`tag_name`): `needs.prepare.outputs.ref` — полный реальный тег в git (релиз привязывается к уже существующему тегу, лишний тег не создаётся)
   - Название зависит от режима: при выключенном `MULTIPLE_PACKAGES` (плоский тег) — просто `Release <tag>` (например, `Release v1.2.3`), без имени ветки; при `MULTIPLE_PACKAGES=true` — `<branch> release <tag>` (например, `main release v1.2.3`), т.к. там ветка значима. Версия — из очищенного `outputs.tag` (без префикса `{branch}/`)
   - Описание: автоматически генерируется из коммитов (`generate_release_notes: true`)
   - Прикреплённые файлы: staged-файлы из шага `stage` (`steps.stage.outputs.files`)

### npm-publish

Публикует npm-пакет в два реестра, каждый отдельным шагом, с разной ролью:
- **GitHub Packages** — **обязательный** реестр, публикуется **первым**. Авторизация по встроенному `GITHUB_TOKEN` (право `packages: write`). Если этот шаг падает — падает вся джоба (общая ошибка релиза).
- **npmjs.org** — **опциональный** (best-effort), идёт вторым, только если GitHub Packages прошёл. Через **OIDC trusted publishing**, без токенов (требует `id-token: write` и настроенного на npmjs.com Trusted Publisher — см. «Настройка»). Помечен `continue-on-error: true`: если публикация не удалась (например, Trusted Publisher не настроен) — это **не роняет** джобу, просто фиксируется как жёлтый (failed, но допустимый) шаг.

Логика намеренно асимметрична: GitHub Packages доступен всегда (по `GITHUB_TOKEN`) и потому обязателен, а npmjs — «бонус», публикация в который лишь пытается выполниться.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `prepare`, `release` |
| `if` | `needs.release.outputs.publish_method == 'npm'` |
| `environment` | `${{ needs.prepare.outputs.branch }}` |
| `permissions` | `contents: read`, `packages: write`, `id-token: write` |

Переменные окружения:

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки (та же, что и в джобе `release`) |

Шаги:
1. Checkout репозитория с `ref: needs.prepare.outputs.ref` (полный реальный тег)
2. Setup Node.js (`actions/setup-node@v4`, Node 22, без выбора реестра — дефолтный npmjs для `npm ci` в Build)
3. **Build** — аналогично джобе `release`, пропускается если `BUILD_COMMAND` пустой. Выполняется **до** публикации, поэтому установка зависимостей (`npm ci`) использует закоммиченный в репозитории `.npmrc`, если он там есть (например, read-токен для приватных пакетов)
4. **Drop committed .npmrc** — `rm -f .npmrc`. Убирает **project-level** `.npmrc` из репозитория, если он закоммичен. Это критично: project-config приоритетнее user-config, который пишет `setup-node`, поэтому строка вида `//npm.pkg.github.com/:_authToken=…` в закоммиченном `.npmrc` **перебивает** `NODE_AUTH_TOKEN` и npm уходит в `401` со «своим» (часто протухшим/захардкоженным) токеном. Удаляем после Build, чтобы авторизацию задавал только `setup-node`
5. **Set version from tag** — берёт тег из `needs.prepare.outputs.tag`; проверяет формат `v#`/`v#.#`/`v#.#.#`; извлекает числа и устанавливает версию пакета через `npm version` (без создания git-тега)
6. **Apply per-branch package name** — только при `pkg_suffix != ''` (т.е. `MULTIPLE_PACKAGES=true`): `npm pkg set name="<name>-<branch>"`, чтобы каждая ветка публиковалась отдельным пакетом (`@owner/name-<branch>`). Применяется до публикации, поэтому действует на оба реестра
7. **Setup Node for GitHub Packages** — повторный `actions/setup-node@v4` с `registry-url: https://npm.pkg.github.com` и `scope: @<owner>` (`github.repository_owner`). Он пишет user-level `.npmrc` со связкой scope↔реестр↔токен (`//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}`)
8. **Publish to GitHub Packages** (обязательный, первый) — авторизация `NODE_AUTH_TOKEN=${{ secrets.GITHUB_TOKEN }}`. Уровень доступа по приватности репозитория (`private` → `restricted`, иначе `public`). Перед публикацией удаляет из локального `package.json` жёсткий `publishConfig.registry` (иначе он навязывает npmjs). Idempotent-проверка `npm view <pkg>@<version> --registry https://npm.pkg.github.com` — если версия уже есть, пропуск. При падении `npm publish` шаг выводит debug-лог npm прямо в лог Actions (сворачиваемая группа) и роняет всю джобу
9. **Setup Node for npmjs.org** — `actions/setup-node@v4` с `registry-url: https://registry.npmjs.org` **и `scope: @<owner>`**. `scope` здесь обязателен: шаг 7 записал `@<owner>:registry=https://npm.pkg.github.com`, а scoped-registry приоритетнее флага `--registry`; без перезаписи scope на npmjs публикация в npmjs ушла бы в GitHub Packages (с невалидным для него токеном) → `401`
10. **Ensure npm CLI supports trusted publishing** — `npm install -g npm@latest` (trusted publishing требует npm ≥ 11.5.1); стоит прямо перед npmjs-публикацией, чтобы гарантировать нужную версию npm
11. **Publish to npmjs.org** (опциональный, второй) — `continue-on-error: true`, идёт только если GitHub Packages прошёл; своим падением джобу не роняет. Публикация через OIDC trusted publishing, та же idempotent-проверка и логика `--access`. Пропускается автоматически, если раньше упал Build/GitHub Packages

> ⚠️ **Не коммить `.npmrc` с токенами в репозиторий** (тем более публичный). Во-первых, это утечка секрета (GitHub secret scanning автоматически отзовёт токен → публикация начнёт падать с `401`). Во-вторых, project-level `.npmrc` перебивает авторизацию от `setup-node`. Джоба страхуется шагом **Drop committed .npmrc**, но правильнее не держать такой файл в репозитории вовсе.

> Имя пакета в `package.json` для GitHub Packages обязано быть scoped (`@owner/name`), причём scope должен совпадать с владельцем репозитория. Для npmjs допускается и unscoped-имя, но при публикации в оба реестра имя должно быть scoped.

> **Первая публикация нового имени.** OIDC trusted publishing работает для **уже существующего** на npmjs пакета (Trusted Publisher настраивается на странице настроек пакета). Для самого первого публиша нового имени (в т.ч. суффиксованного `@owner/name-<branch>` при `MULTIPLE_PACKAGES=true`) пакет сначала нужно создать разово — например, опубликовать вручную токеном, — а затем настроить Trusted Publisher и переключиться на автоматику.

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
| `MULTIPLE_PACKAGES` | `false` | **Repository-level** переменная (не environment — читается в `prepare` до выбора Environment). Если `true` — образ собирается **отдельным на каждую ветку**: к имени добавляется суффикс `-<branch>` (`owner/repo` → `owner/repo-<branch>`), а тег обязан быть в форме `{branch}/vX.Y.Z` (иначе `prepare` падает). По умолчанию (`false`) — один образ репозиторного уровня, а тег обязан быть плоским `vX.Y.Z`. Диктует формат тега для всего workflow — см. раздел «Формат тега» |

Шаги:
1. Checkout репозитория с `ref: needs.prepare.outputs.ref` (полный реальный тег)
2. **Validate Dockerfile** — если файла по `DOCKERFILE_PATH` нет, джоба падает с понятной ошибкой (docker-метод требует Dockerfile)
3. **Build** — опциональная пред-сборка, пропускается если `BUILD_COMMAND` пустой
4. **Resolve image ref & version** — формирует полное имя образа `<registry>/<image>` (lowercase) и версию из очищенного тега `needs.prepare.outputs.tag` (`v1.2.3` → `1.2.3`, валидация формата). Если `MULTIPLE_PACKAGES=true`, к имени образа добавляется суффикс `-<branch>` (например, `owner/repo-main`), ветка берётся из `needs.prepare.outputs.branch` (санитизированная `/`→`-`). Наличие явной ветки в теге в этом режиме уже гарантировано шагом `prepare` (проверка формата в самом начале)
5. **Login** — `docker/login-action@v3`, единый шаг: `username` = `DOCKER_USERNAME` (vars) с фолбэком на `github.actor`, `password` = `DOCKER_TOKEN` (secret) с фолбэком на `GITHUB_TOKEN`. Для `ghcr.io` секреты не нужны; для другого реестра задай `DOCKER_USERNAME` + `DOCKER_TOKEN`
6. **Build & push** — `docker/build-push-action@v6` для платформы `linux/amd64`. Список тегов формируется в шаге `meta`: `<version>` и `latest` — всегда; тег `<branch>` (имя окружения, санитизированное `/`→`-`) добавляется **только** в repo-level режиме (`MULTIPLE_PACKAGES` выключен), где имя образа общее для всех веток. При `MULTIPLE_PACKAGES=true` ветка уже зашита в имя образа (`owner/repo-<branch>`), поэтому тег ветки не добавляется — достаточно `<version>` + `latest`

---

## Как работает определение окружения

> **Почему нельзя просто «узнать ветку тега».** Git-тег указывает на **коммит**, а не на ветку —
> имя ветки в теге нигде не хранится. Дропдаун *Target* в GitHub UI лишь выбирает, на какой коммит
> повесить тег; из CLI (`git tag … && git push`) ветка вообще не участвует. Если коммит достижим из
> нескольких веток, тег «принадлежит» им всем. Поэтому автоопределение — это принципиально
> **догадка**, а единственный надёжный способ привязать релиз к ветке — назвать её прямо в теге.

Workflow определяет целевую ветку и использует её имя как GitHub Environment. Это позволяет иметь разные настройки сборки и разные файлы релиза для разных веток (например, отдельные конфиги для `main` и `develop`). Какой из двух способов действует — задаёт репозиторная переменная `MULTIPLE_PACKAGES` (она же диктует формат тега):

- **`MULTIPLE_PACKAGES` выключен → автоопределение (тег `vX.Y.Z`)** — ветка ищется по коммиту тега через `git branch -r --contains`, берётся первая подходящая. Просто, но при коллизии (коммит есть в нескольких ветках) выбор неоднозначен. Тег с префиксом ветки в этом режиме отклоняется.
- **`MULTIPLE_PACKAGES=true` → явная привязка (тег `{branch}/vX.Y.Z`)** — ветка берётся из префикса тега однозначно, автоопределение не выполняется. Наличие ветки проверяется на remote; если её нет — workflow падает. Пример: `main/v1.2.3` → Environment `main`. Префикс вырезается, поэтому заголовок релиза (`main release v1.2.3`) и версия пакета/образа (`1.2.3`) не дублируют ветку. Плоский тег `vX.Y.Z` в этом режиме отклоняется.

Санитизация: в имени Environment (а также в Docker-теге и заголовке) `/` заменяется на `-`. Поэтому тег `release/1.x/v1.2.3` нацеливается на Environment **`release-1.x`** — именно так и надо назвать Environment в настройках репозитория. Одностегментные ветки (`main`, `develop`) не затрагиваются.

**Отдельный пакет на ветку.** Смысл `MULTIPLE_PACKAGES=true`: если на разных ветках живут по сути
разные проекты, имя артефакта получает суффикс `-<branch>`, и ветки не пересекаются:
- npm-пакет → `@owner/name-<branch>` (шаг `Apply per-branch package name`, действует на оба реестра);
- Docker-образ → `owner/repo-<branch>`.

Именно поэтому режим и обязывает указывать ветку в теге — без достоверной ветки имя было бы
неоднозначным. Суффикс формируется один раз в `prepare` (output `pkg_suffix`) и потребляется и
npm-, и docker-джобой.

> Для composer/Packagist суффикс **не применяется**: Packagist читает имя пакета из `composer.json`
> в самом репозитории (переименовать на лету, как у npm, нельзя), и модель Packagist — «один
> репозиторий = один пакет». Несколько composer-пакетов делаются отдельными репозиториями, а не
> ветками одного.

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

1. Создай GitHub Environment с именем ветки (например, `main`) в Settings → Environments. Для веток со слешами имя Environment пишется через `-`: ветке `release/1.x` соответствует Environment `release-1.x`
2. Задай переменные в этом Environment:
   - `BUILD_COMMAND` — команда сборки (необязательно)
   - `RELEASE_FILES` — glob-паттерны файлов через запятую (необязательно)
   - `PUBLISH_METHOD` — `npm` или `docker` для публикации (необязательно)
   - при `PUBLISH_METHOD=npm` пакет публикуется в **оба** реестра — GitHub Packages (по `GITHUB_TOKEN`, ничего настраивать не нужно) и npmjs.org. Для npmjs.org **токен не нужен** — но требуется один раз настроить **Trusted Publisher** на npmjs.com (см. ниже)
3. Для публикации в npmjs.org настрой Trusted Publisher (только один раз, на стороне npmjs.com):
   Package Settings → Trusted Publisher → GitHub Actions → укажи организацию/владельца, репозиторий,
   имя workflow-файла (`release.yml`), environment оставь пустым. Пакет при этом уже должен
   существовать на npmjs; для самой первой публикации нового имени — опубликуй разово вручную
   токеном, затем настрой Trusted Publisher и дальше публикуй автоматически
4. Создай и запушь тег:
   ```
   git tag v1.0.0
   git push origin v1.0.0
   ```
   Либо с явной привязкой к ветке/окружению (однозначно, без автоопределения):
   ```
   git tag main/v1.0.0
   git push origin main/v1.0.0
   ```
5. Повторная публикация того же тега (например, публикация упала или Trusted Publisher настроили уже
   после релиза) — запусти workflow вручную: Actions → release → **Run workflow**, в дропдауне
   **«Use workflow from»** выбери нужный **тег** (`v1.0.0`). `await-ci` на этом триггере пропускается,
   публикация идёт сразу (idempotent-проверка пропустит реестры, где версия уже есть)

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
# отдельный образ на каждую ветку (owner/repo-<branch>); тогда теги обязаны
# быть в форме {branch}/vX.Y.Z. Задаётся на уровне РЕПОЗИТОРИЯ, не Environment:
MULTIPLE_PACKAGES=true
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
| `GITHUB_TOKEN` | Встроенный токен GitHub Actions, используется для создания Release, публикации npm-пакета в **GitHub Packages** и пуша Docker-образа в `ghcr.io` |
| `DOCKER_TOKEN` | Пароль/токен для входа в нестандартный реестр (когда `DOCKER_REGISTRY` ≠ `ghcr.io`); для `ghcr.io` не нужен |

> Токен npmjs.org (`NPM_TOKEN`) **больше не нужен**: публикация в npmjs.org идёт через OIDC trusted
> publishing. Вместо секрета настраивается Trusted Publisher на npmjs.com (см. «Настройка»).
> npm окончательно отключил classic-токены в ноябре 2025, поэтому OIDC — рекомендуемый способ.

Отладка:
- Если релиз не стартует или зависает — проверь job `await-ci`: она ждёт чек `ci` из `ci-cd.yml`
  на этом теге; если `ci-cd.yml` не настроен, ожидание не должно блокировать (`fail-on-no-checks: false`)
- Если Release не создаётся — проверь тег: джоба `prepare` валидирует формат версии
  `v#`/`v#.#`/`v#.#.#` (для тега `{branch}/vX.Y.Z` — часть после последнего `/`) и падает с
  понятной ошибкой, если он не подходит
- Если `prepare` падает на шаге `find-branch` с `Branch '...' from tag not found` — в теге
  `{branch}/vX.Y.Z` указана несуществующая ветка; проверь имя ветки в префиксе тега (сверяется с
  реальным именем на remote, со слешами)
- Если шаг Build падает — проверь `BUILD_COMMAND` в Environment
- Если файлы не прикрепляются — проверь паттерн `RELEASE_FILES` и результаты сборки; убедись, что хотя бы один файл совпадает
- Если npm-публикация не запускается — проверь, что `PUBLISH_METHOD=npm` задана в Environment и тег соответствует формату `v#.#.#`
- Если публикация в **npmjs.org** падает с ошибкой авторизации/OIDC — проверь, что настроен Trusted Publisher на npmjs.com (владелец, репозиторий, workflow-файл `release.yml`), что у джобы есть право `id-token: write`, и что пакет с таким именем уже существует на npmjs (для нового имени первый публиш — разово вручную токеном)
- Если публикация в **GitHub Packages** падает с `401 Unauthorized … unauthenticated` — имя пакета должно быть scoped и scope должен совпадать с владельцем репозитория (`@owner/...`, задаётся в `Setup Node for GitHub Packages` через `github.repository_owner`); если пакет с этим именем ранее был привязан к другому репозиторию, `GITHUB_TOKEN` из текущего может его не перезаписать. Историческая причина 401: npm 11 удалил `always-auth`, поэтому ручной `.npmrc` без scope-привязки перестал слать токен — сейчас за это отвечает отдельный `setup-node` со `scope`
- Если публикация не происходит, а в логе шага сообщение вида "уже есть — пропускаем" — версия
  из `package.json`/тега уже была опубликована в этом реестре; это не ошибка (idempotent-пропуск).
  Для принудительной переопубликации нужен новый тег/версия
- Уровень доступа (`--access public`/`restricted`) выбирается автоматически по приватности репозитория; для публичного scoped-пакета на npmjs репозиторий должен быть публичным
- Если docker-публикация не запускается — проверь, что `PUBLISH_METHOD=docker`; если падает на шаге Validate — в репозитории нет Dockerfile по пути `DOCKERFILE_PATH` (по умолчанию `./Dockerfile`)
- Если `prepare` падает на несоответствии формата тега и `MULTIPLE_PACKAGES`:
  - `MULTIPLE_PACKAGES=true: тег обязан быть в форме '{branch}/vX.Y.Z'` — включён режим отдельного образа на ветку, но тег плоский; запушь `{branch}/vX.Y.Z`;
  - `MULTIPLE_PACKAGES выключен: тег обязан быть в форме 'vX.Y.Z'` — тег с префиксом ветки, но режим выключен; либо убери префикс, либо задай repo-переменную `MULTIPLE_PACKAGES=true`
- Логи: Actions → выбрать run → шаг `Create GitHub Release`, `Publish to GitHub Packages`/`Publish to npmjs.org` или `Build & push image`

<!-- DOCGEN:END -->
