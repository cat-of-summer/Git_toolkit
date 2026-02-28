# ci-cd.yml

Workflow деплоя проекта на удалённый сервер. Запускается при `push` в любую ветку.
Окружение (`environment`) выбирается автоматически из имени ветки — правила и секреты берутся из соответствующего Environment в GitHub.

## Шаги

| Шаг | Условие | Описание |
|------|----------|------------|
| `SETUP SSH KEY` | `RSYNC` или `GIT` | Создаёт временный файл SSH-ключа, путь экспортируется через `$GITHUB_ENV` |
| `BUILD` | `BUILD_COMMAND != ''` | `npm ci` + произвольная команда сборки |
| `Cache lftp` | `FTP` | Кеширует `.deb`-пакет `lftp` через `actions/cache@v4` |
| `FTP DEPLOY` | `FTP` | Зеркальный перенос файлов через `lftp mirror -R` |
| `RSYNC DEPLOY` | `RSYNC` | Синхронизация через `rsync` по SSH |
| `GIT DEPLOY` | `GIT` | `git clone` / `git fetch+reset` / `git pull` на сервере через SSH |
| `POST DEPLOY COMMAND` | `DEPLOY_COMMAND != ''` и не `FTP` | Выполняет `DEPLOY_COMMAND` в `DEPLOY_PATH` на сервере |
| `CLEANUP SSH KEY` | `always()` | Удаляет временный файл ключа даже при ошибке предыдущих шагов |

## Переменные (GitHub Environment)

### Variables

| Переменная | Обязательна | По умолчанию | Описание |
|------------|----------|--------------|------------|
| `DEPLOY_METHOD` | ✓ | — | `FTP`, `RSYNC` или `GIT` |
| `DEPLOY_HOST` | ✓ | — | IP или домен сервера |
| `DEPLOY_USER` | ✓ | — | Пользователь SSH / FTP |
| `DEPLOY_PATH` | ✓ | — | Целевой путь на сервере |
| `DEPLOY_LOCAL_DIR` | | `./` | Локальная папка для деплоя |
| `DEPLOY_MIRROR` | | `false` | `true` — удалять лишние файлы на сервере |
| `DEPLOY_PORT` | | `22` | Порт SSH / FTP |
| `BUILD_COMMAND` | | — | Команда сборки. Если пустая — шаг пропускается |
| `DEPLOY_COMMAND` | | — | Команда на сервере после деплоя. Недоступна для FTP |

### Secrets

| Секрет | Описание |
|--------|------------|
| `DEPLOY_KEY` | Приватный SSH-ключ (для RSYNC/GIT) или пароль FTP |

## Особенности методов

### FTP
- Использует `lftp mirror -R`. `DEPLOY_PATH` указывать относительно корня FTP-пользователя.
- `DEPLOY_KEY` — это пароль, не SSH-ключ. `DEPLOY_COMMAND` недоступен.
- `lftp` кешируется через `actions/cache` — повторные запуски устанавливают пакет через `dpkg -i` без сети.

### RSYNC
- SSH-ключ создаётся один раз в `SETUP SSH KEY`, используется в RSYNC и POST DEPLOY COMMAND.
- `DEPLOY_MIRROR=true` добавляет `--delete`.
- SSH-опции: `BatchMode=yes`, `ConnectTimeout=30`, `StrictHostKeyChecking=no`.

### GIT
- Первый деплой: `git clone --branch BRANCH`.
- `DEPLOY_MIRROR=true` → `git fetch + reset --hard`; `false` → `git pull`.
- Сервер должен иметь SSH-доступ к GitHub (Deploy key или ключ пользователя).
