<!-- DOCGEN:START -->
# ci-cd.yml — единый CI/CD пайплайн

Подробная документация для workflow `.github/workflows/ci-cd.yml`. Один файл, четыре job:

1. **`get-branch`** — определяет GitHub Environment (= имя ветки). Для тегов ветка ищется по
   коммиту (`git branch -r --contains`), как в `release.yml`; иначе берётся `github.ref_name`.
2. **`gate`** — читает переменные окружения (их **нельзя** прочитать в `if:` на уровне job) и
   вычисляет флаги `run_ci` / `run_cd` / `scope` / `commits`.
3. **`ci`** — сборка/проверки. Запускается, если `run_ci=true`; иначе job **пропускается (серый)**.
4. **`cd`** — деплой (`COMMAND` / `FTP` / `RSYNC` / `GIT`). `needs: ci`; запускается, если
   `run_cd=true` и `ci` не упал; иначе **пропускается (серый)**.

Ключевая идея статусов: когда деплоить/собирать нечего — job **серый/skipped**, а не «зелёный
успешный деплой». Реальные ошибки шагов — **красный**.

Триггеры:
- `push` — и обычные пуши в ветку, и пуши тега вида `v*` (`tags: ['v*']`); внутри job `gate`
  они различаются через `github.ref_type` (`branch` / `tag`). Тег нужен для
  `ACTION_TRIGGER=RELEASE`, как в `release.yml`.
- `pull_request` — выполняется только `ci` (деплоя нет).
- `workflow_dispatch` — ручной запуск (выполняется всегда).

Запуск управляется переменной `ACTION_TRIGGER` (на уровне Environment, т.е. **per-branch**):

| `ACTION_TRIGGER` | Когда срабатывает автоматически |
|------------------|---------------------------------|
| `DISPATCH` (по умолчанию) | Никогда автоматически — только ручной `workflow_dispatch` |
| `PUSH` | На каждый push этой ветки |
| `RELEASE` | При пуше тега на коммите этой ветки, **строго после** `release.yml` |

Входные параметры (`workflow_dispatch`):
- `commits` — список коммитов через запятую (необязательный; пусто = вся история).
- `deploy_mirror` — `false/true/full` (по умолчанию `false`): `false` — ничего не удалять; `true` —
  удалять только файлы, убранные в задеплоенных коммитах (selective); `full` — точная зеркальная
  синхронизация — снесёт нетрекаемые файлы на сервере.

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание | Рекомендация |
|------------|---------|--------------|
| `ACTION_TRIGGER` | `DISPATCH` (default) / `PUSH` / `RELEASE` — когда запускать пайплайн | Задаётся per-branch |
| `BUILD_COMMAND` | Команда сборки (`ci` и `cd`); если пусто — шаг пропускается | Пример: `npm ci && npm run build` |
| `CI_COMMAND` | Команда проверок в job `ci`; если пусто — шаг пропускается | Пример: `npm test` |
| `DEPLOY_METHOD` | Метод деплоя: `COMMAND` (default) / `FTP` / `RSYNC` / `GIT` | См. ниже |
| `DEPLOY_HOST` | Хост или IP сервера | Обязательно для деплоя |
| `DEPLOY_USER` | SSH/FTP пользователь | Обязательно для деплоя |
| `DEPLOY_PATH` | Целевая директория на сервере | Обязательно для деплоя |
| `DEPLOY_LOCAL_DIR` | Локальная директория для деплоя | По умолчанию `./` |
| `DEPLOY_MIRROR` | `false`/`true`/`full` — режим удаления файлов на сервере (см. ниже) | По умолчанию `false`; `full` опасен |
| `DEPLOY_PORT` | Порт подключения | По умолчанию `22` |
| `DEPLOY_LAST_COMMITS` | `true` — при push-деплое брать только коммиты этого push (выборочный режим) | По умолчанию `false` |
| `BEFORE_DEPLOY_COMMAND` | Команда на сервере **до** деплоя по SSH (не для FTP) | Пример: `php artisan down` |
| `AFTER_DEPLOY_COMMAND` | Команда на сервере **после** деплоя по SSH (не для FTP) | Пример: `php artisan migrate` |

Режимы `DEPLOY_MIRROR`:
- `false` — ничего не удалять.
- `true` — удалять только файлы, убранные в задеплоенных коммитах (выборочная очистка).
- `full` — точная зеркальная синхронизация: `rsync`/`lftp mirror --delete` или `git reset --hard` —
  снесёт на сервере вообще все нетрекаемые файлы, не только из коммитов.

Secrets:

| Секрет | Описание |
|-------|----------|
| `DEPLOY_KEY` | Для `RSYNC`/`GIT`/`COMMAND` — приватный SSH-ключ; для `FTP` — пароль |

«Данные для деплоя есть» = заданы все `DEPLOY_HOST` + `DEPLOY_USER` + `DEPLOY_KEY` + `DEPLOY_PATH`.
Если чего-то нет → `cd` **пропускается (серый)**, это не ошибка. На ручном запуске
`DEPLOY_MIRROR` берётся из входа `deploy_mirror`; на `push` — из `vars.DEPLOY_MIRROR`.

---

## Методы деплоя

- **`COMMAND`** (по умолчанию) — не передаёт файлы; просто подключается по SSH и выполняет
  `BEFORE_DEPLOY_COMMAND`, затем `AFTER_DEPLOY_COMMAND`. Типовой кейс: `docker compose pull && docker compose up -d`.
- **`FTP` / `RSYNC` / `GIT`** — заливка кодовой базы (полное зеркало или выборочно).
- **`RSYNC`** при сетевом сбое повторяет каждую операцию передачи до 3 раз с паузой 5с.

Режим деплоя (`SCOPE`) определяется триггером и переменными:

| Триггер | `DEPLOY_METHOD` | `DEPLOY_LAST_COMMITS` | Режим | Источник файлов |
|---------|-----------------|-----------------------|-------|-----------------|
| любой | `COMMAND` | — | none (без файлов) | только команды по SSH |
| `pull_request` | — | — | деплоя нет (только `ci`) | — |
| `push` (`PUSH`) | FTP/RSYNC/GIT | `false` | полное зеркало | весь `DEPLOY_LOCAL_DIR` |
| `push` (`PUSH`) | FTP/RSYNC | `true` | выборочный | коммиты текущего push |
| `push` тег (`RELEASE`) | FTP/RSYNC/GIT | — | полное зеркало | весь `DEPLOY_LOCAL_DIR` |
| `workflow_dispatch` | FTP/RSYNC | — | выборочный | вход `commits` (пусто = вся история) |

- ⚠️ `GIT` несовместим с выборочным режимом (он подтягивает ветку целиком) — `cd` завершается ошибкой (🔴).

---

## Модель статусов

`get-branch` и `gate` всегда зелёные; реальный статус несут `ci` и `cd`.

Гейт `ci`/`cd`:

| event | ref_type | `ACTION_TRIGGER` | ci | cd |
|---|---|---|---|---|
| push | branch | `PUSH` | run | run |
| push | branch | `DISPATCH`/`RELEASE` | серый | серый |
| push | tag | `RELEASE` | run | run (ждёт `release.yml`) |
| push | tag | `DISPATCH`/`PUSH` | серый | серый |
| workflow_dispatch | — | любой | run | run |
| pull_request | — | — | run | серый |

Внутри `ci`: есть `BUILD_COMMAND`/`CI_COMMAND` → выполняет (оба, если заданы), 🟢; нет ни одной →
серый; ошибка команды → 🔴 (тогда `cd` не запускается).

Внутри `cd` (`run_cd=true`): `COMMAND` → SSH + before/after, 🟢/🔴; `FTP`/`RSYNC`/`GIT` → деплой по
scope, 🟢/🔴; `GIT`+selective → 🔴; ошибка деплоя → 🔴.

---

## Связка с `release.yml` (ACTION_TRIGGER=RELEASE)

При пуше тега `v*` запускаются оба workflow, и связь между ними двусторонняя:

1. `ci-cd.yml` (эта job `ci`) собирает и проверяет код.
2. `release.yml` (job `await-ci`) сам дожидается зелёного чека `ci` из `ci-cd.yml` и только
   после этого резолвит тег/ветку и запускает релиз.
3. `release.yml` собирает и пушит артефакт (npm-пакет / Docker-образ).
4. `cd` (эта job) ждёт окончания `release.yml` через `lewagon/wait-on-check-action`
   (чеки `release` / `npm-publish` / `docker-publish`) и только потом деплоит.

Итоговая цепочка при пуше тега: `ci` → `release.yml` (`await-ci` → `prepare` → `release` →
`npm-publish`/`docker-publish`) → `cd`.

- Если `ci` **упал** — `release.yml` не публикует ничего (`await-ci` не пропускает дальше).
- Если `release.yml` **упал** — ожидание в `cd` завершается неуспехом, `cd` → 🔴 (деплой над сломанным релизом бессмыслен).
- Если `ci-cd.yml`/`release.yml` не настроен или нужного чека нет — соответствующая сторона не ждёт
  (`fail-on-no-checks: false`) и продолжает сама по себе.

---

## Шаги job `cd` (кратко)

1. `Wait for release workflow` — только для тега; ждёт `release.yml`.
2. `Validate deploy mode` — проверка `GIT`+selective (→ 🔴).
3. `Setup SSH key` — при `RSYNC`/`GIT`/`COMMAND` (временный `TMP_KEY`).
4. `Checkout` + (выборочный) `Resolve REF_COMMIT` / `Resolve changed files` — при `SCOPE != none`.
5. `Build` — если задан `BUILD_COMMAND` и `SCOPE != none`.
6. `Before deploy command` — по SSH (не для FTP).
7. Деплой по методу/режиму (`FTP` / `RSYNC` / `GIT`; для `COMMAND` файловых шагов нет).
8. `After deploy command` — по SSH (не для FTP).
9. `Cleanup SSH key`.

Примеры конфигураций (Environment vars):

1) Автодеплой при пуше через `rsync`, с CI-проверками:

```
ACTION_TRIGGER=PUSH
CI_COMMAND=npm test
DEPLOY_METHOD=RSYNC
DEPLOY_HOST=example.com
DEPLOY_USER=deploy
DEPLOY_PATH=/var/www/project
DEPLOY_LOCAL_DIR=build/
DEPLOY_MIRROR=true
BUILD_COMMAND=npm ci && npm run build
BEFORE_DEPLOY_COMMAND=php artisan down
AFTER_DEPLOY_COMMAND=php artisan migrate && php artisan up
```

2) Деплой Docker-образа при релизе (только команды на сервере):

```
ACTION_TRIGGER=RELEASE
DEPLOY_METHOD=COMMAND
DEPLOY_HOST=example.com
DEPLOY_USER=deploy
DEPLOY_PATH=/srv/app
AFTER_DEPLOY_COMMAND=docker compose pull && docker compose up -d
# DEPLOY_KEY — SSH-ключ как secret
# в release.yml: PUBLISH_METHOD=docker
```

3) FTP, выборочный деплой указанных коммитов (ручной запуск):

```
DEPLOY_METHOD=FTP
DEPLOY_HOST=ftp.example.com
DEPLOY_USER=ftpuser
DEPLOY_PATH=./public_html
DEPLOY_KEY=<ftp-password-as-secret>

# workflow_dispatch:
commits=abc123,def456,7890ab
deploy_mirror=true
```

4) Только CI (без деплоя): `ACTION_TRIGGER=PUSH`, задайте `CI_COMMAND`/`BUILD_COMMAND` и не
   задавайте данные деплоя — `cd` пропустится (серый).

Безопасность и рекомендации:
- `DEPLOY_MIRROR=full` — самый опасный режим: снесёт на сервере все нетрекаемые файлы
  (`rsync`/`lftp --delete`, `git reset --hard`). Используйте только если сервер должен быть
  точным зеркалом репозитория.
- `DEPLOY_MIRROR=true` безопаснее — удаляет только файлы, убранные в задеплоенных коммитах.
- Для больших наборов коммитов сначала протестируйте на тестовом окружении.
- Тестируйте `BUILD_COMMAND` / `CI_COMMAND` локально перед пушем.

Отладка:
- `gate` логирует `run_ci / run_cd / method / scope` — отсюда понятно, почему `ci`/`cd` серые.
- Если `cd` не запустился — проверьте `ACTION_TRIGGER` для нужного окружения и наличие `DEPLOY_HOST/USER/KEY/PATH`.
- При `RELEASE` смотрите лог шага `Wait for release workflow`: дождался ли он `release.yml`.

<!-- DOCGEN:END -->
