# Паттерн Abstract Factory (Абстрактная фабрика)

## Обзор

**Abstract Factory** (Абстрактная фабрика) — это порождающий паттерн проектирования из «Банды четырёх» (GoF), который предоставляет интерфейс для создания семейств взаимосвязанных или взаимозависимых объектов без указания их конкретных классов.

### Основная задача паттерна

Создание **семейства объектов** с одинаковым контрактом, где таких семейств может быть много. Паттерн позволяет:

- Создавать несколько классов для доступа к одной базе данных и такое же количество классов с аналогичным контрактом для доступа к другой базе данных
- Работать с разными провайдерами сервисов (например, AWS, Azure, Google Cloud)
- Создавать компоненты пользовательского интерфейса для разных платформ
- Абстрагироваться от конкретных реализаций backend-компонентов

---

## Применение паттерна

Abstract Factory применяется в самых разных областях:

- **Frontend**: Рендеринг UI-компонентов для разных платформ (Web, Mobile, Desktop)
- **Backend**: Работа с различными базами данных, файловыми системами, облачными провайдерами
- **Кросс-платформенные решения**: Создание унифицированного API для разных операционных систем

---

## Соблюдение принципов SOLID

Паттерн Abstract Factory помогает соблюдать важные принципы проектирования:

1. **Single Responsibility Principle (SRP)** — каждая фабрика отвечает только за создание своего семейства объектов
2. **Low Coupling** — слабое зацепление между клиентским кодом и конкретными реализациями
3. **Dependency Inversion Principle** — зависимость от абстракций, а не от конкретных классов

---

## Структура паттерна

### Классическая реализация (для языков с интерфейсами)

```
┌─────────────────────────┐
│  AbstractFactory        │ ◄──── Абстрактная фабрика
├─────────────────────────┤
│ + createProductA()      │
│ + createProductB()      │
└─────────────────────────┘
           △
           │
           │ Наследование
           │
┌──────────┴──────────┐
│                     │
┌─────────────────────┴─────┐    ┌─────────────────────────────┐
│  ConcreteFactory1         │    │  ConcreteFactory2           │
├───────────────────────────┤    ├─────────────────────────────┤
│ + createProductA()        │    │ + createProductA()          │
│ + createProductB()        │    │ + createProductB()          │
└───────────────────────────┘    └─────────────────────────────┘
         │                                  │
         │ Создаёт                          │ Создаёт
         ▼                                  ▼
┌─────────────────────┐            ┌─────────────────────┐
│ ConcreteProductA1   │            │ ConcreteProductA2   │
└─────────────────────┘            └─────────────────────┘
         △                                  △
         │                                  │
         │ Реализует                        │ Реализует
         │                                  │
┌─────────────────────┐            ┌─────────────────────┐
│  AbstractProductA   │            │  AbstractProductA   │
└─────────────────────┘            └─────────────────────┘
```

### Две иерархии классов

Паттерн использует **две параллельные иерархии**:

1. **Абстрактная иерархия**:
   - `AbstractFactory` — определяет методы создания
   - `AbstractProductA`, `AbstractProductB` — определяют контракты продуктов

2. **Конкретная иерархия**:
   - `ConcreteFactory1`, `ConcreteFactory2` — реализации фабрик
   - `ConcreteProductA1`, `ConcreteProductB1` — конкретные продукты для семейства 1
   - `ConcreteProductA2`, `ConcreteProductB2` — конкретные продукты для семейства 2

---

## Реализация для JavaScript/TypeScript

### Проблема с абстрактными классами в JavaScript

В JavaScript **нет встроенной поддержки** интерфейсов и абстрактных классов. Поэтому для имитации абстрактности используются следующие подходы:

#### Имитация абстрактного класса

```javascript
class AbstractFactory {
  constructor() {
    // Бросаем исключение при попытке создания экземпляра абстрактного класса
    if (new.target === AbstractFactory) {
      throw new Error('Cannot instantiate abstract class AbstractFactory');
    }
  }

  createProductA() {
    // Бросаем исключение для абстрактного метода
    throw new Error('Method createProductA() must be implemented');
  }

  createProductB() {
    throw new Error('Method createProductB() must be implemented');
  }
}

class AbstractProductA {
  constructor() {
    if (new.target === AbstractProductA) {
      throw new Error('Cannot instantiate abstract class');
    }
  }

  operation() {
    throw new Error('Method operation() must be implemented');
  }
}
```

### Упрощённая реализация для JavaScript

Для JavaScript можно значительно упростить реализацию, **избавившись от абстрактных классов**:

```javascript
// Конкретные классы продуктов (без абстрактных базовых классов)
class FileStorageDatabase {
  constructor(filename) {
    this.filename = filename;
  }

  select(filters) {
    // Создаём курсор для итерации по файлу
    return new FileStorageCursor(this.filename, filters);
  }
}

class FileStorageCursor {
  constructor(filename, filters) {
    this.filename = filename;
    this.filters = filters;
  }

  // Асинхронный итератор для построчного чтения
  async *[Symbol.asyncIterator]() {
    const fs = require('fs');
    const readline = require('readline');

    const fileStream = fs.createReadStream(this.filename);
    const rl = readline.createInterface({
      input: fileStream,
      crlfDelay: Infinity
    });

    for await (const line of rl) {
      const data = JSON.parse(line);

      // Фильтрация данных
      if (this.matchesFilters(data)) {
        yield data;
      }
    }
  }

  matchesFilters(data) {
    // Проверка соответствия фильтрам
    for (const [field, value] of Object.entries(this.filters)) {
      if (data[field] !== value) {
        return false;
      }
    }
    return true;
  }
}

// Фабрика как простой объект с методами
const fsDataAccessLayer = {
  createDatabase(filename) {
    return new FileStorageDatabase(filename);
  },

  createCursor(filename, filters) {
    return new FileStorageCursor(filename, filters);
  }
};

// Экспортируем фабрику
module.exports = fsDataAccessLayer;
```

---

## Практический пример: Работа с базами данных

### Полная реализация с файловым хранилищем

Рассмотрим реальный пример создания абстракции для работы с данными, где одна реализация использует файловую систему, а другая может использовать PostgreSQL или MySQL.

#### Шаг 1: Абстрактная фабрика (для языков с интерфейсами)

```javascript
class AbstractDataAccessLayer {
  constructor() {
    if (new.target === AbstractDataAccessLayer) {
      throw new Error('Cannot instantiate abstract factory');
    }
  }

  createDatabase() {
    throw new Error('Method createDatabase() must be implemented');
  }

  createCursor() {
    throw new Error('Method createCursor() must be implemented');
  }
}
```

#### Шаг 2: Абстрактные продукты

```javascript
// Абстрактная база данных
class AbstractDatabase {
  constructor() {
    if (new.target === AbstractDatabase) {
      throw new Error('Cannot instantiate abstract class');
    }
  }

  select(filters) {
    throw new Error('Method select() must be implemented');
  }
}

// Абстрактный курсор
class AbstractCursor {
  constructor() {
    if (new.target === AbstractCursor) {
      throw new Error('Cannot instantiate abstract class');
    }
  }

  async *[Symbol.asyncIterator]() {
    throw new Error('Async iterator must be implemented');
  }
}
```

#### Шаг 3: Конкретная реализация — FileStorage

```javascript
const fs = require('fs');
const readline = require('readline');

// Конкретная база данных для файлового хранилища
class FileStorageDatabase extends AbstractDatabase {
  constructor(filename) {
    super();
    this.filename = filename;
  }

  select(filters) {
    // Создаём курсор с фильтрами
    return new FileStorageCursor(this.filename, filters);
  }
}

// Конкретный курсор для файлового хранилища
class FileStorageCursor extends AbstractCursor {
  constructor(filename, filters) {
    super();
    this.filename = filename;
    this.filters = filters;
  }

  // Асинхронный итератор для построчного чтения и парсинга
  async *[Symbol.asyncIterator]() {
    const fileStream = fs.createReadStream(this.filename);
    const rl = readline.createInterface({
      input: fileStream,
      crlfDelay: Infinity
    });

    for await (const line of rl) {
      try {
        const record = JSON.parse(line);

        // Проверяем соответствие фильтрам
        if (this.matchesFilters(record)) {
          yield record;
        }
      } catch (error) {
        // Пропускаем невалидные строки
        console.error('Invalid JSON line:', line);
      }
    }
  }

  matchesFilters(record) {
    // Если фильтров нет — возвращаем все записи
    if (!this.filters || Object.keys(this.filters).length === 0) {
      return true;
    }

    // Проверяем каждое поле в фильтре
    for (const [field, value] of Object.entries(this.filters)) {
      if (record[field] !== value) {
        return false;
      }
    }

    return true;
  }
}
```

#### Шаг 4: Конкретная фабрика

```javascript
// Конкретная фабрика для файлового хранилища
class FSDataAccessLayer extends AbstractDataAccessLayer {
  createDatabase(filename) {
    return new FileStorageDatabase(filename);
  }

  createCursor(filename, filters) {
    return new FileStorageCursor(filename, filters);
  }
}

module.exports = FSDataAccessLayer;
```

#### Шаг 5: Использование

```javascript
// Создаём экземпляр фабрики
const fsDAL = new FSDataAccessLayer();

// Создаём базу данных
const db = fsDAL.createDatabase('./data/users.jsonl');

// Выполняем select с фильтром
const cursor = db.select({ city: 'Moscow' });

// Итерируемся по результатам
(async () => {
  for await (const user of cursor) {
    console.log(`User: ${user.name}, City: ${user.city}`);
  }
})();
```

**Преимущества такого подхода:**

- Очень большой файл читается **построчно** (не загружается целиком в память)
- Используется **асинхронный итератор** для эффективной работы с I/O
- Данные **фильтруются на лету** во время чтения
- Код легко расширить для работы с другими источниками данных (PostgreSQL, MongoDB, S3)

---

## Упрощённая реализация для JavaScript (без абстракций)

Для JavaScript можно полностью отказаться от абстрактных классов и использовать **простые объекты и функции**:

### Вариант 1: Объект с методами

```javascript
// fs-dal.js — Модуль для файловой системы
const fs = require('fs');
const readline = require('readline');

class FileStorageDatabase {
  constructor(filename) {
    this.filename = filename;
  }

  select(filters) {
    return new FileStorageCursor(this.filename, filters);
  }
}

class FileStorageCursor {
  constructor(filename, filters) {
    this.filename = filename;
    this.filters = filters;
  }

  async *[Symbol.asyncIterator]() {
    const fileStream = fs.createReadStream(this.filename);
    const rl = readline.createInterface({
      input: fileStream,
      crlfDelay: Infinity
    });

    for await (const line of rl) {
      const record = JSON.parse(line);
      if (this.matchesFilters(record)) {
        yield record;
      }
    }
  }

  matchesFilters(record) {
    if (!this.filters) return true;
    return Object.entries(this.filters).every(
      ([key, value]) => record[key] === value
    );
  }
}

// Экспортируем фабрику как объект с методами
module.exports = {
  createDatabase(filename) {
    return new FileStorageDatabase(filename);
  },

  createCursor(filename, filters) {
    return new FileStorageCursor(filename, filters);
  }
};
```

### Коллекция фабрик (паттерн Strategy)

Для переключения между различными реализациями используем **коллекцию фабрик**:

```javascript
// data-access-factories.js
const fsDAL = require('./fs-dal');
const pgDAL = require('./pg-dal');
const mysqlDAL = require('./mysql-dal');

// Коллекция фабрик для разных хранилищ
const factories = {
  fs: fsDAL,
  postgres: pgDAL,
  mysql: mysqlDAL
};

// Функция для выбора нужной фабрики
function getFactory(storageType) {
  const factory = factories[storageType];

  if (!factory) {
    throw new Error(`Unknown storage type: ${storageType}`);
  }

  return factory;
}

module.exports = { factories, getFactory };
```

### Использование с выбором стратегии

```javascript
const { getFactory } = require('./data-access-factories');

// Выбираем стратегию (можно из конфига или переменных окружения)
const storageType = process.env.STORAGE_TYPE || 'fs';
const dal = getFactory(storageType);

// Создаём базу данных
const db = dal.createDatabase('./data/users.jsonl');

// Выполняем запрос
const cursor = db.select({ city: 'Moscow' });

// Используем результаты
(async () => {
  for await (const user of cursor) {
    console.log(`User: ${user.name}, Email: ${user.email}`);
  }
})();
```

**Важно:** Здесь сочетаются два паттерна:
- **Abstract Factory** — для создания семейства объектов (Database + Cursor)
- **Strategy** — для выбора конкретной фабрики из коллекции

---

## Реализация с TypeScript

TypeScript позволяет использовать **интерфейсы** для строгой типизации:

```typescript
// Интерфейсы для продуктов
interface IDatabase {
  select(filters: Record<string, any>): ICursor;
}

interface ICursor {
  [Symbol.asyncIterator](): AsyncIterator<any>;
}

// Интерфейс для фабрики
interface IDataAccessLayer {
  createDatabase(config: any): IDatabase;
  createCursor(config: any, filters: Record<string, any>): ICursor;
}

// Реализация для файлового хранилища
class FileStorageDatabase implements IDatabase {
  constructor(private filename: string) {}

  select(filters: Record<string, any>): ICursor {
    return new FileStorageCursor(this.filename, filters);
  }
}

class FileStorageCursor implements ICursor {
  constructor(
    private filename: string,
    private filters: Record<string, any>
  ) {}

  async *[Symbol.asyncIterator](): AsyncIterator<any> {
    const fs = await import('fs');
    const readline = await import('readline');

    const fileStream = fs.createReadStream(this.filename);
    const rl = readline.createInterface({
      input: fileStream,
      crlfDelay: Infinity
    });

    for await (const line of rl) {
      const record = JSON.parse(line);
      if (this.matchesFilters(record)) {
        yield record;
      }
    }
  }

  private matchesFilters(record: any): boolean {
    if (!this.filters || Object.keys(this.filters).length === 0) {
      return true;
    }

    return Object.entries(this.filters).every(
      ([key, value]) => record[key] === value
    );
  }
}

// Конкретная фабрика
class FSDataAccessLayer implements IDataAccessLayer {
  createDatabase(filename: string): IDatabase {
    return new FileStorageDatabase(filename);
  }

  createCursor(filename: string, filters: Record<string, any>): ICursor {
    return new FileStorageCursor(filename, filters);
  }
}

// Использование
const dal: IDataAccessLayer = new FSDataAccessLayer();
const db: IDatabase = dal.createDatabase('./data/users.jsonl');
const cursor: ICursor = db.select({ city: 'Moscow' });

(async () => {
  for await (const user of cursor) {
    console.log(user);
  }
})();
```

**Преимущества TypeScript-реализации:**

- ✅ **Строгая типизация** всех методов и возвращаемых значений
- ✅ **Автодополнение** в IDE на основе интерфейсов
- ✅ **Проверка на этапе компиляции** — ошибки находятся до выполнения
- ✅ **Явные контракты** через интерфейсы

---

## Классический пример: UI-библиотека для разных платформ

### Задача

Создать библиотеку UI-компонентов, которая может рендерить элементы для:
- Web (HTML/CSS)
- Windows (WinAPI)
- macOS (Cocoa)

### Реализация

```typescript
// Интерфейсы продуктов
interface IButton {
  render(): void;
  onClick(callback: () => void): void;
}

interface ICheckbox {
  render(): void;
  toggle(): void;
  isChecked(): boolean;
}

// Интерфейс абстрактной фабрики
interface IUIFactory {
  createButton(label: string): IButton;
  createCheckbox(label: string): ICheckbox;
}

// Реализация для Web
class WebButton implements IButton {
  constructor(private label: string) {}

  render(): void {
    console.log(`<button>${this.label}</button>`);
  }

  onClick(callback: () => void): void {
    console.log(`Attached click handler to button: ${this.label}`);
  }
}

class WebCheckbox implements ICheckbox {
  private checked = false;

  constructor(private label: string) {}

  render(): void {
    console.log(`<input type="checkbox" /> ${this.label}`);
  }

  toggle(): void {
    this.checked = !this.checked;
  }

  isChecked(): boolean {
    return this.checked;
  }
}

// Фабрика для Web
class WebUIFactory implements IUIFactory {
  createButton(label: string): IButton {
    return new WebButton(label);
  }

  createCheckbox(label: string): ICheckbox {
    return new WebCheckbox(label);
  }
}

// Реализация для Windows
class WindowsButton implements IButton {
  constructor(private label: string) {}

  render(): void {
    console.log(`[Win32 Button: ${this.label}]`);
  }

  onClick(callback: () => void): void {
    console.log(`Attached WM_COMMAND handler to: ${this.label}`);
  }
}

class WindowsCheckbox implements ICheckbox {
  private checked = false;

  constructor(private label: string) {}

  render(): void {
    console.log(`[Win32 Checkbox: ${this.label}]`);
  }

  toggle(): void {
    this.checked = !this.checked;
  }

  isChecked(): boolean {
    return this.checked;
  }
}

// Фабрика для Windows
class WindowsUIFactory implements IUIFactory {
  createButton(label: string): IButton {
    return new WindowsButton(label);
  }

  createCheckbox(label: string): ICheckbox {
    return new WindowsCheckbox(label);
  }
}

// Клиентский код
class Application {
  private button: IButton;
  private checkbox: ICheckbox;

  constructor(factory: IUIFactory) {
    this.button = factory.createButton('Submit');
    this.checkbox = factory.createCheckbox('Remember me');
  }

  render(): void {
    this.button.render();
    this.checkbox.render();
  }
}

// Использование
function main() {
  const platform = process.platform;

  let factory: IUIFactory;

  if (platform === 'win32') {
    factory = new WindowsUIFactory();
  } else {
    factory = new WebUIFactory();
  }

  const app = new Application(factory);
  app.render();
}

main();
```

---

## Преимущества паттерна Abstract Factory

### ✅ 1. Изоляция конкретных классов

Клиентский код работает только с интерфейсами, не зная о конкретных реализациях:

```javascript
// Клиент знает только об интерфейсах
function processData(factory) {
  const db = factory.createDatabase(config);
  const cursor = db.select({ status: 'active' });

  // Не важно, используется PostgreSQL, MySQL или файловая система
  for await (const record of cursor) {
    console.log(record);
  }
}
```

### ✅ 2. Гарантия совместимости продуктов

Все продукты одной фабрики **гарантированно совместимы** между собой:

```typescript
// FileStorageDatabase всегда создаёт FileStorageCursor
// PostgresDatabase всегда создаёт PostgresCursor
// Нет риска смешать курсор от одной БД с соединением к другой
```

### ✅ 3. Лёгкость добавления новых семейств

Для добавления новой реализации (например, MongoDB) нужно:

1. Создать классы продуктов (`MongoDatabase`, `MongoCursor`)
2. Создать фабрику (`MongoDataAccessLayer`)
3. Добавить её в коллекцию

**Существующий код не меняется!**

### ✅ 4. Соответствие принципу Open/Closed

- **Open for extension** — легко добавить новую фабрику
- **Closed for modification** — не нужно менять существующий код

---

## Недостатки и ограничения

### ❌ 1. Сложность кода

Добавление каждого нового типа продукта требует изменений во **всех фабриках**:

```typescript
// Если нужно добавить createIndex(), придётся изменить:
interface IDataAccessLayer {
  createDatabase(config: any): IDatabase;
  createCursor(config: any, filters: any): ICursor;
  createIndex(config: any): IIndex; // ← Новый метод
}

// И все реализации:
class FSDataAccessLayer implements IDataAccessLayer {
  // Нужно реализовать createIndex()
}

class PostgresDataAccessLayer implements IDataAccessLayer {
  // Нужно реализовать createIndex()
}
```

### ❌ 2. Излишняя абстракция для простых случаев

Если нужно создать только **один класс**, Abstract Factory избыточен. В таких случаях достаточно **Factory Method** или простой функции.

---

## Отличие от Factory Method

| Характеристика | Factory Method | Abstract Factory |
|---------------|----------------|------------------|
| **Количество продуктов** | Один тип продукта | Семейство продуктов |
| **Структура** | Один метод `create()` | Несколько методов создания |
| **Применение** | Когда нужна вариативность одного объекта | Когда нужно создавать взаимосвязанные объекты |
| **Пример** | Создание логгера (FileLogger, ConsoleLogger) | Создание UI-компонентов (Button + Checkbox) |

### Пример Factory Method

```javascript
// Простая фабричная функция
function createLogger(type) {
  switch (type) {
    case 'file':
      return new FileLogger();
    case 'console':
      return new ConsoleLogger();
    default:
      throw new Error('Unknown logger type');
  }
}
```

### Пример Abstract Factory

```javascript
// Фабрика создаёт целое семейство
const uiFactory = new WebUIFactory();
const button = uiFactory.createButton('OK');
const checkbox = uiFactory.createCheckbox('Agree');
const textbox = uiFactory.createTextbox('Enter name');
```

---

## Когда использовать Abstract Factory

### ✅ Используйте, когда:

1. **Нужно создать семейство взаимосвязанных объектов**
   - UI-компоненты одной темы (Light/Dark)
   - Классы доступа к одной БД (Connection + Cursor + Transaction)

2. **Система должна быть независима от способа создания объектов**
   - Подмена реализации через конфигурацию
   - Тестирование с моками

3. **Требуется обеспечить совместимость продуктов**
   - Все объекты из одной фабрики гарантированно работают вместе

4. **Нужно предоставить библиотеку, скрывающую детали реализации**
   - Клиент работает только с интерфейсами

### ❌ Не используйте, когда:

1. Нужно создать **один объект** — используйте Factory Method
2. Объекты **не связаны** между собой — используйте отдельные фабрики
3. **Простая система** без вариативности реализаций

---

## Комбинация с другими паттернами

### 1. Abstract Factory + Strategy

```javascript
// Коллекция фабрик — это паттерн Strategy
const factories = {
  development: new MockDataAccessLayer(),
  production: new PostgresDataAccessLayer(),
  testing: new InMemoryDataAccessLayer()
};

const factory = factories[process.env.NODE_ENV];
```

### 2. Abstract Factory + Singleton

```javascript
class DatabaseFactory {
  static #instance = null;

  static getInstance() {
    if (!DatabaseFactory.#instance) {
      const type = process.env.DB_TYPE || 'postgres';
      DatabaseFactory.#instance = factories[type];
    }
    return DatabaseFactory.#instance;
  }
}

// Использование
const factory = DatabaseFactory.getInstance();
const db = factory.createDatabase(config);
```

### 3. Abstract Factory + Builder

```javascript
// Фабрика создаёт Builder для сложных объектов
class QueryBuilderFactory {
  createQueryBuilder() {
    return new PostgresQueryBuilder();
  }
}

const factory = new QueryBuilderFactory();
const builder = factory.createQueryBuilder();

const query = builder
  .select(['name', 'email'])
  .from('users')
  .where({ active: true })
  .build();
```

---

## Резюме

### Ключевые моменты

1. **Abstract Factory** создаёт **семейства объектов** с общим контрактом
2. Использует **две иерархии**: абстрактную и конкретную
3. В **JavaScript** можно упростить, убрав абстрактные классы
4. В **TypeScript** лучше использовать интерфейсы для строгой типизации
5. Часто комбинируется с паттерном **Strategy** для выбора фабрики

### Основные принципы

- **Слабое зацепление** (Low Coupling) — клиент не зависит от конкретных классов
- **Single Responsibility** — каждая фабрика отвечает за своё семейство
- **Open/Closed Principle** — легко добавлять новые фабрики без изменения существующего кода

### Практическое применение

Abstract Factory идеально подходит для:

- Работы с **разными базами данных** (PostgreSQL, MySQL, MongoDB, FileSystem)
- Рендеринга **UI для разных платформ** (Web, Desktop, Mobile)
- Абстрагирования **облачных провайдеров** (AWS, Azure, GCP)
- Создания **тестовых моков** вместо реальных сервисов

---

## Дополнительные ресурсы

- [Refactoring Guru: Abstract Factory](https://refactoring.guru/design-patterns/abstract-factory)
- [GoF Design Patterns Book](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)
- [TypeScript Design Patterns Repository](https://github.com/refactoringguru/design-patterns-typescript)

---

## Примеры кода из лекции

Все примеры кода из лекции доступны по ссылке в описании видео. Рекомендуется:

1. **Скачать исходники** и запустить их локально
2. **Модифицировать** код, добавив новые фабрики (например, для PostgreSQL)
3. **Попробовать** реализовать Abstract Factory для своего проекта
4. **Поэкспериментировать** с комбинацией паттернов (Strategy + Abstract Factory)

---

**Автор конспекта:** Claude Code
**Дата создания:** 2025-12-18
**Источник:** Лекция по паттерну Abstract Factory для JavaScript/TypeScript
