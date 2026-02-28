<!-- DOCGEN:START -->
# GitHub_workflows

## Папки

- [.github](.github/)
- [.gitignore](.gitignore/)

<!-- DOCGEN:END -->

# CI/CD — GitHub Actions

Единый workflow для автоматического деплоя проектов через GitHub Actions.

Поддерживает опциональный шаг сборки (`BUILD`) и три метода доставки файлов на сервер:

| Метод | Описание |
|-------|----------|
| `FTP` | Зеркальный перенос через `lftp` |
| `RSYNC` | Синхронизация по SSH через `rsync` |
| `GIT` | `git clone` / `git pull` на сервере |

После деплоя (для RSYNC и GIT) опционально выполняется произвольная команда на сервере — `DEPLOY_COMMAND` (например, `php artisan migrate` или `pm2 restart app`).

---

## Быстрая настройка

### 1. Инициализация репозитория

```bash
git init
git checkout -b main
```

### 2. Добавить файлы в репозиторий

```
.github/workflows/ci-cd.yml   # workflow
.gitignore                     # список игнорируемых файлов
```

### 3. Закоммитить `.gitignore`

```bash
git add .gitignore
git commit -m "Add .gitignore"
```

### 4. Пересобрать индекс (если есть файлы, которые должны игнорироваться)

> Убедись, что нет незакоммиченных изменений перед выполнением.

```bash
git rm -r --cached .
git add .
git commit -m "Rebuild index according to updated .gitignore"
```

### 5. Связать с GitHub и запушить

```bash
git remote add origin git@github.com:OWNER/REPO.git
git push -u origin main
```

---

## Настройка Variables и Secrets

Открой: **Settings → Environments** → выбери или создай окружение (название совпадает с именем ветки).

### Variables

| Переменная | Описание | По умолчанию |
|------------|----------|--------------|
| `DEPLOY_METHOD` | Метод деплоя: `FTP`, `RSYNC`, `GIT` | — |
| `DEPLOY_HOST` | IP или домен сервера | — |
| `DEPLOY_USER` | Пользователь для подключения (SSH / FTP) | — |
| `DEPLOY_PATH` | Целевой путь на сервере | — |
| `DEPLOY_LOCAL_DIR` | Локальная папка в репозитории для деплоя | `./` |
| `DEPLOY_MIRROR` | `true` — удалять на сервере файлы, которых нет локально | `false` |
| `DEPLOY_PORT` | Порт SSH / FTP | `22` |
| `BUILD_COMMAND` | Команда сборки (если пусто — шаг пропускается) | — |
| `DEPLOY_COMMAND` | Команда на сервере после деплоя (если пусто — шаг пропускается). Не работает с FTP. | — |

### Secrets

| Секрет | Описание |
|--------|----------|
| `DEPLOY_KEY` | Приватный SSH-ключ (для RSYNC и GIT) или пароль FTP (для FTP) |

---

## Методы деплоя

### FTP

Использует `lftp mirror -R` для отправки файлов.

- `DEPLOY_PATH` указывается **относительно корня FTP-пользователя**.  
  Если при подключении сессия открывается в `/home/ftpuser/www/` — используй `./` или `subfolder/`.
- `DEPLOY_MIRROR=true` включает удаление файлов на сервере, которых нет локально — **используй осторожно**, сначала тестируй без него.
- `DEPLOY_COMMAND` для FTP **не поддерживается**: `DEPLOY_KEY` при FTP — это пароль, а не SSH-ключ.

### RSYNC

Синхронизирует файлы по SSH через `rsync`.

- Создаёт временный файл с приватным ключом, используется `-e "ssh -i TMP_KEY -o StrictHostKeyChecking=no -p PORT"`.
- `DEPLOY_MIRROR=true` добавляет флаг `--delete` — удаляет на сервере файлы, которых нет локально.
- После синхронизации опционально выполняется `DEPLOY_COMMAND` на сервере (в директории `DEPLOY_PATH`).

### GIT

Подключается по SSH к серверу и выполняет `git clone` / `git pull`.

- При первом деплое: `git clone --branch BRANCH git@github.com:OWNER/REPO.git .`
- `DEPLOY_MIRROR=true` → `git fetch origin BRANCH && git reset --hard origin/BRANCH`
- `DEPLOY_MIRROR=false` → `git pull origin BRANCH`
- Сервер должен иметь SSH-доступ к GitHub (Deploy key или ключ пользователя).
- После операции опционально выполняется `DEPLOY_COMMAND` на сервере (в директории `DEPLOY_PATH`).

---

## Генерация SSH-ключей

### Для RSYNC / FTP (ключ на runner → сервер)

```bash
ssh-keygen -t ed25519 -C "deploy@project" -f deploy_key
# deploy_key     → приватный ключ → сохранить как DEPLOY_KEY в Secrets
# deploy_key.pub → публичный ключ → добавить в ~/.ssh/authorized_keys на сервере
```

### Для GIT (ключ сервер → GitHub)

Генерация прямо на сервере от имени пользователя деплоя:

```bash
su - deploy
ssh-keygen -t ed25519 -C "git-deploy"
# ~/.ssh/id_ed25519.pub → добавить в GitHub → Settings → Deploy keys
```

---

## Настройка пользователя деплоя на сервере (для GIT)

```bash
# 1. Создать пользователя
useradd -m -s /bin/bash deploy

# 2. Добавить в группу web-сервера (если нужен доступ к веб-папке)
usermod -aG www-data deploy

# 3. Дать права на директорию проекта
chown -R deploy:www-data /var/www/project
chmod -R 775 /var/www/project

# 4. Сгенерировать SSH-ключ от имени пользователя deploy
su - deploy
ssh-keygen -t ed25519 -C "git-deploy"
# ~/.ssh/id_ed25519.pub добавить в GitHub → репозиторий → Settings → Deploy keys

# 5. Проверить доступ
ssh -T git@github.com
# Ожидаемый ответ: "Hi USERNAME! You've successfully authenticated..."
```

---

## Конфигурация SSH-ключей (deploy key на сервере)

Используется, когда серверу нужен доступ к приватному репозиторию:

```bash
# 1. Сгенерировать пару ключей
ssh-keygen -t ed25519 -C "REPONAME"

# 2. Запустить ssh-agent
eval "$(ssh-agent -s)"

# 3. Добавить приватный ключ в агент
ssh-add ~/.ssh/REPONAME

# 4. Проверить подключение
ssh -T git@github.com
```

Добавить в `~/.ssh/config`:

```
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/REPONAME
  IdentitiesOnly yes
```

### Несколько репозиториев с разными ключами

Если нужно работать с несколькими репозиториями под разными Deploy keys — создавай алиасы хоста:

```bash
# 1. Сгенерировать ключ для каждого репозитория
ssh-keygen -t ed25519 -C "REPO_A" -f ~/.ssh/REPO_A
ssh-keygen -t ed25519 -C "REPO_B" -f ~/.ssh/REPO_B

# 2. Добавить ключи в ssh-agent
ssh-add ~/.ssh/REPO_A
ssh-add ~/.ssh/REPO_B
```

Добавить в `~/.ssh/config`:

```
Host REPO_A
  HostName github.com
  User git
  IdentityFile ~/.ssh/REPO_A
  IdentitiesOnly yes

Host REPO_B
  HostName github.com
  User git
  IdentityFile ~/.ssh/REPO_B
  IdentitiesOnly yes
```

Указать remote с алиасом хоста в каждом репозитории:

```bash
git remote set-url origin git@REPO_A:OWNER/REPO_A.git

# Проверка
ssh -T git@REPO_A
```

---

## Команды для быстрых операций

```bash
# Инициализация и пуш в новый репозиторий
git init
git checkout -b main
git add .
git commit -m "Initial commit — add CI/CD workflow"
git remote add origin git@github.com:OWNER/REPO.git
git push -u origin main

# Добавление DEPLOY_KEY через gh CLI
gh secret set DEPLOY_KEY --body-file ./deploy_key

# Обновление .gitignore и пересборка индекса
git add .gitignore
git commit -m "Update .gitignore"
git rm -r --cached .
git add .
git commit -m "Rebuild index according to updated .gitignore"
```

---

## Тестирование и отладка

- Используй тестовую ветку (например, `ci-test`) для безопасной проверки.
- Логи доступны: **Actions** → выбери запуск → смотри `stdout/stderr` каждого шага.
- `BUILD_COMMAND` тестируй локально (`npm ci && npm run build`) до пуша.
- `DEPLOY_COMMAND` тестируй вручную на сервере через SSH перед добавлением в переменную.
- Для FTP: подключись клиентом и проверь корневую директорию при коннекте — это поможет правильно указать `DEPLOY_PATH`.

---

## Troubleshooting

| Ошибка | Причина и решение |
|--------|-------------------|
| Отказ доступа по SSH (RSYNC/GIT) | Проверь ключ: `chmod 600 deploy_key`, публичный ключ добавлен в `~/.ssh/authorized_keys` |
| `lftp` не подключается | Проверь хост, порт, логин и пароль; определи рабочую директорию FTP для корректного `DEPLOY_PATH` |
| `git clone` не проходит на сервере | Публичный ключ сервера не добавлен в GitHub Deploy keys |
| Ошибки сборки (BUILD) | Запусти `npm ci && BUILD_COMMAND` локально — убедись, что зависимости и скрипты работают без UI |
| `DEPLOY_COMMAND` не выполняется | Убедись, что метод не `FTP`, переменная не пустая, у `DEPLOY_USER` есть SSH-доступ и права на `DEPLOY_PATH` |
| `DEPLOY_COMMAND` завершается с ошибкой | Проверь, что команда корректно работает вручную: `ssh USER@HOST "cd DEPLOY_PATH && DEPLOY_COMMAND"` |
