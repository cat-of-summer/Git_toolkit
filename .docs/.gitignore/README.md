<!-- DOCGEN:START -->
# .gitignore

## Папки

- [Bitrix](Bitrix/)
- [Wordpress](Wordpress/)

<!-- DOCGEN:END -->

Здесь собраны шаблонные решения .gitignore, чтобы вновь применить правила в уже созданном репозитории:

# 1 Добавляем .gitignore и коммитим его
git add .gitignore
git commit -m "Update .gitignore."

# 2 Прекратить отслеживать файлы в индексе (оставит файлы на диске)
git rm -r --cached .

# 3 Добавить всё по-новой (индекс пересобран с учётом .gitignore)
git add .
git commit -m "Rebuild index according to updated .gitignore"
