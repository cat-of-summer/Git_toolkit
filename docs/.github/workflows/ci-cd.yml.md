<!-- DOCGEN:START -->
# ci-cd.yml — CI/CD деплой

Подробная документация для workflow `.github/workflows/ci-cd.yml`.

Назначение: универсальный деплой (FTP / RSYNC / GIT) с опциональной сборкой.

Триггеры:
- `push` — любые ветки
- `workflow_dispatch` — ручной запуск с вводом `branch` (environment)

Входные параметры (workflow_dispatch):
- `branch` — имя ветки / environment (тип: environment)

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание | Рекомендация |
|------------|---------|--------------|
| `DEPLOY_METHOD` | Метод деплоя: `FTP`, `RSYNC`, `GIT` | Настрой в Environment |
| `DEPLOY_HOST` | Хост или IP сервера | Обязательно для всех методов |
| `DEPLOY_USER` | SSH/FTP пользователь | — |
| `DEPLOY_PATH` | Целевая директория на сервере | Рекомендуется полный путь |
| `DEPLOY_LOCAL_DIR` | Локальная директория для деплоя | По умолчанию `./` |
| `DEPLOY_MIRROR` | `true` — удалять файлы на сервере | По умолчанию `false` (опасно) |
| `DEPLOY_PORT` | Порт подключения | По умолчанию `22` |
| `BUILD_COMMAND` | Команда сборки (если пусто — шаг пропускается) | Пример: `npm ci && npm run build` |
| `DEPLOY_COMMAND` | Команда на сервере после деплоя (не для FTP) | Пример: `php artisan migrate` |

Secrets:

| Секрет | Описание |
|-------|----------|
| `DEPLOY_KEY` | Для `RSYNC`/`GIT` — приватный SSH-ключ; для `FTP` — пароль | 

---

Порядок шагов (кратко):

1. Checkout репозитория (с нужной веткой).
2. При `RSYNC`/`GIT` — создаётся временный файл с приватным ключом (`TMP_KEY`).
3. Опциональный `BUILD` — запускается, если задан `BUILD_COMMAND`.
4. В зависимости от `DEPLOY_METHOD` выполняется один из деплой-алгоритмов:
	- `FTP` — `lftp mirror -R` (логин: `$DEPLOY_USER,$DEPLOY_KEY`).
	- `RSYNC` — `rsync -e "ssh -i $TMP_KEY ..."` с опцией `--delete` при `DEPLOY_MIRROR=true`.
	- `GIT` — SSH-подключение к серверу, `git clone` или `git pull`/`reset` в зависимости от `DEPLOY_MIRROR`.
5. `POST DEPLOY COMMAND` — выполняется по SSH (не для FTP) при наличии `DEPLOY_COMMAND`.
6. Очистка временного ключа (`CLEANUP SSH KEY`).

Примеры конфигураций:

1) Простая синхронизация через `rsync` (Environment vars):

```
DEPLOY_METHOD=RSYNC
DEPLOY_HOST=example.com
DEPLOY_USER=deploy
DEPLOY_PATH=/var/www/project
DEPLOY_LOCAL_DIR=build/
DEPLOY_MIRROR=true
DEPLOY_PORT=22
BUILD_COMMAND=npm ci && npm run build
DEPLOY_COMMAND=php artisan migrate
```

2) FTP деплой (пароль в `DEPLOY_KEY`):

```
DEPLOY_METHOD=FTP
DEPLOY_HOST=ftp.example.com
DEPLOY_USER=ftpuser
DEPLOY_PATH=./public_html
DEPLOY_KEY=<ftp-password-as-secret>
```

Безопасность и рекомендации:
- Всегда храните приватные ключи в `Secrets` (не в `vars`).
- Не ставьте `DEPLOY_MIRROR=true` без тестирования — это удалит файлы на сервере.
- Тестируйте `BUILD_COMMAND` локально перед пушем.

Отладка:
- Логи workflow показывают stdout/stderr каждого шага (Actions → run).
- Для проверки SSH используйте `ssh -i deploy_key -T deploy@host` локально.
- Для FTP проверьте рабочую директорию клиента, чтобы корректно указать `DEPLOY_PATH`.

<!-- DOCGEN:END -->
