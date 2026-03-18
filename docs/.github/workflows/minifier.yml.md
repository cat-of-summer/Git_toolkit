# minifier.yml — автоминификация CSS & JS

Workflow: `.github/workflows/minifier.yml`

Назначение: автоматически минифицировать фронтенд-ассеты (`*.css`, `*.js`) и закоммитить результаты в ту же ветку.

Триггер: `push`

Что делает шаг `Find and minify`:
- Устанавливает `csso-cli` и `terser` локально (npm)
- Находит все `*.css` и `*.js`, исключая `*.min.*` и каталоги: `node_modules`, `vendor`, `dist`, `build`, `bower_components`.
- Для каждого файла создаётся `file.min.css` / `file.min.js` рядом с исходником.
- Параллелит через `xargs -P $(nproc)` для ускорения на раннерах.

Настройки и рекомендации:
- Версия Node.js: 18 (устанавливается через `actions/setup-node@v3`).
- Не храните в репо исходники, которые не должны минифицироваться (используйте `.docignore`/`.gitignore`).
- Если вы предпочитаете собирать ассеты в `dist/` локально, настройте `DEPLOY_LOCAL_DIR` чтобы деплой отправлял нужную папку.

Коммит и push:
- После генерации workflow stages только minified файлы в индекс и делает коммит `auto:minify [skip ci]`.
- Если нет изменений — workflow завершится с сообщением `No minified changes to commit`.

Примеры преобразования:

```
src/styles/app.css  → src/styles/app.min.css
assets/main.js      → assets/main.min.js
```

Отладка:
- Посмотрите вывод шага `Find and minify` в Actions для списка файлов и ошибок минификации.
- Локально можно проверить команды:

```bash
npx csso src/styles/app.css --output src/styles/app.min.css
npx terser assets/main.js -o assets/main.min.js --compress --mangle
```

