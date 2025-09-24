README — CI/CD (GitHub Actions)
================================

Назначение
---------
Это руководство описывает, как быстро и безопасно развернуть единый workflow GitHub Actions (файл: .github/workflows/workflow.yml).
Workflow поддерживает: BUILD (опционально) и три варианта деплоя:
  - FTP (mirror через lftp)
  - RSYNC (через ssh)
  - GIT (clone / git pull на сервере)
Параметры подаются через Repository Variables и Secrets. Этот файл даёт пошаговую инструкцию по инициализации репозитория, применению .gitignore и настройке секретов/переменных в GitHub.

Быстрая настройка — шаг за шагом
--------------------------------
1. Инициализация git-репозитория (если ещё не создан)
   git init
   git checkout -b main

2. Положить в корень проекта нужные файлы:
   .github/workflows/workflow.yml
   .gitignore
   - остальные исходники проекта (themes, src, local и т.п.)

3. Добавить .gitignore и закоммитить его
   git add .gitignore
   git commit -m "Add .gitignore"

4. Пересчитать индекс (если в репозитории сейчас есть файлы, которые должны быть игнорированы)
   Универсально (пересобирает весь индекс — убедись, что нет незакоммиченных изменений):
   git rm -r --cached .
   git add .
   git commit -m "Rebuild index according to updated .gitignore"

5. Создать пустой репозиторий на GitHub (через UI или gh cli), связать и запушить
   git remote add origin ...
   git push -u origin ...

Настройка Variables и Secrets в GitHub
-------------------------------------
Открой: Settings → Environments

**Variables (видимые в Actions):**
  - DEPLOY_METHOD — тип деплоя: FTP, RSYNC, GIT
  - DEPLOY_HOST — IP или доменное имя сервера
  - DEPLOY_USER — пользователь для подключения (ssh/ftp)
  - DEPLOY_PATH — целевой путь на сервере (ВАЖНО: для FTP — путь от корня FTP-пользователя)
  - DEPLOY_LOCAL_DIR — локальная папка в репозитории для деплоя (по умолчанию ./, из этой папки будут переносится файлы)
  - DEPLOY_MIRROR — true/false (если true — включается удаление удалённых файлов) (по умолчанию true)
  - DEPLOY_PORT — порт SSH/FTP (по умолчанию 22)
  - BUILD_COMMAND — команда сборки (если пусто — шаг BUILD пропускается)

**Secrets (хранятся в секции Secrets):**
  - DEPLOY_KEY — приватный SSH-ключ (или пароль для FTP).
    - Для RSYNC / GIT: это приватный SSH-ключ, публичную часть которого кладём в ~/.ssh/authorized_keys на сервере.
    - Для FTP: пароль от пользователя FTP.

Особенности по методам деплоя
----------------------------
1. FTP (lftp)
   - Workflow использует lftp mirror -R для отправки файлов.
   - **ВАЖНО:** DEPLOY_PATH указывать относительно корня FTP-пользователя. Если при подключении вы попадаете в /home/ftpuser/www/, то DEPLOY_PATH = ./ или www/your_site.
   - При использовании --delete (DEPLOY_MIRROR=true) включает удаление на стороне сервера. Используй осторожно: сначала тестируй с DEPLOY_MIRROR=false.

2. RSYNC (ssh)
   - Workflow генерирует временный файл с приватным ключом, запускает rsync с опцией -e "ssh -i TMP_KEY -o StrictHostKeyChecking=no -p DEPLOY_PORT".
   - При использовании --delete (DEPLOY_MIRROR=true) rsync будет удалять удалённые файлы, которых нет локально.

3. GIT
   - Workflow подключается по SSH, выполняет git clone / git fetch+reset / git pull на сервере.
   - Если DEPLOY_MIRROR=true выполняет команду git fetch+reset origin, если DEPLOY_MIRROR=false, тогда выполняет команду git pull origin.
   - Для этого сервер должен иметь доступ к GitHub: у DEPLOY_USER на сервере настроен SSH-ключ (публичный ключ добавлен в GitHub как Deploy key или в аккаунт с доступом к репо).

Генерация SSH-ключей (коротко)
-----------------------------
1. На локальной машине:
   ssh-keygen -t rsa -b 4096 -C "deploy@project" -f deploy_key
   # deploy_key -> приватный ключ (сохранить как DEPLOY_KEY в Secrets)
   # deploy_key.pub -> публичный ключ (добавить в server: ~/.ssh/authorized_keys для DEPLOY_USER)

2. Для GIT-deploy можно генерировать ключи прямо на сервере и добавлять публичный ключ в GitHub как Deploy key.

Тестирование и отладка
---------------------
- Используй тестовую ветку (например, ci-test) для безопасной проверки.
- Логи запуска доступны в Actions → выбери run → смотри stdout/stderr шагов.
- Отдельно тестируй BUILD-команду локально (npm ci && npm run build), чтобы избежать неожиданных ошибок на runner.
- Для FTP: подключись через FTP-клиент и посмотри корневую папку при коннекте, чтобы корректно указать DEPLOY_PATH.

Пояснение ENV-переменных (кратко)
---------------------------------
- DEPLOY_HOST — адрес сервера
- DEPLOY_USER — пользователь для подключения
- DEPLOY_PATH — целевая папка на сервере (для FTP — от корня FTP-юзера)
- DEPLOY_LOCAL_DIR — локальная папка в репозитории для деплоя (по умолчанию ./, из этой папки будут переносится файлы)
- DEPLOY_KEY — приватный SSH-ключ или пароль для FTP (секрет)
- DEPLOY_MIRROR — true/false, включение удаления удалённых файлов (true по умолчанию)
- DEPLOY_PORT — порт (22 по умолчанию)
- DEPLOY_METHOD — FTP | RSYNC | GIT
- BUILD_COMMAND — команда сборки (если пусто — шаг BUILD пропускается)

Команды для быстрых операций (copy/paste)
----------------------------------------
# Инициализация и пуш в новый репозиторий
git init
git checkout -b main
git add .
git commit -m "Initial commit — add CI/CD workflow"
git remote add origin ...
git push -u origin ...

# Добавление secret DEPLOY_KEY (через gh CLI)
gh secret set DEPLOY_KEY --body-file ./deploy_key

# Обновление .gitignore и пересборка индекса (если требуется)
git add .gitignore
git commit -m "Update .gitignore."
git rm -r --cached .
git add .
git commit -m "Rebuild index according to updated .gitignore"

# Создание пользователя на удаленной машине для выполнения GIT-команд в таком выборе CI/CD
1. Создаем пользователя:
   - На сервере, под root, создаём нового пользователя (например, deploy):
      adduser deploy
   - Или (в Ubuntu/Debian c автоматическим созданием домашней папки):
      useradd -m -s /bin/bash deploy
2. Добавляем пользователя в группу, если нужно права на web-папку (например, www-data):
   usermod -aG www-data deploy
3. Даем права на директорию проекта:
   - Замени /var/www/project на твой $DEPLOY_PATH
      chown -R deploy:www-data /var/www/project
      chmod -R 775 /var/www/project
4. Генерируем SSH-ключ от имени пользователя deploy:
   su - deploy
   ssh-keygen -t rsa -b 4096 -C "git-deploy"
   - Публичный ключ (~/.ssh/id_rsa.pub) добавь в GitHub → Deploy Keys для репозитория.
5. Проверяем доступ:
   su - deploy
   ssh -T git@github.com
   - Должно вернуться сообщение от GitHub («Hi USERNAME!»).

Troubleshooting — типичные ошибки и решения
-----------------------------------------
- Ошибка доступа по SSH при rsync/ssh: проверь ключи, права на файл ключа (chmod 600), и что публичный ключ добавлен в authorized_keys.
- lftp не может подключиться: проверь правильность хоста/порта, FTP-пользователя и пароля; проверь рабочую директорию FTP (для корректного DEPLOY_PATH).
- git clone на сервере не проходит: убедись, что серверный пользователь имеет доступ к GitHub (Deploy key или ключ пользователя).
- BUILD-ошибки: запускай BUILD локально и проверяй, чтобы все зависимости корректно устанавливали и скрипты работали в headless runner.
