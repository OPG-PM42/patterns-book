# Паттерн Factory Method (Фабричный метод)

## Введение

**Factory Method** (Фабричный метод) — это порождающий паттерн проектирования из книги Gang of Four, который определяет общий интерфейс для создания объектов в суперклассе, позволяя подклассам изменять тип создаваемых объектов.

### Терминология

В различных языках программирования и контекстах можно встретить разные названия:
- **Factory Method** — фабричный метод
- **Factory** — фабрика (иногда так называют этот паттерн, но важно уточнять, что именно имеется в виду)
- **Abstract Factory** — абстрактная фабрика (это другой паттерн!)

**Важно**: Всегда уточняйте, о каком именно паттерне идёт речь — Factory Method или Abstract Factory, так как они решают разные задачи.

---

## Ключевые отличия от Abstract Factory

### Abstract Factory
- Имеет **несколько методов** для создания объектов
- Каждый метод создаёт экземпляры **разных контрактов**
- Создаёт семейства связанных объектов

### Factory Method
- **Один метод** для создания объектов
- Переход от использования оператора `new` к вызову метода
- Создаёт объекты одного контракта, но разных реализаций

---

## Основная идея паттерна

### Проблема

В коде напрямую используется оператор `new` для создания экземпляров классов:

```javascript
// Проблема: жёсткая связанность с конкретным классом
const product = new ConcreteProduct();
```

Это создаёт **сильное зацепление** (tight coupling) между кодом и конкретными классами.

### Решение

Factory Method предлагает заменить прямое создание объектов вызовом специального метода:

```javascript
// Решение: создание через фабричный метод
const product = creator.factoryMethod();
```

### Цели паттерна

1. **Снизить зацепление (coupling)** между компонентами системы
2. **Подставлять различные фабрики** в зависимости от контекста
3. **Делегировать создание объектов** другим частям программы
4. Подготовить код к использованию паттерна **Strategy**

---

## Классическая реализация

### Структура паттерна

```
AbstractCreator
    └── factoryMethod(): AbstractProduct
         ↑
         |
ConcreteCreator
    └── factoryMethod(): ConcreteProduct
                              ↓
                         (extends AbstractProduct)
```

### JavaScript реализация (классическая)

```javascript
// Абстрактный продукт
class AbstractProduct {
  constructor() {
    if (new.target === AbstractProduct) {
      throw new Error('Cannot instantiate abstract class');
    }
  }

  operation() {
    throw new Error('Abstract method must be implemented');
  }
}

// Конкретный продукт
class ConcreteProduct extends AbstractProduct {
  constructor(value) {
    super();
    this.value = value;
  }

  operation() {
    return `ConcreteProduct operation with value: ${this.value}`;
  }
}

// Абстрактный создатель (Creator)
class AbstractCreator {
  constructor() {
    if (new.target === AbstractCreator) {
      throw new Error('Cannot instantiate abstract class');
    }
  }

  // Фабричный метод — заглушка
  factoryMethod() {
    throw new Error('Abstract factory method must be implemented');
  }

  // Другие методы могут использовать фабричный метод
  someOperation() {
    const product = this.factoryMethod();
    return product.operation();
  }
}

// Конкретный создатель
class ConcreteCreator extends AbstractCreator {
  factoryMethod() {
    // Вызов оператора new скрыт внутри метода
    return new ConcreteProduct('example');
  }
}

// Использование
const creator = new ConcreteCreator();
const product = creator.factoryMethod();
console.log(product.operation());
// Вывод: ConcreteProduct operation with value: example
```

**Ключевая идея**: Мы переходим от прямого инстанцирования через `new` к вызову метода, который скрывает детали создания объекта.

---

## TypeScript реализация

TypeScript позволяет использовать настоящие абстрактные классы и методы, что делает код более типобезопасным.

### Пример с абстрактными классами

```typescript
// Абстрактный продукт
abstract class AbstractProduct {
  abstract operation(): string;
}

// Конкретный продукт
class ConcreteProduct extends AbstractProduct {
  constructor(private value: string) {
    super();
  }

  operation(): string {
    return `ConcreteProduct operation with value: ${this.value}`;
  }
}

// Абстрактный создатель
abstract class AbstractCreator {
  // Абстрактный фабричный метод
  abstract factoryMethod(): AbstractProduct;

  // Конкретный метод, который может использовать фабричный метод
  someOperation(): string {
    const product = this.factoryMethod();
    return product.operation();
  }
}

// Конкретный создатель
class ConcreteCreator extends AbstractCreator {
  factoryMethod(): AbstractProduct {
    return new ConcreteProduct('TypeScript example');
  }
}

// Использование
const creator: AbstractCreator = new ConcreteCreator();
const product = creator.factoryMethod();
console.log(product.operation());
// Вывод: ConcreteProduct operation with value: TypeScript example
```

**Преимущества TypeScript**:
- Компилятор проверяет реализацию всех абстрактных методов
- Не нужны проверки `new.target` и выбрасывание ошибок
- Лучшая поддержка IDE (автодополнение, рефакторинг)
- Типобезопасность на этапе компиляции

### Замечания по абстрактным классам

В абстрактных классах могут быть:
- **Абстрактные методы** — должны быть реализованы в наследниках
- **Конкретные методы** — наследники могут их переопределить (но не обязаны)

```typescript
abstract class Creator {
  // Абстрактный метод — обязателен для реализации
  abstract factoryMethod(): Product;

  // Конкретный метод — может быть переопределён
  createProduct(): Product {
    console.log('Default product creation logic');
    return this.factoryMethod();
  }
}
```

---

## JavaScript-идиоматичный подход

В JavaScript не обязательно использовать классы. Фабрика может быть простой функцией.

### Простая функция-фабрика

```javascript
// Продукт — простой объект или класс
class Product {
  constructor(type, value) {
    this.type = type;
    this.value = value;
  }

  describe() {
    return `Product of type "${this.type}" with value: ${this.value}`;
  }
}

// Фабричная функция
function createProduct(type, value) {
  return new Product(type, value);
}

// Экспорт из модуля
export { createProduct };

// Использование
import { createProduct } from './product-factory.js';

const product = createProduct('standard', 100);
console.log(product.describe());
// Вывод: Product of type "standard" with value: 100
```

### Коллекция фабрик

Один из распространённых подходов в JavaScript — хранить фабрики в объекте и выбирать нужную по ключу:

```javascript
// Разные типы продуктов
class StandardProduct {
  constructor(value) {
    this.type = 'standard';
    this.value = value;
  }

  calculate() {
    return this.value;
  }
}

class PremiumProduct {
  constructor(value) {
    this.type = 'premium';
    this.value = value;
  }

  calculate() {
    return this.value * 1.5; // Премиум-множитель
  }
}

// Коллекция фабрик
const productFactories = {
  standard: (value) => new StandardProduct(value),
  premium: (value) => new PremiumProduct(value),
};

// Функция для получения фабрики по ключу
function getFactory(type) {
  const factory = productFactories[type];
  if (!factory) {
    throw new Error(`Unknown product type: ${type}`);
  }
  return factory;
}

// Использование
const factory = getFactory('premium');
const product = factory(100);
console.log(product.calculate()); // Вывод: 150
```

### Передача фабрики как аргумента

Мощная идиома JavaScript — передавать фабрику в другие функции:

```javascript
// Функция высшего порядка, принимающая фабрику
function processItems(items, factory) {
  return items.map(item => {
    const product = factory(item);
    return product.calculate();
  });
}

// Использование с разными фабриками
const standardFactory = (value) => new StandardProduct(value);
const premiumFactory = (value) => new PremiumProduct(value);

const values = [10, 20, 30];

const standardResults = processItems(values, standardFactory);
console.log(standardResults); // [10, 20, 30]

const premiumResults = processItems(values, premiumFactory);
console.log(premiumResults); // [15, 30, 45]
```

**Преимущество**: Инстанцирование происходит не в нашем коде, а в другой части программы, которая получила фабрику как dependency.

---

## Практический пример: Database и Cursor

Рассмотрим реалистичный пример работы с базами данных и курсорами.

### Структура

```
Database (abstract)
  ├── constructor (abstract)
  ├── select() → Cursor (abstract)
  │
  ├── FileDatabase (concrete)
  │   ├── select() → FileCursor
  │
  └── SqlDatabase (concrete)
      └── select() → SqlCursor

Cursor (abstract)
  └── [Symbol.iterator]() (abstract)
      │
      ├── FileCursor (concrete)
      │   └── итерирует строки из файла
      │
      └── SqlCursor (concrete)
          └── итерирует записи из SQL
```

### TypeScript реализация

```typescript
// ========================================
// Абстрактный курсор
// ========================================
abstract class Cursor {
  // Абстрактный итератор
  abstract [Symbol.iterator](): Iterator<any>;
}

// ========================================
// Абстрактная база данных
// ========================================
abstract class Database {
  // Абстрактный конструктор
  constructor() {}

  // Фабричный метод — возвращает курсор
  abstract select(query: string): Cursor;
}

// ========================================
// Конкретная реализация: File Database
// ========================================
class FileCursor extends Cursor {
  constructor(private filePath: string) {
    super();
  }

  *[Symbol.iterator]() {
    // Чтение файла построчно
    const fs = require('fs');
    const lines = fs.readFileSync(this.filePath, 'utf-8').split('\n');

    for (const line of lines) {
      if (line.trim()) {
        yield line;
      }
    }
  }
}

class FileDatabase extends Database {
  constructor(private basePath: string) {
    super();
  }

  // Реализация фабричного метода
  select(query: string): Cursor {
    // query здесь — имя файла
    const filePath = `${this.basePath}/${query}`;
    return new FileCursor(filePath); // Создание курсора через new скрыто здесь
  }
}

// ========================================
// Конкретная реализация: SQL Database (mock)
// ========================================
class SqlCursor extends Cursor {
  constructor(private rows: any[]) {
    super();
  }

  *[Symbol.iterator]() {
    for (const row of this.rows) {
      yield row;
    }
  }
}

class SqlDatabase extends Database {
  constructor(private connectionString: string) {
    super();
  }

  // Реализация фабричного метода
  select(query: string): Cursor {
    // Mock: имитируем выполнение SQL запроса
    console.log(`Executing SQL: ${query}`);
    const mockRows = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' },
    ];
    return new SqlCursor(mockRows); // Создание курсора
  }
}

// ========================================
// Использование
// ========================================

// Работа с файловой БД
const fileDb: Database = new FileDatabase('./data');
const fileCursor = fileDb.select('users.txt');

for (const line of fileCursor) {
  console.log('File row:', line);
}

// Работа с SQL БД
const sqlDb: Database = new SqlDatabase('host=localhost;db=test');
const sqlCursor = sqlDb.select('SELECT * FROM users');

for (const row of sqlCursor) {
  console.log('SQL row:', row);
}
```

**Ключевые моменты**:
1. Клиентский код работает с абстракцией `Database` и не знает о конкретных классах
2. Метод `select()` — это фабричный метод, который создаёт конкретные курсоры
3. Вместо `new FileCursor()` в клиентском коде мы вызываем `db.select()`
4. Легко добавить новые типы БД без изменения клиентского кода

### JavaScript упрощённая версия

```javascript
// Фабрики для создания курсоров
function createFileCursor(filePath) {
  const fs = require('fs');
  const lines = fs.readFileSync(filePath, 'utf-8').split('\n');

  return {
    [Symbol.iterator]: function* () {
      for (const line of lines) {
        if (line.trim()) yield line;
      }
    }
  };
}

function createSqlCursor(query) {
  console.log(`Executing SQL: ${query}`);
  const mockRows = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
  ];

  return {
    [Symbol.iterator]: function* () {
      for (const row of mockRows) {
        yield row;
      }
    }
  };
}

// Фабрика баз данных
function createDatabase(type, config) {
  if (type === 'file') {
    return {
      select(query) {
        return createFileCursor(`${config.basePath}/${query}`);
      }
    };
  }

  if (type === 'sql') {
    return {
      select(query) {
        return createSqlCursor(query);
      }
    };
  }

  throw new Error(`Unknown database type: ${type}`);
}

// Использование
const db = createDatabase('sql', {});
const cursor = db.select('SELECT * FROM users');

for (const row of cursor) {
  console.log('Row:', row);
}
```

---

## Преимущества паттерна Factory Method

### 1. Снижение зацепления (Low Coupling)

```javascript
// ❌ Плохо: жёсткая связанность
class OrderProcessor {
  processOrder(orderData) {
    const order = new StandardOrder(orderData); // Зависимость от конкретного класса
    return order.process();
  }
}

// ✅ Хорошо: слабое зацепление через фабрику
class OrderProcessor {
  constructor(orderFactory) {
    this.orderFactory = orderFactory;
  }

  processOrder(orderData) {
    const order = this.orderFactory.create(orderData); // Абстракция
    return order.process();
  }
}
```

### 2. Гибкость и расширяемость

Легко добавлять новые типы продуктов без изменения существующего кода:

```javascript
// Добавление нового типа продукта
class ExpressOrder extends Order {
  process() {
    return 'Express order processed with priority!';
  }
}

// Добавление новой фабрики
class ExpressOrderFactory {
  create(data) {
    return new ExpressOrder(data);
  }
}

// Клиентский код не меняется!
const processor = new OrderProcessor(new ExpressOrderFactory());
```

### 3. Единая точка создания объектов

Вся логика создания сосредоточена в одном месте:

```javascript
class UserFactory {
  create(userData) {
    // Валидация
    if (!userData.email) {
      throw new Error('Email is required');
    }

    // Нормализация данных
    userData.email = userData.email.toLowerCase();

    // Логирование
    console.log('Creating user:', userData.email);

    // Создание
    return new User(userData);
  }
}
```

### 4. Подготовка к тестированию

Легко подменить реальные объекты на mock-объекты:

```javascript
// Продакшн
const realDatabase = new SqlDatabase('production-connection');
const service = new DataService(realDatabase);

// Тестирование
const mockDatabase = new MockDatabase();
const service = new DataService(mockDatabase);
```

### 5. Инверсия управления (IoC)

Создание объектов делегируется внешнему коду:

```javascript
function processData(items, factory) {
  // Эта функция не знает, какие именно объекты создаются
  return items.map(item => {
    const processor = factory(item);
    return processor.process();
  });
}

// Внедрение зависимости
const results = processData(data, createStandardProcessor);
```

---

## Недостатки и когда НЕ использовать

### Недостатки

1. **Усложнение кода** — появляется больше классов и абстракций
2. **Избыточность для простых случаев** — если у вас только один тип продукта
3. **Косвенность** — менее очевидно, какой именно класс будет создан

### Когда НЕ использовать

```javascript
// ❌ Избыточно для простого случая
class ButtonFactory {
  createButton() {
    return new Button(); // Всегда одна и та же кнопка
  }
}

// ✅ Просто используйте конструктор
const button = new Button();
```

### Когда использовать

1. **Заранее неизвестен тип создаваемых объектов**
2. **Нужно делегировать создание подклассам**
3. **Хотите локализовать логику создания** в одном месте
4. **Готовитесь к расширению** системы новыми типами
5. **Нужна возможность переключения** между разными реализациями

---

## Связь с другими паттернами

### Factory Method + Strategy

Factory Method часто используется совместно с паттерном Strategy:

```javascript
// Стратегии обработки
class StandardProcessingStrategy {
  process(data) {
    return data.toUpperCase();
  }
}

class FastProcessingStrategy {
  process(data) {
    return data.substring(0, 10);
  }
}

// Фабрика стратегий
class StrategyFactory {
  create(type) {
    switch (type) {
      case 'standard':
        return new StandardProcessingStrategy();
      case 'fast':
        return new FastProcessingStrategy();
      default:
        throw new Error(`Unknown strategy: ${type}`);
    }
  }
}

// Использование
class DataProcessor {
  constructor(strategyFactory) {
    this.strategyFactory = strategyFactory;
  }

  process(data, strategyType) {
    const strategy = this.strategyFactory.create(strategyType);
    return strategy.process(data);
  }
}

const processor = new DataProcessor(new StrategyFactory());
console.log(processor.process('hello world', 'standard')); // HELLO WORLD
console.log(processor.process('hello world', 'fast'));     // hello worl
```

### Factory Method vs Abstract Factory

| Аспект | Factory Method | Abstract Factory |
|--------|----------------|------------------|
| Методов | Один метод | Несколько методов |
| Продуктов | Один тип продукта | Семейство связанных продуктов |
| Наследование | Использует наследование | Использует композицию |
| Пример | `createButton()` | `createButton()`, `createInput()`, `createCheckbox()` |

### Factory Method + Template Method

```javascript
abstract class DataImporter {
  // Template Method
  import(filePath) {
    const data = this.readFile(filePath);
    const parsed = this.parse(data);
    const validated = this.validate(parsed);
    return this.save(validated);
  }

  // Factory Method
  abstract createParser(): Parser;

  parse(data) {
    const parser = this.createParser(); // Фабричный метод
    return parser.parse(data);
  }

  abstract readFile(path): string;
  abstract validate(data): any;
  abstract save(data): void;
}
```

---

## Реальные примеры из практики

### Пример 1: Логирование

```javascript
// Абстрактный логгер
class Logger {
  log(message) {
    throw new Error('Method must be implemented');
  }
}

// Конкретные логгеры
class ConsoleLogger extends Logger {
  log(message) {
    console.log(`[CONSOLE] ${new Date().toISOString()} - ${message}`);
  }
}

class FileLogger extends Logger {
  constructor(filePath) {
    super();
    this.filePath = filePath;
  }

  log(message) {
    const fs = require('fs');
    const logEntry = `${new Date().toISOString()} - ${message}\n`;
    fs.appendFileSync(this.filePath, logEntry);
  }
}

class CloudLogger extends Logger {
  log(message) {
    // Отправка в облачный сервис
    console.log(`[CLOUD] Sending to cloud: ${message}`);
  }
}

// Фабрика логгеров
class LoggerFactory {
  static create(type, options = {}) {
    switch (type) {
      case 'console':
        return new ConsoleLogger();
      case 'file':
        return new FileLogger(options.filePath || './app.log');
      case 'cloud':
        return new CloudLogger();
      default:
        throw new Error(`Unknown logger type: ${type}`);
    }
  }
}

// Использование
const logger = LoggerFactory.create('file', { filePath: './logs/app.log' });
logger.log('Application started');

// Легко переключиться на другой логгер
const cloudLogger = LoggerFactory.create('cloud');
cloudLogger.log('Critical error occurred');
```

### Пример 2: HTTP-клиенты

```javascript
// Абстрактный HTTP-клиент
class HttpClient {
  get(url) {
    throw new Error('Method must be implemented');
  }

  post(url, data) {
    throw new Error('Method must be implemented');
  }
}

// Реализация с fetch
class FetchHttpClient extends HttpClient {
  async get(url) {
    const response = await fetch(url);
    return response.json();
  }

  async post(url, data) {
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    return response.json();
  }
}

// Реализация с axios
class AxiosHttpClient extends HttpClient {
  constructor() {
    super();
    this.axios = require('axios');
  }

  async get(url) {
    const response = await this.axios.get(url);
    return response.data;
  }

  async post(url, data) {
    const response = await this.axios.post(url, data);
    return response.data;
  }
}

// Фабрика HTTP-клиентов
class HttpClientFactory {
  static create(type = 'fetch') {
    if (type === 'fetch') {
      return new FetchHttpClient();
    }
    if (type === 'axios') {
      return new AxiosHttpClient();
    }
    throw new Error(`Unknown HTTP client type: ${type}`);
  }
}

// API сервис использует фабрику
class ApiService {
  constructor(httpClientFactory) {
    this.httpClient = httpClientFactory.create('fetch');
  }

  async getUsers() {
    return this.httpClient.get('/api/users');
  }

  async createUser(userData) {
    return this.httpClient.post('/api/users', userData);
  }
}
```

### Пример 3: Парсеры данных

```javascript
// Интерфейс парсера
class DataParser {
  parse(data) {
    throw new Error('Method must be implemented');
  }
}

// JSON парсер
class JsonParser extends DataParser {
  parse(data) {
    try {
      return JSON.parse(data);
    } catch (error) {
      throw new Error(`JSON parsing failed: ${error.message}`);
    }
  }
}

// CSV парсер
class CsvParser extends DataParser {
  parse(data) {
    const lines = data.split('\n');
    const headers = lines[0].split(',');

    return lines.slice(1).map(line => {
      const values = line.split(',');
      return headers.reduce((obj, header, index) => {
        obj[header.trim()] = values[index]?.trim() || '';
        return obj;
      }, {});
    });
  }
}

// XML парсер (упрощённый)
class XmlParser extends DataParser {
  parse(data) {
    // Упрощённая реализация
    return { xml: 'parsed', data };
  }
}

// Фабрика парсеров по расширению файла
class ParserFactory {
  static createByExtension(filePath) {
    const extension = filePath.split('.').pop().toLowerCase();

    switch (extension) {
      case 'json':
        return new JsonParser();
      case 'csv':
        return new CsvParser();
      case 'xml':
        return new XmlParser();
      default:
        throw new Error(`Unsupported file extension: ${extension}`);
    }
  }
}

// Использование
function loadData(filePath) {
  const fs = require('fs');
  const content = fs.readFileSync(filePath, 'utf-8');

  const parser = ParserFactory.createByExtension(filePath);
  return parser.parse(content);
}

const jsonData = loadData('./data.json');
const csvData = loadData('./data.csv');
```

---

## Распространённые ошибки

### Ошибка 1: Забывают скрывать new

```javascript
// ❌ Неправильно: клиент всё ещё видит new
class ProductFactory {
  factoryMethod() {
    return Product; // Возвращаем класс, а не экземпляр!
  }
}

const Factory = new ProductFactory();
const product = new (Factory.factoryMethod())(); // Клиент использует new

// ✅ Правильно: new скрыт внутри фабрики
class ProductFactory {
  factoryMethod() {
    return new Product(); // Возвращаем готовый экземпляр
  }
}

const factory = new ProductFactory();
const product = factory.factoryMethod(); // Клиент не знает про new
```

### Ошибка 2: Путают Factory Method и Abstract Factory

```javascript
// ❌ Это Abstract Factory, а не Factory Method!
class GUIFactory {
  createButton() {
    return new Button();
  }

  createInput() {
    return new Input();
  }

  createCheckbox() {
    return new Checkbox();
  }
}

// ✅ Это Factory Method — один метод для одного типа продукта
class ButtonFactory {
  createButton(type) {
    if (type === 'primary') return new PrimaryButton();
    if (type === 'secondary') return new SecondaryButton();
    throw new Error(`Unknown button type: ${type}`);
  }
}
```

### Ошибка 3: Чрезмерное использование

```javascript
// ❌ Избыточно для простого случая
class StringFactory {
  createString(value) {
    return new String(value);
  }
}

// ✅ Просто используйте литералы
const str = 'Hello, world!';
```

---

## Контрольные вопросы

1. **В чём главное отличие Factory Method от Abstract Factory?**
   - Factory Method — один метод для одного типа продукта
   - Abstract Factory — множество методов для семейства продуктов

2. **Почему в JavaScript Factory Method может быть простой функцией?**
   - JavaScript не требует строгой иерархии классов
   - Функции — это объекты первого класса
   - Можно передавать функции как аргументы

3. **Когда стоит использовать Factory Method?**
   - Тип создаваемого объекта определяется во время выполнения
   - Нужно делегировать создание подклассам
   - Хотите изолировать логику создания

4. **Как Factory Method связан с паттерном Strategy?**
   - Factory Method часто создаёт экземпляры стратегий
   - Позволяет подменять стратегии во время выполнения

---

## Резюме

### Ключевые концепции

1. **Замена прямого инстанцирования** методом создания
2. **Снижение зацепления** между компонентами
3. **Делегирование создания** подклассам или внешнему коду
4. **Гибкость** при добавлении новых типов продуктов

### Когда использовать

- Заранее неизвестен точный тип создаваемых объектов
- Нужна возможность расширения системы новыми типами
- Хотите централизовать логику создания объектов
- Требуется возможность подмены реализации (тестирование, конфигурация)

### JavaScript особенности

- Фабрика может быть простой функцией
- Можно передавать фабрики как аргументы (высшие функции)
- Коллекции фабрик в объектах — распространённый паттерн
- Не обязательно использовать наследование классов

### TypeScript преимущества

- Настоящие абстрактные классы и методы
- Проверка типов на этапе компиляции
- Лучшая поддержка IDE
- Не нужны проверки с `new.target`

---

## Дополнительные материалы

### Связанные паттерны

- **Abstract Factory** — создание семейств связанных объектов
- **Strategy** — выбор алгоритма во время выполнения
- **Template Method** — определение скелета алгоритма
- **Prototype** — создание объектов через клонирование

### Принципы SOLID

Factory Method помогает соблюдать:
- **Single Responsibility** — одна причина для изменения
- **Open/Closed** — открыт для расширения, закрыт для модификации
- **Dependency Inversion** — зависимость от абстракций, а не от конкретных классов

### Паттерны GRASP

- **Creator** — кто отвечает за создание объекта?
- **Low Coupling** — минимизация зависимостей
- **High Cohesion** — логика создания в одном месте
- **Polymorphism** — использование полиморфизма для вариативности

---

## Практические упражнения

### Упражнение 1: Система уведомлений

Создайте фабрику для системы уведомлений, которая может отправлять уведомления через:
- Email
- SMS
- Push-уведомления
- Slack

### Упражнение 2: Обработчики файлов

Реализуйте фабрику обработчиков файлов для различных форматов:
- Изображения (PNG, JPEG, GIF)
- Документы (PDF, DOCX, TXT)
- Видео (MP4, AVI, MKV)

### Упражнение 3: Валидаторы форм

Создайте систему валидаторов с использованием Factory Method:
- Email валидатор
- Телефон валидатор
- Дата валидатор
- Кастомный валидатор

---

**Успехов в изучении паттернов проектирования!**
