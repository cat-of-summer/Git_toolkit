# remote-grabber.yml

Описание рабочего процесса, который тянет файлы с удалённого сервера на GitHub Actions runner,
фильтрует их по `.gitignore`, создаёт новую ветку и пушит изменения в репозиторий.

Файл workflow: [.github/workflows/sync-from-server.yml](.github/workflows/sync-from-server.yml)

**Кратко:**
- Получает список файлов на сервере (без скачивания).
- Пропускает пути, указанные в `.gitignore` проекта (через `git check-ignore`).
- Скачивает только отфильтрованные файлы (rsync или lftp).
- Создаёт ветку `sync/<branch>_YYYY-MM-DD_HH-MM-SS`, коммитит и пушит.

Назначение
-- Поднять изменения, которые появились непосредственно на продакшн-сервере, в git-репозиторий.
-- Поддерживает случаи, когда на сервере нет git (например, shared/FTP-хостинг).

Как это работает
1. Checkout репозитория (`actions/checkout@v4`) с `fetch-depth: 0`.
2. Получение списка файлов на сервере:
	- для `RSYNC` — через `ssh` и `find`;
	- для `FTP` — через `lftp find`.
3. Добавление префикса `DEPLOY_LOCAL_DIR` при необходимости и проверка списка через `git check-ignore`.
4. Скачивание только тех файлов, которые не игнорируются `.gitignore`:
	- `rsync --files-from=...` или lftp-скрипт для отдельных `get`.
5. (Опционально) Удаление локальных файлов, которые больше не присутствуют на сервере, если `delete_removed=true`.
6. Создание ветки `sync/...`, коммит и `git push`.

Входные параметры (`workflow_dispatch` inputs)
- `branch` — окружение / ветка (тип `environment`).
- `delete_removed` — `true|false` (по умолчанию `false`) — удалять ли локальные файлы, отсутствующие на сервере.

Переменные окружения (env)
- `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, `DEPLOY_LOCAL_DIR`, `DEPLOY_KEY`, `DEPLOY_PORT`, `DEPLOY_METHOD` — берутся из `vars` / `secrets` (см. примеры в [.github/workflows/ci-cd.yml](.github/workflows/ci-cd.yml)).

Особенности и рекомендации
- Фильтрация по `.gitignore` выполняется локально на раннере — это позволяет не скачивать тяжёлые игнорируемые директории (например, `node_modules`, `upload/`, `bitrix/cache`).
- Для крупных проектов сначала выполняйте dry-run на тестовом окружении.
- `DELETE_REMOVED=true` аккуратно: файлы удаляются из рабочей копии перед коммитом.
- Для FTP используется `lftp`; кэш пакета настроен аналогично другим workflow.

Примеры запуска
- Вручную через GitHub Actions → **Run workflow** → выбрать `branch` и опцию `delete_removed`.

Отладка
- Логи шага `LIST SERVER FILES` покажут количество найденных файлов.
- Если `to_download.txt` пуст — в логах будет предупреждение `No files to download after applying .gitignore filter`.

Безопасность
- Ключи SSH передаются через секрет `DEPLOY_KEY`. Файл ключа создаётся временно и удаляется в конце.

Если нужно — могу добавить раздел «Примеры exclude-паттернов» или шаблон для CI-переменных.
