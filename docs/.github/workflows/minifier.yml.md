# minifier.yml

Workflow автоминификации CSS и JS файлов. Запускается при `push` в любую ветку.
Находит все файлы `*.css` и `*.js` (не `*.min.*`), создаёт рядом `filename.min.css` / `filename.min.js` и коммитит их в ту же ветку.

## Шаги

| Шаг | Описание |
|------|------------|
| `Checkout` | `fetch-depth: 0`, `persist-credentials: true` |
| `Configure git user` | Устанавливает `github-actions[bot]` для коммитов |
| `Setup Node.js 18` | `actions/setup-node@v3` |
| `Install latest minifiers` | `npm install csso-cli terser@latest` (no-save, no-audit) |
| `Find and minify .css & .js` | Рекурсивный поиск + параллельная минификация через `xargs -P` |
| `Stage only minified files` | `git add *.min.css *.min.js` |
| `Commit & push` | Коммит `auto:minify [skip ci]` если есть изменения |

## Инструменты

| Инструмент | Назначение |
|-----------|------------|
| `csso-cli` | Минификация CSS |
| `terser` | Минификация JS (`--compress --mangle`) |

## Исключения при поиске

Workflow не минифицирует файлы в следующих папках:
- Скрытые (начинающиеся с `.`)
- `node_modules/`, `vendor/`, `dist/`, `build/`, `bower_components/`
- Файлы уже названные `*.min.*`

## Параллельная обработка

Файлы обрабатываются параллельно: `xargs -P $(nproc)`. На стандартных GitHub-раннерах доступно 2 ядра.

## Выходные файлы

```
src/app.css    →  src/app.min.css
src/main.js    →  src/main.min.js
```

Минифицированные файлы подключай напрямую в HTML: `<link rel="stylesheet" href="app.min.css">`.
