# Patterns Book

Книга о паттернах проектирования, созданная с использованием [mdBook](https://rust-lang.github.io/mdBook/).

## Требования

- [mdBook](https://rust-lang.github.io/mdBook/guide/installation.html) должен быть установлен в системе

## Установка mdBook

### Windows
```bash
cargo install mdbook
```

### Linux/MacOS
```bash
cargo install mdbook
```

Или скачайте готовые бинарники с [GitHub Releases](https://github.com/rust-lang/mdBook/releases).

## Работа с книгой

### Генерация книги
```bash
mdbook build
```
HTML-файлы будут созданы в директории `book/`.

### Локальный сервер для разработки
```bash
mdbook serve
```
Книга будет доступна по адресу http://localhost:3000 с автоматической перезагрузкой при изменении файлов.

### Очистка сгенерированных файлов
```bash
mdbook clean
```

## Структура проекта

```
patterns-book/
├── book.toml          # Конфигурация книги
├── src/               # Исходные файлы книги
│   ├── SUMMARY.md     # Оглавление
│   └── ...            # Главы книги
└── book/              # Сгенерированные HTML-файлы
```

## Редактирование содержимого

1. Добавьте новые главы в директорию `src/`
2. Обновите оглавление в `src/SUMMARY.md`
3. Запустите `mdbook serve` для предварительного просмотра изменений
4. Используйте `mdbook build` для финальной сборки

## Дополнительная информация

- [Документация mdBook](https://rust-lang.github.io/mdBook/)
- [Руководство по форматированию](https://rust-lang.github.io/mdBook/format/index.html)
