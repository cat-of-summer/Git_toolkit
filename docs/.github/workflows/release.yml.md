<!-- DOCGEN:START -->
# release.yml — создание GitHub Release

Workflow `.github/workflows/release.yml` автоматически создаёт GitHub Release при создании тега и при необходимости публикует npm-пакет.

Назначение:
- Определить ветку, из которой был создан тег
- Загрузить настройки деплой-окружения для этой ветки
- Опционально собрать проект
- Подготовить и переименовать файлы релиза (flatten)
- Создать GitHub Release с автоматически сгенерированным описанием и прикреплёнными файлами
- Опционально опубликовать npm-пакет в GitHub Packages

Триггеры:
- `create` — срабатывает при создании любого тега или ветки; джоба `get-branch` дополнительно проверяет `github.ref_type == 'tag'`

Права (permissions):
- `contents: write` — необходимо для создания Release
- `packages: write` — необходимо для публикации npm-пакета в GitHub Packages

---

## Джобы

### get-branch

Определяет ветку, содержащую коммит тега.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `if` | `github.ref_type == 'tag'` — выполняется только для тегов |
| `outputs` | `branch` — имя ветки |

Шаги:
1. Checkout с полной историей (`fetch-depth: 0`)
2. Поиск ветки через `git branch -r --contains <sha>` — берётся первая удалённая ветка (не HEAD), из неё отрезается префикс `origin/`

Результат передаётся в джобы `release` и `npm-publish` через `needs.get-branch.outputs.branch`.

### release

Выполняет сборку, подготовку файлов и создаёт GitHub Release.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `get-branch` |
| `environment` | `${{ needs.get-branch.outputs.branch }}` — имя ветки используется как Environment |
| `outputs` | `publish_to_npm` — флаг для джобы `npm-publish` |

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки. Если пустая — шаг Build пропускается |
| `RELEASE_FILES` | Glob-паттерны файлов для прикрепления к Release, через запятую (например, `dist/*.zip`) |
| `PUBLISH_TO_NPM` | Если `'true'` — после релиза запускается джоба `npm-publish` |

Шаги:
1. Checkout репозитория
2. **Build** — выполняется только если `BUILD_COMMAND` не пустой; запускается через `eval` в bash с `set -euo pipefail`
3. **Flatten release asset names** — выполняется только если `RELEASE_FILES` не пустой:
   - Создаёт временную папку `.release-staging/`
   - Разбивает `RELEASE_FILES` по запятой на отдельные паттерны
   - Для каждого совпадающего файла: вычисляет относительный путь и заменяет `/` на `__` в имени файла (flatten)
   - Копирует файлы в `.release-staging/` с плоскими именами
   - Если ни один файл не совпал — завершается с ошибкой
   - Список staged-файлов передаётся в output `files` (multiline, через EOF-блок)
4. **Create GitHub Release** — используется `softprops/action-gh-release@v2`:
   - Тег: `github.ref_name`
   - Название: `Release <tag>`
   - Описание: автоматически генерируется из коммитов (`generate_release_notes: true`)
   - Прикреплённые файлы: staged-файлы из шага `stage` (`steps.stage.outputs.files`)

### npm-publish

Публикует npm-пакет в GitHub Packages.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `get-branch`, `release` |
| `if` | `needs.release.outputs.publish_to_npm == 'true'` |
| `environment` | `${{ needs.get-branch.outputs.branch }}` |

Переменные окружения:

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки (та же, что и в джобе `release`) |

Шаги:
1. Checkout репозитория
2. Setup Node.js с реестром `https://npm.pkg.github.com`
3. **Build** — аналогично джобе `release`, пропускается если `BUILD_COMMAND` пустой
4. **Override npm auth for publish** — патчит `.npmrc`, подставляя `GITHUB_TOKEN` в качестве токена аутентификации
5. **Set version from tag** — проверяет, что тег соответствует формату `v#`, `v#.#` или `v#.#.#`; извлекает числа и устанавливает версию пакета через `npm version` (без создания git-тега)
6. `npm publish` — публикует пакет

---

## Как работает определение окружения

Тег может быть создан из любой ветки. Workflow определяет исходную ветку и использует её имя как GitHub Environment. Это позволяет иметь разные настройки сборки и разные файлы релиза для разных веток (например, отдельные конфиги для `main` и `develop`).

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
   - `PUBLISH_TO_NPM` — `true` для публикации npm-пакета (необязательно)
3. Создай и запушь тег:
   ```
   git tag v1.0.0
   git push origin v1.0.0
   ```

Примеры значений переменных:

```
BUILD_COMMAND=npm ci && npm run build
RELEASE_FILES=dist/*.zip
PUBLISH_TO_NPM=true
```

```
BUILD_COMMAND=make release
RELEASE_FILES=bin/app-linux,bin/app-windows.exe
```

---

Секреты:

| Секрет | Описание |
|--------|----------|
| `GITHUB_TOKEN` | Встроенный токен GitHub Actions, используется для создания Release и публикации npm-пакета |

Отладка:
- Если Release не создаётся — проверь, что создан именно **тег** (не ветка); джоба `get-branch` проверяет `ref_type == 'tag'`
- Если шаг Build падает — проверь `BUILD_COMMAND` в Environment
- Если файлы не прикрепляются — проверь паттерн `RELEASE_FILES` и результаты сборки; убедись, что хотя бы один файл совпадает
- Если npm-публикация не запускается — проверь, что `PUBLISH_TO_NPM=true` задана в Environment и тег соответствует формату `v#.#.#`
- Логи: Actions → выбрать run → шаг `Create GitHub Release` или `npm publish`

<!-- DOCGEN:END -->
