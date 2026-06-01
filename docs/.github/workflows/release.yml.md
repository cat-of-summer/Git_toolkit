<!-- DOCGEN:START -->
# release.yml — создание GitHub Release

Workflow `.github/workflows/release.yml` автоматически создаёт GitHub Release при пуше тега.

Назначение:
- Определить ветку, из которой был создан тег
- Загрузить настройки деплой-окружения для этой ветки
- Опционально собрать проект
- Создать GitHub Release с автоматически сгенерированным описанием и прикреплёнными файлами

Триггеры:
- `push` — теги, соответствующие паттерну `v*` (например, `v1.0.0`, `v2.3.1-rc1`)

Права (permissions):
- `contents: write` — необходимо для создания Release

---

## Джобы

### get-branch

Определяет ветку, содержащую коммит тега.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `outputs` | `branch` — имя ветки |

Шаги:
1. Checkout с полной историей (`fetch-depth: 0`)
2. Поиск ветки через `git branch -r --contains <sha>` — берётся первая удалённая ветка (не HEAD), из неё отрезается префикс `origin/`

Результат передаётся в джобу `release` через `needs.get-branch.outputs.branch`.

### release

Выполняет сборку и создаёт GitHub Release.

| Поле | Значение |
|------|----------|
| `runs-on` | `ubuntu-latest` |
| `needs` | `get-branch` |
| `environment` | `${{ needs.get-branch.outputs.branch }}` — имя ветки используется как Environment |

Переменные окружения (через GitHub `Environments` / `vars`):

| Переменная | Описание |
|------------|----------|
| `BUILD_COMMAND` | Команда сборки. Если пустая — шаг Build пропускается |
| `RELEASE_FILES` | Паттерн файлов для прикрепления к Release (например, `dist/*.zip`) |

Шаги:
1. Checkout репозитория
2. **Build** — выполняется только если `BUILD_COMMAND` не пустой; запускается через `eval` в bash с `set -euo pipefail`
3. **Create GitHub Release** — используется `softprops/action-gh-release@v2`:
   - Тег: `github.ref_name`
   - Название: `Release <tag>`
   - Описание: автоматически генерируется из коммитов (`generate_notes: true`)
   - Прикреплённые файлы: по паттерну из `RELEASE_FILES`

---

## Как работает определение окружения

Тег может быть создан из любой ветки. Workflow определяет исходную ветку и использует её имя как GitHub Environment. Это позволяет иметь разные настройки сборки и разные файлы релиза для разных веток (например, отдельные конфиги для `main` и `develop`).

---

## Настройка

1. Создай GitHub Environment с именем ветки (например, `main`) в Settings → Environments
2. Задай переменные в этом Environment:
   - `BUILD_COMMAND` — команда сборки (необязательно)
   - `RELEASE_FILES` — паттерн файлов для релиза (необязательно)
3. Создай и запушь тег:
   ```
   git tag v1.0.0
   git push origin v1.0.0
   ```

Примеры значений переменных:

```
BUILD_COMMAND=npm ci && npm run build
RELEASE_FILES=dist/*.zip
```

```
BUILD_COMMAND=make release
RELEASE_FILES=bin/app-linux bin/app-windows.exe
```

---

Секреты:

| Секрет | Описание |
|--------|----------|
| `GITHUB_TOKEN` | Встроенный токен GitHub Actions, используется для создания Release |

Отладка:
- Если Release не создаётся — проверь, что тег соответствует паттерну `v*`
- Если шаг Build падает — проверь `BUILD_COMMAND` в Environment
- Если файлы не прикрепляются — проверь паттерн `RELEASE_FILES` и результаты сборки
- Логи: Actions → выбрать run → шаг `Create GitHub Release`

<!-- DOCGEN:END -->
