<!-- DOCGEN:START -->
# .gitignore

## Папки

- [Bitrix](Bitrix/)
- [Wordpress](Wordpress/)

<!-- DOCGEN:END -->

Коллекция готовых `.gitignore` шаблонов для популярных CMS и фреймворков.
Каждый шаблон заточен под конкретную платформу: не трекает core/runtime, трекает только исходники проекта.

## Как использовать

Скопируй нужный `.gitignore` в корень проекта:

```bash
# Для Bitrix-проекта
cp .gitignore/Bitrix/.gitignore /path/to/project/.gitignore

# Для WordPress-проекта
cp .gitignore/Wordpress/.gitignore /path/to/project/.gitignore
```

Затем пересобери индекс, если репозиторий уже инициализирован:

```bash
git rm -r --cached .
git add .
git commit -m "Apply .gitignore"
```

## Шаблоны

| Шаблон | Исключает | Трекает |
|--------|-----------|--------|
| [Bitrix](Bitrix/) | `/bitrix/` core, кеши, загрузки, логи | `/local/`, шаблоны, конфигурацию |
| [Wordpress](Wordpress/) | `/wp-admin/`, `/wp-includes/`, плагины, кеши | `/wp-content/themes/`, конфигурацию |

Здесь собраны шаблонные решения .gitignore, чтобы вновь применить правила в уже созданном репозитории:

# 1 Добавляем .gitignore и коммитим его
```bash
git add .gitignore
git commit -m "Update .gitignore."
```

# 2 Прекратить отслеживать файлы в индексе (оставит файлы на диске)
```bash
git rm -r --cached .
```

# 3 Добавить всё по-новой (индекс пересобран с учётом .gitignore)
```bash
git add .
git commit -m "Rebuild index according to updated .gitignore"
```
