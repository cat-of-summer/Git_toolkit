<!-- DOCGEN:START -->
# cd.yml — единый CI/CD пайплайн

Подробная документация для workflow `.github/workflows/cd.yml`. Один файл, два job:

1. **`ci`** — сборка/проверки. Запускается всегда (push, pull_request, workflow_dispatch). Если ни `BUILD_COMMAND`, ни `CI_COMMAND` не заданы — проходит как успех (gate открыт).
2. **`cd`** — деплой (FTP / RSYNC / GIT). `needs: ci` — выполняется только после **успешного** `ci`. Не запускается на `pull_request`. Пропускается, если не задан `DEPLOY_METHOD`.

Назначение деплоя: универсальный деплой с двумя режимами — **полное зеркало** (весь каталог) и **выборочный** (только файлы из указанных коммитов).

Триггеры:
- `push` — любые ветки.
- `pull_request` — выполняется только `ci` (деплоя нет).
- `workflow_dispatch` — ручной запуск (выполняются оба job).

Входные параметры (`workflow_dispatch`):
- `commits` — список коммитов через запятую (необязательный; пусто = вся история).
- `deploy_mirror` — `true/false` (по умолчанию `true`) — удалять на сервере файлы, отсутствующие в `REF_COMMIT`.

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание | Рекомендация |
|------------|---------|--------------|
| `BUILD_COMMAND` | Команда сборки (`ci` и `cd`); если пусто — шаг пропускается | Пример: `npm ci && npm run build` |
| `CI_COMMAND` | Команда проверок в job `ci`; если пусто — шаг пропускается | Пример: `npm test` |
| `DEPLOY_METHOD` | Метод деплоя: `FTP`, `RSYNC`, `GIT`. Пусто → job `cd` пропускается | Настрой в Environment |
| `DEPLOY_HOST` | Хост или IP сервера | Обязательно для всех методов |
| `DEPLOY_USER` | SSH/FTP пользователь | — |
| `DEPLOY_PATH` | Целевая директория на сервере | Рекомендуется полный путь |
| `DEPLOY_LOCAL_DIR` | Локальная директория для деплоя | По умолчанию `./` |
| `DEPLOY_MIRROR` | `true` — удалять файлы на сервере | По умолчанию `false` (опасно) |
| `DEPLOY_PORT` | Порт подключения | По умолчанию `22` |
| `DEPLOY_ON_PUSH` | `true` — деплоить при `push`; иначе `cd` сразу завершается | По умолчанию `false` |
| `DEPLOY_LAST_COMMITS` | `true` — при push-деплое брать только коммиты этого push (выборочный режим) | По умолчанию `false` |
| `BEFORE_DEPLOY_COMMAND` | Команда на сервере **до** деплоя по SSH (не для FTP) | Пример: `php artisan down` |
| `AFTER_DEPLOY_COMMAND` | Команда на сервере **после** деплоя по SSH (не для FTP) | Пример: `php artisan migrate` |

Secrets:

| Секрет | Описание |
|-------|----------|
| `DEPLOY_KEY` | Для `RSYNC`/`GIT` — приватный SSH-ключ; для `FTP` — пароль |

На ручном запуске `DEPLOY_MIRROR` берётся из входа `deploy_mirror`; на `push` — из `vars.DEPLOY_MIRROR`.

---

Режим деплоя определяется триггером и переменными:

| Триггер | `DEPLOY_ON_PUSH` | `DEPLOY_LAST_COMMITS` | Режим | Источник файлов |
|---------|------------------|-----------------------|-------|-----------------|
| `pull_request` | — | — | деплоя нет (только `ci`) | — |
| `push` | `false` | — | пропуск (ничего) | — |
| `push` | `true` | `false` | полное зеркало | весь `DEPLOY_LOCAL_DIR` |
| `push` | `true` | `true` | выборочный | коммиты текущего push |
| `workflow_dispatch` | — | — | выборочный | вход `commits` (пусто = вся история) |

- **Полное зеркало** — заливает весь `DEPLOY_LOCAL_DIR`: FTP `lftp mirror -R`, RSYNC всей папки или GIT `clone`/`pull` ветки на сервере. Методы: `FTP` / `RSYNC` / `GIT`.
- **Выборочный** — резолвит `REF_COMMIT`, собирает списки `files_upload.txt` / `files_delete.txt` и заливает только их. Методы: `FTP` / `RSYNC`.
- ⚠️ `GIT` несовместим с выборочным режимом (он подтягивает ветку целиком) — в этом случае `cd` завершается с ошибкой.

Шаги job `cd` (кратко):

1. `Resolve deployment mode` — определяет режим (`SKIP_DEPLOY` / `SCOPE` / `COMMITS`); пустой `DEPLOY_METHOD` → пропуск.
2. `Checkout` (полная история — нужна для выборочного режима).
3. `Setup SSH key` — при `RSYNC`/`GIT` (временный `TMP_KEY`).
4. Выборочный режим: `Resolve REF_COMMIT` + `Resolve changed files`.
5. `Build` — если задан `BUILD_COMMAND`.
6. `Before deploy command` — по SSH (не для FTP) при наличии `BEFORE_DEPLOY_COMMAND`.
7. Деплой по методу и режиму: `FTP deploy (full mirror)` / `FTP deploy (selective)` / `RSYNC deploy (full)` / `RSYNC deploy (selective)` / `GIT deploy`.
8. `After deploy command` — по SSH (не для FTP) при наличии `AFTER_DEPLOY_COMMAND`.
9. `Cleanup SSH key` — очистка временного ключа.

Примеры конфигураций:

1) Автодеплой при пуше через `rsync`, с CI-проверками (Environment vars):

```
CI_COMMAND=npm test
DEPLOY_METHOD=RSYNC
DEPLOY_HOST=example.com
DEPLOY_USER=deploy
DEPLOY_PATH=/var/www/project
DEPLOY_LOCAL_DIR=build/
DEPLOY_MIRROR=true
DEPLOY_PORT=22
DEPLOY_ON_PUSH=true
BUILD_COMMAND=npm ci && npm run build
BEFORE_DEPLOY_COMMAND=php artisan down
AFTER_DEPLOY_COMMAND=php artisan migrate && php artisan up
```

2) FTP, выборочный деплой указанных коммитов (ручной запуск):

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

3) Только CI (без деплоя): задайте `CI_COMMAND`/`BUILD_COMMAND` и не задавайте `DEPLOY_METHOD` — `cd` пропустится.

Безопасность и рекомендации:
- Всегда храните приватные ключи в `Secrets` (не в `vars`).
- Не ставьте `DEPLOY_MIRROR=true` без тестирования — это удалит файлы на сервере.
- Для больших наборов коммитов сначала протестируйте на тестовом окружении.
- Тестируйте `BUILD_COMMAND` / `CI_COMMAND` локально перед пушем.

Отладка:
- `Resolve deployment mode` логирует итоговые `event / scope / method / commits`.
- В выборочном режиме `Resolve changed files` показывает списки на загрузку и удаление; пустой набор — предупреждение без ошибки.
- Если `cd` не запустился — проверьте, что `ci` завершился успешно (`cd` зависит от него через `needs`).

<!-- DOCGEN:END -->
