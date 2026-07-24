<!-- DOCGEN:START -->
# ci-cd.yml — сборка, релиз, публикация и деплой в одном файле

Один workflow `.github/workflows/ci-cd.yml` закрывает весь путь от пуша до сервера: собирает
проект (на нескольких ОС), прикрепляет файлы к GitHub Release, публикует пакет (npm / Docker /
Packagist) и деплоит. Всё настраивается **переменными окружения**, менять YAML не нужно.

---

## Быстрый старт

1. Скопируй `ci-cd.yml` в `.github/workflows/` своего проекта.
2. Создай **Environment** с именем своей ветки (Settings → Environments → New environment,
   например `main`).
3. Задай в этом Environment нужные переменные (см. ниже). Минимум для «просто прогонять тесты»:
   ```
   ACTION_TRIGGER=PUSH
   CI_COMMAND=npm test
   ```
4. Запушь — workflow запустится сам.

Ничего лишнего задавать не надо: любая незаданная переменная просто выключает свой шаг, а джоба
становится **серой (skipped)**, а не красной. Красный статус = реальная ошибка команды.

---

## Как задавать переменные и секреты

Всё берётся из **GitHub Environments** (Settings → Environments → выбрать окружение):

- **Variables** — обычные значения (`vars.*`). Все переменные из таблиц ниже — отсюда.
- **Secrets** — чувствительные значения (`secrets.*`): ключи, токены, пароли.

Имя Environment = имя ветки. Для веток со слешами `/` заменяется на `-`: ветке `release/1.x`
соответствует Environment `release-1.x`.

Разные окружения = разные настройки: можно собирать `main` под npm, а `develop` — только тесты.

> **Одно исключение:** `MULTIPLE_PACKAGES` задаётся **на уровне репозитория**
> (Settings → Secrets and variables → Actions → Variables), а не в Environment. Причина — внизу
> страницы.

---

## Переменные

Колонки «ci / release / cd» показывают, какая часть пайплайна переменную читает.

### Что и когда запускать

| Переменная | Область | По умолчанию | Значения / смысл |
|---|---|---|---|
| `ACTION_TRIGGER` | все | `WORKFLOW_DISPATCH` | Когда пайплайн «активен» — см. таблицу ниже. `WORKFLOW_DISPATCH` / `PUSH` / `RELEASE` |

### Сборка и проверки (ci)

| Переменная | Область | По умолчанию | Значения / смысл |
|---|---|---|---|
| `BUILD_COMMAND` | ci | пусто | Команда сборки. Пусто → шаг пропускается. Пример: `npm ci && npm run build` |
| `CI_COMMAND` | ci | пусто | Команда проверок/тестов. Пусто → шаг пропускается. Пример: `npm test` |
| `RUNS_ON` | ci | `ubuntu-latest` | ОС сборки через запятую. `ubuntu-latest,windows-latest,macos-latest` |
| `TOOLCHAIN` | ci | пусто | Языки и версии через запятую: `node:24,python:3.12`. Пусто → берутся предустановленные на runner'е |

### Релиз (release-publish)

| Переменная | Область | По умолчанию | Значения / смысл |
|---|---|---|---|
| `RELEASE_FILES` | release | пусто | Какие файлы прикрепить к Release. Glob'ы через запятую: `dist/*.zip,bin/app-*` |
| `PUBLISH_METHOD` | release | пусто | Куда публиковать после релиза: `npm` / `docker` / `packagist`. Пусто → не публиковать |
| `MULTIPLE_PACKAGES` | release | `false` | **repository-level.** `true` → отдельный пакет/образ на каждую ветку |

### Публикация Docker (при `PUBLISH_METHOD=docker`)

| Переменная | По умолчанию | Смысл |
|---|---|---|
| `DOCKER_REGISTRY` | `ghcr.io` | Реестр |
| `DOCKER_IMAGE` | `<owner>/<repo>` | Имя образа без реестра |
| `DOCKERFILE_PATH` | `./Dockerfile` | Путь к Dockerfile |
| `BUILD_CONTEXT` | `.` | Контекст сборки |
| `DOCKER_USERNAME` | `github.actor` | Логин для нестандартного реестра (для `ghcr.io` не нужен) |
| `DOCKER_BUILD_ARGS` | пусто | Доп. build-args, по одному на строку |

### Публикация Packagist (при `PUBLISH_METHOD=packagist`)

| Переменная | По умолчанию | Смысл |
|---|---|---|
| `PACKAGIST_USERNAME` | — | Имя пользователя Packagist |

### Деплой (cd)

| Переменная | По умолчанию | Смысл |
|---|---|---|
| `DEPLOY_METHOD` | `COMMAND` | `COMMAND` (только SSH-команды) / `FTP` / `RSYNC` / `GIT` |
| `DEPLOY_HOST` | — | Хост сервера. **Обязателен для деплоя** |
| `DEPLOY_USER` | — | Пользователь. **Обязателен** |
| `DEPLOY_PATH` | — | Каталог на сервере. **Обязателен** |
| `DEPLOY_LOCAL_DIR` | `./` | Что заливать: локальный каталог-источник |
| `DEPLOY_MIRROR` | `false` | `false` — ничего не удалять; `true` — удалять убранное в коммитах; `full` — точное зеркало (⚠️ снесёт всё лишнее на сервере). `full` **нельзя** сочетать с `DEPLOY_LAST_COMMITS=true` |
| `DEPLOY_PORT` | `22` | Порт |
| `DEPLOY_LAST_COMMITS` | `false` | `true` → на push деплоить только файлы этого push (выборочно) |
| `BEFORE_DEPLOY_COMMAND` | пусто | Команда по SSH **до** деплоя (не для FTP). Пример: `php artisan down` |
| `AFTER_DEPLOY_COMMAND` | пусто | Команда по SSH **после** деплоя (не для FTP). Пример: `docker compose pull && docker compose up -d` |

Деплой включается, только когда заданы **все четыре**: `DEPLOY_HOST` + `DEPLOY_USER` +
`DEPLOY_PATH` + секрет `DEPLOY_KEY`. Иначе `cd` серая.

### Секреты

| Секрет | Где нужен | Смысл |
|---|---|---|
| `GITHUB_TOKEN` | всегда | Встроенный, задавать не надо. Release, GitHub Packages, `ghcr.io` |
| `DEPLOY_KEY` | cd | Приватный SSH-ключ; для `FTP` — пароль |
| `DOCKER_TOKEN` | docker | Пароль для нестандартного реестра (для `ghcr.io` не нужен) |
| `PAT_TOKEN` | docker | Пробрасывается в образ как build-arg `GH_TOKEN` (если нужен приватный доступ на сборке) |
| `PACKAGIST_API_TOKEN` | packagist | Токен Packagist |

> Токен npmjs **не нужен**: публикация идёт через OIDC trusted publishing (см. настройку внизу).

---

## Что именно выполнится: ACTION_TRIGGER

`ACTION_TRIGGER` задаёт, при каком событии пайплайн «активен».

| `ACTION_TRIGGER` | Событие | ci | release | cd |
|---|---|:--:|:--:|:--:|
| `WORKFLOW_DISPATCH` (по умолчанию) | push ветки/тега | — | — | — |
| `WORKFLOW_DISPATCH` | ручной запуск на ветке | ✔ | — | ✔ |
| `WORKFLOW_DISPATCH` | ручной запуск на теге | ✔ | ✔ | ✔ |
| `PUSH` | push ветки | ✔ | — | ✔ |
| `PUSH` | push тега | ✔ | ✔ | ✔ |
| `RELEASE` | push ветки | — | — | — |
| `RELEASE` | push тега | ✔ | ✔ | ✔ |
| любой | pull request | ✔ | — | — |

Кратко:
- **`WORKFLOW_DISPATCH`** (по умолчанию) — автоматически ничего. Только ручной запуск
  (Actions → Run workflow) даёт ci + cd; релиз — если запущен на теге.
- **`PUSH`** — на каждый push этой ветки идут ci + cd; на push тега добавляется релиз.
- **`RELEASE`** — на push ветки ничего, релиз и деплой только на push тега.
- **pull request** — всегда только ci (чтобы чек разрешал слияние), деплоя нет.

`release` бывает лишь на теге: без тега публиковать нечего. `cd` требует заданных `DEPLOY_*`.

---

## Ручной запуск: входы

Actions → **ci/cd** → **Run workflow**. В **Use workflow from** выбирается ветка **или тег**
(релиз возможен только при выборе тега).

| Вход | Тип | По умолчанию | Что делает |
|---|---|---|---|
| `run_ci` | галочка | вкл | Снять — пропустить сборку и тесты |
| `run_release` | галочка | вкл | Снять — не трогать Release и реестры |
| `run_cd` | галочка | вкл | Снять — не деплоить |
| `environment` | строка | пусто | Задать окружение вручную (переопределяет автоопределение) |
| `runs_on` | строка | пусто | Переопределить `RUNS_ON` на этот запуск |
| `publish_method` | список | пусто | Переопределить `PUBLISH_METHOD` |
| `commits` | строка | пусто | Список коммитов через запятую для выборочного деплоя |
| `deploy_mirror` | список | `false` | `false` / `true` / `full` на этот запуск |

Галочки только **снимают** флаг: если `ACTION_TRIGGER` не разрешает деплой, включённая
`run_cd` его не добавит.

**Переопубликовать тег** (публикация упала / Trusted Publisher настроили позже): Run workflow,
в **Use workflow from** выбрать тег, снять `run_ci` и `run_cd` — пойдёт только релиз и публикация.
Повтор безопасен: уже опубликованные версии пропускаются.

---

## Как настраивается сборка под несколько ОС

`RUNS_ON=ubuntu-latest,windows-latest` → сборка идёт на обеих; в Release попадут артефакты обеих.

В `BUILD_COMMAND` доступны:
- `$RUNNER_OS` — `Linux` / `Windows` / `macOS`;
- `$MATRIX_OS` — `ubuntu-latest` / `windows-latest` / …

Имена результатов **должны различаться по ОС**, иначе они перезапишут друг друга:
```
BUILD_COMMAND=pyinstaller --onefile --name "app-$RUNNER_OS" main.py
RELEASE_FILES=dist/app-*
```

---

## TOOLCHAIN: языки и версии

Формат `<инструмент>:<версия>` через запятую. Версию можно опустить — будет `latest`.

```
TOOLCHAIN=node:24
TOOLCHAIN=node:24,python:3.12
TOOLCHAIN=go:1.23,php:8.3
```

Список **открытый**: node, python, go, php, java, ruby, rust, bun, deno, terraform и сотни
других. Добавить язык = дописать в переменную, YAML не трогать. Пусто → используются версии,
предустановленные на runner'е.

---

## Формат тега

Определяется repository-level переменной `MULTIPLE_PACKAGES`:

- **`false` (по умолчанию)** — тег плоский: `v1.2.3`. Ветку (Environment) workflow определяет
  сам, по коммиту тега.
- **`true`** — тег с явной веткой: `main/v1.2.3`. Ветка берётся из префикса, имя пакета/образа
  получает суффикс (`@owner/name-main`, `owner/repo-main`).

Допустимы `v#`, `v#.#`, `v#.#.#`. Для пакета/образа версия очищается: `v1.2.3` → `1.2.3`.

```
git tag v1.0.0 && git push origin v1.0.0
# либо при MULTIPLE_PACKAGES=true:
git tag main/v1.0.0 && git push origin main/v1.0.0
```

---

## Настройка публикации

### npm (`PUBLISH_METHOD=npm`)

Пакет уходит в **оба** реестра: GitHub Packages (по `GITHUB_TOKEN`, настраивать нечего) и
npmjs.org. Для npmjs токен не нужен, но один раз настрой **Trusted Publisher**:

npmjs.com → Package Settings → Trusted Publisher → GitHub Actions → укажи владельца, репозиторий,
имя workflow-файла **`ci-cd.yml`**, environment оставь пустым.

Требования: имя в `package.json` scoped и совпадает с владельцем (`@owner/name`); **не коммить
`.npmrc` с токенами** (перебивает авторизацию и утекает). Для самого первого публиша нового имени
на npmjs — опубликуй разово вручную токеном, потом включай Trusted Publisher.

### docker (`PUBLISH_METHOD=docker`)

Нужен `Dockerfile`. Собранные файлы уже лежат в контексте сборки, поэтому в образе достаточно
`COPY dist/ /app/` без пересборки. Для `ghcr.io` секреты не нужны.

```
PUBLISH_METHOD=docker
DOCKER_IMAGE=myorg/myapp        # опционально
DOCKERFILE_PATH=./docker/Dockerfile  # опционально
```

### packagist (`PUBLISH_METHOD=packagist`)

```
PUBLISH_METHOD=packagist
PACKAGIST_USERNAME=<user>
# secret: PACKAGIST_API_TOKEN
```

---

## Примеры конфигураций

**Тесты + автодеплой по rsync на каждый push:**
```
ACTION_TRIGGER=PUSH
TOOLCHAIN=node:24
BUILD_COMMAND=npm ci && npm run build
CI_COMMAND=npm test
DEPLOY_METHOD=RSYNC
DEPLOY_HOST=example.com
DEPLOY_USER=deploy
DEPLOY_PATH=/var/www/project
DEPLOY_LOCAL_DIR=dist/
DEPLOY_MIRROR=true
AFTER_DEPLOY_COMMAND=php artisan migrate && php artisan up
# secret: DEPLOY_KEY
```

**Бинарники Linux + Windows в Release:**
```
ACTION_TRIGGER=RELEASE
RUNS_ON=ubuntu-latest,windows-latest
TOOLCHAIN=python:3.12
BUILD_COMMAND=pip install pyinstaller && pyinstaller --onefile --name "app-$RUNNER_OS" main.py
RELEASE_FILES=dist/app-*
```

**Docker-образ из собранного + деплой командой:**
```
ACTION_TRIGGER=RELEASE
TOOLCHAIN=node:24
BUILD_COMMAND=npm ci && npm run build
PUBLISH_METHOD=docker
DEPLOY_METHOD=COMMAND
DEPLOY_HOST=example.com
DEPLOY_USER=deploy
DEPLOY_PATH=/srv/app
AFTER_DEPLOY_COMMAND=docker compose pull && docker compose up -d
# secret: DEPLOY_KEY
```

**Только релиз с публикацией в npm (без деплоя):**
```
ACTION_TRIGGER=RELEASE
TOOLCHAIN=node:24
BUILD_COMMAND=npm ci && npm run build
PUBLISH_METHOD=npm
# DEPLOY_* не заданы → cd серая
```

---

## Если что-то не запускается

- **Всё серое.** Смотри лог джобы `resolve-config`: там печатаются `ACTION_TRIGGER`, событие и
  итоговые `run_ci / run_release / run_cd`. Чаще всего `ACTION_TRIGGER` не тот.
- **`cd` серая при `ACTION_TRIGGER=PUSH`.** Заданы не все из `DEPLOY_HOST/USER/PATH` + секрет
  `DEPLOY_KEY`.
- **`ci` серая.** Не заданы ни `BUILD_COMMAND`, ни `CI_COMMAND`.
- **Ошибка на формате тега.** Тег не соответствует режиму `MULTIPLE_PACKAGES` — в сообщении
  указано, какая форма ожидается.
- **`Branch '...' from tag not found`.** В теге `{branch}/vX.Y.Z` указана несуществующая ветка.
- **Файлы не попали в Release.** Проверь `RELEASE_FILES`; собранных файлов нет в дереве, если
  `ci` не запускалась (`run_ci=false`).
- **Собранное не доехало до сервера.** Скорее всего сработал выборочный режим деплоя — см.
  предупреждение в логе `cd` (деплой результатов сборки работает только в полном режиме).
- **PR не разблокируется.** При нескольких ОС имя чека — `ci (ubuntu-latest)`, а не `ci`;
  обнови требуемые чеки в branch protection.
- **npm `401` (GitHub Packages).** Имя пакета должно быть scoped и совпадать с владельцем; в
  репозитории не должно быть закоммиченного `.npmrc`.
- **npm OIDC падает.** Не настроен Trusted Publisher, либо в нём осталось старое имя файла, либо
  пакета ещё нет на npmjs.
- **«уже есть — пропускаем».** Не ошибка: версия уже опубликована. Нужен новый тег.

---

## Миграция со старой схемы (`ci-cd.yml` + `release.yml`)

Если проект использовал раздельные workflow:

1. **Переименуй переменные Environment:**

   | Было | Стало |
   |---|---|
   | `BUILD_MATRIX=["ubuntu-latest","windows-latest"]` | `RUNS_ON=ubuntu-latest,windows-latest` |
   | `BUILD_ARTIFACT_COMMAND` | `BUILD_COMMAND` |
   | `BUILD_ARTIFACT_PATH` | удалить (каталог задаёт сама команда) |
   | `PYTHON_VERSION=3.12` | `TOOLCHAIN=python:3.12` |
   | `ACTION_TRIGGER=DISPATCH` | `ACTION_TRIGGER=WORKFLOW_DISPATCH` (старое пока принимается с warning) |

2. **npmjs Trusted Publisher** привязан к имени файла: в настройках каждого опубликованного
   пакета замени `release.yml` на `ci-cd.yml`, иначе OIDC-публикация начнёт падать.
3. **`ACTION_TRIGGER` теперь влияет и на релиз, и на деплой** (раньше — только на деплой).
   Проверь, что для нужных веток стоит `PUSH` или `RELEASE`.
4. Удали старые `release.yml` и (если был) `python-build.yml` — их работу делает `ci-cd.yml`.

---
---

# Как это устроено (справочно)

Ниже — внутренняя механика; для настройки проекта читать не обязательно.

## Граф джоб

```
resolve-branch               резолв Environment + тега + суффикса пакета
└─ resolve-config env:<ветка>  vars → run_ci / run_release / run_cd / scope / список ОС
   └─ ci          matrix RUNS_ON, env:<ветка>
      │           BUILD_COMMAND + CI_COMMAND → артефакт workspace-<os>
      ├─ release-publish            RELEASE_FILES → GitHub Release
      │  ├─ npm-publish             PUBLISH_METHOD=npm
      │  ├─ docker-publish          PUBLISH_METHOD=docker
      │  └─ packagist-publish       PUBLISH_METHOD=packagist
      └─ cd       needs: [ci, release-publish, все *-publish]
                  DEPLOY_LOCAL_DIR → сервер
```

Раньше пайплайн был разнесён на `ci-cd.yml` + `release.yml` и склеен ожиданием чужих чеков
(`lewagon/wait-on-check-action`) — это давало только порядок, без передачи файлов. Теперь всё в
одном run: зависимости через `needs:`, собранные файлы едут дальше артефактом.

## Поток файлов

Джоба `ci` выгружает `workspace-<os>` — **весь рабочий каталог** (исходники + результаты сборки).
Нижестоящие джобы делают свой checkout и распаковывают артефакт поверх, получая уже собранное
дерево. Дальше каждая берёт свою выборку: `release-publish` — `RELEASE_FILES`, `cd` —
`DEPLOY_LOCAL_DIR`, docker/npm — весь каталог как контекст.

Отдельной переменной под каталог сборки нет намеренно: путь определяет сама `BUILD_COMMAND`,
поэтому выгружается всё. Артефакт живёт один день (только внутри run). Скрытые файлы
(`.git`, `.github`, `.env`, `.npmrc`) в артефакт **не попадают** — `upload-artifact` их не берёт;
если сборка пишет в скрытый каталог (`.next`, `.output`), направь вывод в обычный (`dist/`).

При нескольких ОС артефакты сливаются; одинаковые пути перезаписываются (для исходников
безвредно, поэтому имена сборочных результатов должны различаться по ОС).

## Правило запуска (джоба resolve-config)

```
PR      = событие pull_request
ACTIVE  = ручной запуск
        | push ветки при ACTION_TRIGGER=PUSH
        | push тега   при ACTION_TRIGGER=PUSH или RELEASE

run_ci      = (PR или ACTIVE) и задана BUILD_COMMAND или CI_COMMAND
run_release = ACTIVE и это тег
run_cd      = ACTIVE и не PR и заданы DEPLOY_HOST + USER + KEY + PATH
```
Галочки ручного запуска применяются сверху и могут только снять флаг. Старое
`ACTION_TRIGGER=DISPATCH` нормализуется в `WORKFLOW_DISPATCH` с предупреждением.

## Почему две служебные джобы, а не одна

`resolve-branch` вычисляет имя Environment и потому **не может** быть к нему привязана. А
переменные окружения читаются только внутри джобы, у которой `environment:` уже указан, — и
недоступны ни в job-level `if:`, ни в `strategy.matrix`. Поэтому чтение `vars` и расчёт флагов
вынесены в отдельную `resolve-config` с `environment: <ветка>`. По той же причине
`MULTIPLE_PACKAGES` обязана быть repository-level: она читается в `resolve-branch` до выбора
Environment.

## Почему ветку нельзя «просто узнать» из тега

Git-тег указывает на коммит, а не на ветку — имя ветки в теге не хранится. Если коммит достижим
из нескольких веток, тег принадлежит им всем. Автоопределение (`git branch -r --contains`) — это
догадка; единственный надёжный способ привязать релиз к ветке — назвать её в теге
(`MULTIPLE_PACKAGES=true`) или задать входом `environment` при ручном запуске.

## Установка TOOLCHAIN

Ставит [mise](https://mise.jdx.dev) (MIT, бесплатный). Воркфлоу скачивает один бинарник mise
(для каждой ОС свой, архивов и `unzip` не требуется), выполняет `mise use --global <tools>` и
кладёт в `PATH` и шимы, и реальные каталоги `bin` (шимов одних мало: на Windows `command -v`
их не находит). Установленные инструменты кешируются между запусками по ОС и составу `TOOLCHAIN`.

Источник бинаря mise — по убыванию: CDN mise (`mise.jdx.dev`, без лимитов) → endpoint `VERSION` →
GitHub API → вшитая в workflow версия-фолбэк. Специальной переменной для версии mise нет —
цепочка фолбэков подбирает её сама.

> В `npm-publish` node ставится через `actions/setup-node` (версия — из `TOOLCHAIN`): он нужен
> ради `.npmrc` c registry-url и scope, чего шимы не делают.

## flatten имён файлов Release

Чтобы файлы из разных каталогов не конфликтовали именами, `/` в относительном пути заменяется
на `__`:
```
dist/linux/app        →  linux__app
dist/windows/app.exe  →  windows__app.exe
```

## Порядок в npm-publish

GitHub Packages — обязательный, первым, по `GITHUB_TOKEN`; npmjs — опциональный
(`continue-on-error`), вторым, по OIDC. Между ними переустанавливается `setup-node` со `scope`:
scoped-registry приоритетнее флага `--registry`, без перезаписи публикация в npmjs ушла бы в
GitHub Packages и дала `401`. Перед каждой публикацией — идемпотентная проверка `npm view`.
Шаг `Drop committed .npmrc` удаляет закоммиченный project-level `.npmrc`, который иначе перебил бы
`NODE_AUTH_TOKEN`.

## Режимы деплоя

- `COMMAND` — файлы не передаются, только `BEFORE_/AFTER_DEPLOY_COMMAND` по SSH.
- `FTP` / `RSYNC` / `GIT` — заливка кодовой базы; `RSYNC`/`FTP` повторяют операцию при сбоях.

Режим (`SCOPE`) выбирается автоматически: `COMMAND` → `none`; push тега или push без
`DEPLOY_LAST_COMMITS` → `full` (весь `DEPLOY_LOCAL_DIR`); push с `DEPLOY_LAST_COMMITS=true` или
ручной запуск с `commits` → `selective` (только изменённые файлы).

> **Результаты сборки деплоятся только при полном режиме (`full`-scope).** В выборочном список
> файлов строится из git-истории, а собранных файлов в git нет; к тому же дерево переводится на
> другой коммит. Поэтому в выборочном режиме артефакт не распаковывается, а вместо него
> выполняется `BUILD_COMMAND` на месте (джоба предупреждает в логе).

Несовместимые сочетания (джоба падает с понятной ошибкой):
- `GIT` + выборочный режим — `GIT` тянет ветку целиком, а не подмножество файлов.
- `DEPLOY_MIRROR=full` + выборочный режим — `full` означает точное зеркало всего дерева, что
  противоречит идее «залить только изменённые файлы»; для очистки убранного в коммитах есть
  `DEPLOY_MIRROR=true`.

`.git` и `.github` при заливке никогда не передаются и не удаляются.

## Порядок и ожидание в cd

`cd` зависит от `ci`, `release-publish` и всех `*-publish`, поэтому стартует строго после них —
`docker compose pull` не выполнится раньше, чем образ запушен. Пропущенные зависимости (не-тег →
релизные джобы серые) не блокируют за счёт условия `!failure() && !cancelled()`.

<!-- DOCGEN:END -->
