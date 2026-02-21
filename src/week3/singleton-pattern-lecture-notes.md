# Паттерн Singleton (Одиночка) — GoF Patterns

## Обзор

Паттерн Singleton (Одиночка) — это один из самых простых порождающих паттернов из коллекции Gang of Four (GoF). Его основная задача — гарантировать, что у класса существует только один экземпляр в системе, и предоставить глобальную точку доступа к этому экземпляру. Независимо от того, сколько раз вызывается конструктор, всегда возвращается один и тот же объект.

В JavaScript и TypeScript существует гораздо больше способов реализации Singleton, чем в традиционных типизированных языках, благодаря особенностям языка и системы модульности.

---

## Основная концепция

**Цель паттерна**: Обеспечить существование единственного экземпляра (instance) в системе.

**Принцип работы**: Сколько раз ни вызывай конструктор, получается один и тот же экземпляр объекта.

---

## Способы реализации Singleton в JavaScript/TypeScript

### 1. Модульная система (Module-based Singleton)

Самый удобный и естественный способ для JavaScript — использование системы модулей. Система модульности автоматически делает из каждого модуля Singleton.

#### Особенности модульной системы:

- Всё, что экспортируется из модуля, при каждом импорте в других местах возвращает **тот же самый объект**
- Пока не сброшен кэш модулей, объект остаётся единственным
- Можно принудительно сбросить кэш, но обычно это не требуется
- Любой runtime (Node.js, браузеры) кэширует модули по умолчанию

#### Пример с CommonJS:

```javascript
// singleton.js
const singleton = {
  data: [],

  addData(item) {
    this.data.push(item);
  },

  getData() {
    return this.data;
  }
};

module.exports = singleton;
```

**Использование:**

```javascript
// file1.js
const singleton = require('./singleton');
singleton.addData('item1');

// file2.js
const singleton = require('./singleton');
singleton.addData('item2');
console.log(singleton.getData()); // ['item1', 'item2']
// Один и тот же объект во всех модулях!
```

#### Пример с ES6 модулями:

```javascript
// singleton.mjs
class Singleton {
  constructor() {
    this.data = [];
  }

  addData(item) {
    this.data.push(item);
  }

  getData() {
    return this.data;
  }
}

// Создаём экземпляр и экспортируем его
const singleton = new Singleton();
export default singleton;
```

**Использование:**

```javascript
// app.mjs
import singleton from './singleton.mjs';

singleton.addData('test');
console.log(singleton.getData()); // ['test']

// В любом другом модуле
import singleton from './singleton.mjs';
// Получим тот же самый экземпляр с теми же данными
```

---

### 2. Global Scope Singleton

Можно использовать глобальную область видимости (`global`, `globalThis`, `window`).

#### Пример:

```javascript
// Записываем singleton в глобальный объект
globalThis.appSingleton = {
  config: {},

  setConfig(key, value) {
    this.config[key] = value;
  },

  getConfig(key) {
    return this.config[key];
  }
};

// Использование из любого места
globalThis.appSingleton.setConfig('apiUrl', 'https://api.example.com');
```

#### Недостатки глобального подхода:

- **Плохо для тестирования** — глобальное состояние трудно изолировать между тестами
- **Замусоривание глобальной области видимости** — риск конфликтов имён
- **Race conditions (состояния гонки)** — если у singleton есть изменяемое состояние, из разных мест можно его модифицировать одновременно
- **Отсутствие владельца** — никто не контролирует и не защищает состояние примитивами синхронизации
- **Сложность очистки** — при юнит-тестах нужно вручную очищать состояние

---

### 3. Статические поля класса

Сохранение ссылки на экземпляр в статических полях класса.

#### Пример с TypeScript (современный синтаксис):

```typescript
class Singleton {
  // Приватное статическое поле для хранения экземпляра
  private static instance: Singleton | null = null;

  // Приватный конструктор предотвращает создание экземпляров извне
  private constructor() {
    // Инициализация
  }

  // Статический метод для получения единственного экземпляра
  public static getInstance(): Singleton {
    if (!Singleton.instance) {
      Singleton.instance = new Singleton();
    }
    return Singleton.instance;
  }

  public someBusinessLogic(): void {
    console.log('Executing business logic');
  }
}

// Использование
const s1 = Singleton.getInstance();
const s2 = Singleton.getInstance();

console.log(s1 === s2); // true - один и тот же экземпляр
```

#### Пример с приватными полями ES2022:

```typescript
class Singleton {
  static #instance: Singleton;

  private constructor() { }

  public static get instance(): Singleton {
    if (!Singleton.#instance) {
      Singleton.#instance = new Singleton();
    }
    return Singleton.#instance;
  }

  public someBusinessLogic() {
    console.log("Executing business logic");
  }
}

// Использование
const s1 = Singleton.instance;
const s2 = Singleton.instance;

if (s1 === s2) {
  console.log('Singleton работает, обе переменные содержат один экземпляр.');
}
```

---

### 4. Замыкания (Closures)

Использование замыканий для сокрытия экземпляра.

#### Пример 1: Прототипный подход

```javascript
// Функция-конструктор с замыканием
function Singleton() {
  // Деструктурируем свойство instance из самой функции
  const { instance } = Singleton;

  // Если экземпляр уже существует, возвращаем его
  if (instance) {
    return instance;
  }

  // Иначе сохраняем текущий контекст (this) как instance
  Singleton.instance = this;

  // Инициализация свойств
  this.data = [];
}

// Добавляем методы в прототип
Singleton.prototype.addData = function(item) {
  this.data.push(item);
};

// Использование
const s1 = new Singleton();
const s2 = new Singleton();
console.log(s1 === s2); // true
```

#### Пример 2: Функция, возвращающая функцию-конструктор

```javascript
const Singleton = (function() {
  // Локальная переменная для хранения экземпляра
  let instance;

  // Внутренний конструктор
  function Constructor() {
    // Инициализация
    this.data = [];
  }

  Constructor.prototype.addData = function(item) {
    this.data.push(item);
  };

  // Возвращаем функцию, которая проверяет и создаёт экземпляр
  return function() {
    if (!instance) {
      instance = new Constructor();
    }
    return instance;
  };
})();

// Использование
const s1 = new Singleton();
const s2 = new Singleton();
console.log(s1 === s2); // true
```

**Важно**: Нельзя использовать стрелочные функции для конструкторов, так как они не имеют своего `this` контекста.

#### Пример 3: IIFE с классом

```javascript
const Singleton = (function() {
  // Переменная для хранения экземпляра в замыкании
  let instance;

  // Определяем класс внутри замыкания
  class SingletonClass {
    constructor() {
      // Если экземпляр уже существует, возвращаем его
      if (instance) {
        return instance;
      }

      // Сохраняем текущий экземпляр
      instance = this;

      // Инициализация
      this.timestamp = Date.now();
    }

    someMethod() {
      console.log('Method called at', this.timestamp);
    }
  }

  // Возвращаем класс из IIFE
  return SingletonClass;
})();

// Использование
const s1 = new Singleton();
const s2 = new Singleton();
console.log(s1 === s2); // true
```

---

### 5. Объектный литерал (Object Literal)

Самый простой способ — использовать объектный литерал без классов.

#### Пример 1: Простое замыкание с объектом

```javascript
const Singleton = (function() {
  // Создаём объект один раз в замыкании
  const instance = {
    data: [],
    config: {},

    addData(item) {
      this.data.push(item);
    },

    setConfig(key, value) {
      this.config[key] = value;
    }
  };

  // Возвращаем функцию, которая всегда отдаёт этот объект
  return function() {
    return instance;
  };
})();

// Использование (без new)
const s1 = Singleton();
const s2 = Singleton();
console.log(s1 === s2); // true

s1.addData('test');
console.log(s2.data); // ['test'] - тот же объект
```

#### Пример 2: Передача объекта как параметр

```javascript
const Singleton = (function(instance) {
  // Возвращаем функцию, которая возвращает переданный параметр
  return function() {
    return instance;
  };
})({ /* объект singleton */
  data: [],

  addData(item) {
    this.data.push(item);
  },

  getData() {
    return this.data;
  }
});

// Использование
const s1 = Singleton();
const s2 = Singleton();
console.log(s1 === s2); // true
```

---

## Преимущества паттерна Singleton

### 1. Единственный экземпляр

Гарантируется, что в любом случае в системе будет существовать только один экземпляр объекта.

```javascript
class DatabaseConnection {
  private static instance: DatabaseConnection;
  private connection: any;

  private constructor() {
    // Дорогостоящее создание подключения
    this.connection = this.createConnection();
  }

  public static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  private createConnection() {
    console.log('Creating database connection...');
    return { connected: true };
  }

  public query(sql: string) {
    return `Executing: ${sql}`;
  }
}

// Подключение создаётся только один раз
const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();
console.log(db1 === db2); // true
```

### 2. Глобальная точка доступа

Предоставляется единая точка доступа к ресурсу из любой части приложения.

### 3. Ленивая инициализация

Экземпляр создаётся только тогда, когда он действительно нужен.

```javascript
class Logger {
  private static instance: Logger | null = null;

  private constructor() {
    console.log('Logger initialized');
  }

  public static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger(); // Создаётся только при первом вызове
    }
    return Logger.instance;
  }

  public log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}

// Logger не создан
console.log('App started');

// Logger создаётся только здесь
const logger = Logger.getInstance(); // Выведет: "Logger initialized"
logger.log('First message');
```

---

## Недостатки паттерна Singleton

### 1. Глобальный доступ откуда угодно

Любая часть кода может получить доступ к singleton, что затрудняет отслеживание зависимостей.

### 2. Race Conditions (Состояния гонки)

Если у singleton есть изменяемое состояние, возможны проблемы с конкурентным доступом:

```javascript
class Counter {
  private static instance: Counter;
  private count = 0;

  private constructor() {}

  public static getInstance(): Counter {
    if (!Counter.instance) {
      Counter.instance = new Counter();
    }
    return Counter.instance;
  }

  public increment(): void {
    this.count++;
  }

  public getCount(): number {
    return this.count;
  }
}

// Проблема: из разных мест можно изменять состояние
const counter1 = Counter.getInstance();
const counter2 = Counter.getInstance();

counter1.increment(); // count = 1
counter2.increment(); // count = 2
// Состояние разделяется между всеми потребителями
```

### 3. Проблемы с юнит-тестированием

**Проблема изоляции тестов:**

```javascript
// test1.spec.js
test('test 1', () => {
  const config = Config.getInstance();
  config.set('apiUrl', 'http://test1.com');
  // ...тест использует эту конфигурацию
});

// test2.spec.js
test('test 2', () => {
  const config = Config.getInstance();
  // Получаем тот же экземпляр с состоянием из предыдущего теста!
  console.log(config.get('apiUrl')); // 'http://test1.com' - не то что ожидалось
});
```

**Решение — метод сброса:**

```typescript
class Config {
  private static instance: Config | null = null;
  private settings: Map<string, any> = new Map();

  private constructor() {}

  public static getInstance(): Config {
    if (!Config.instance) {
      Config.instance = new Config();
    }
    return Config.instance;
  }

  // Метод для сброса состояния (для тестов)
  public static reset(): void {
    Config.instance = null;
  }

  public set(key: string, value: any): void {
    this.settings.set(key, value);
  }

  public get(key: string): any {
    return this.settings.get(key);
  }

  public clear(): void {
    this.settings.clear();
  }
}

// В тестах:
afterEach(() => {
  Config.reset(); // Сбрасываем singleton перед каждым тестом
});
```

### 4. Сложность очистки глобального состояния

Singleton остаётся в памяти, пока работает приложение. Если другие объекты создаются и уничтожаются, singleton продолжает жить.

### 5. Скрытые зависимости

Использование singleton скрывает реальные зависимости класса, что усложняет понимание кода.

```typescript
class UserService {
  // Неявная зависимость от Logger
  public createUser(name: string) {
    Logger.getInstance().log(`Creating user: ${name}`);
    // ...
  }
}

// Лучше использовать Dependency Injection:
class UserService {
  constructor(private logger: Logger) {}

  public createUser(name: string) {
    this.logger.log(`Creating user: ${name}`);
    // ...
  }
}
```

---

## Практические примеры использования

### Пример 1: Менеджер конфигурации

```typescript
class ConfigManager {
  private static instance: ConfigManager;
  private config: Map<string, any>;

  private constructor() {
    this.config = new Map();
    this.loadDefaultConfig();
  }

  public static getInstance(): ConfigManager {
    if (!ConfigManager.instance) {
      ConfigManager.instance = new ConfigManager();
    }
    return ConfigManager.instance;
  }

  private loadDefaultConfig(): void {
    this.config.set('appName', 'MyApp');
    this.config.set('version', '1.0.0');
    this.config.set('debug', false);
  }

  public get(key: string): any {
    return this.config.get(key);
  }

  public set(key: string, value: any): void {
    this.config.set(key, value);
  }

  public has(key: string): boolean {
    return this.config.has(key);
  }
}

// Использование
const config = ConfigManager.getInstance();
config.set('apiUrl', 'https://api.example.com');

// В другой части приложения
const config2 = ConfigManager.getInstance();
console.log(config2.get('apiUrl')); // 'https://api.example.com'
```

### Пример 2: Пул соединений с базой данных

```typescript
class ConnectionPool {
  private static instance: ConnectionPool;
  private connections: Connection[] = [];
  private readonly maxConnections = 10;

  private constructor() {
    this.initializePool();
  }

  public static getInstance(): ConnectionPool {
    if (!ConnectionPool.instance) {
      ConnectionPool.instance = new ConnectionPool();
    }
    return ConnectionPool.instance;
  }

  private initializePool(): void {
    console.log('Initializing connection pool...');
    for (let i = 0; i < this.maxConnections; i++) {
      this.connections.push(this.createConnection());
    }
  }

  private createConnection(): Connection {
    return {
      id: Math.random().toString(36).substr(2, 9),
      inUse: false,
      query: (sql: string) => `Result for: ${sql}`
    };
  }

  public getConnection(): Connection | null {
    const available = this.connections.find(conn => !conn.inUse);
    if (available) {
      available.inUse = true;
      return available;
    }
    return null;
  }

  public releaseConnection(connection: Connection): void {
    connection.inUse = false;
  }
}

interface Connection {
  id: string;
  inUse: boolean;
  query: (sql: string) => string;
}

// Использование
const pool = ConnectionPool.getInstance();
const conn = pool.getConnection();
if (conn) {
  console.log(conn.query('SELECT * FROM users'));
  pool.releaseConnection(conn);
}
```

### Пример 3: Логгер

```typescript
enum LogLevel {
  DEBUG,
  INFO,
  WARN,
  ERROR
}

class Logger {
  private static instance: Logger;
  private logLevel: LogLevel = LogLevel.INFO;

  private constructor() {}

  public static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  public setLogLevel(level: LogLevel): void {
    this.logLevel = level;
  }

  private log(level: LogLevel, message: string): void {
    if (level >= this.logLevel) {
      const timestamp = new Date().toISOString();
      const levelName = LogLevel[level];
      console.log(`[${timestamp}] [${levelName}] ${message}`);
    }
  }

  public debug(message: string): void {
    this.log(LogLevel.DEBUG, message);
  }

  public info(message: string): void {
    this.log(LogLevel.INFO, message);
  }

  public warn(message: string): void {
    this.log(LogLevel.WARN, message);
  }

  public error(message: string): void {
    this.log(LogLevel.ERROR, message);
  }
}

// Использование
const logger = Logger.getInstance();
logger.setLogLevel(LogLevel.DEBUG);

logger.debug('This is a debug message');
logger.info('Application started');
logger.warn('Warning: low memory');
logger.error('Error occurred');
```

---

## Сравнение различных реализаций

| Подход | Преимущества | Недостатки | Когда использовать |
|--------|--------------|------------|-------------------|
| **Модули (ES6/CommonJS)** | Просто, естественно для JS, автоматическое кэширование | Невозможность контроля создания | Большинство случаев в Node.js |
| **Статические поля класса** | Явный контроль, приватный конструктор | Более многословно | Когда нужен явный контроль |
| **Замыкания** | Скрывает детали реализации | Сложнее для понимания | Легаси код или специфические случаи |
| **Объектный литерал** | Максимально просто | Нет возможности ленивой инициализации | Простые конфигурации |
| **Global scope** | Доступно везде | Засоряет глобальную область, проблемы с тестами | Избегать в production коде |

---

## Альтернативы Singleton

### 1. Dependency Injection (Внедрение зависимостей)

Вместо использования singleton, передавайте зависимости явно:

```typescript
// Плохо: использование Singleton
class UserService {
  createUser(name: string) {
    const db = Database.getInstance(); // Скрытая зависимость
    const logger = Logger.getInstance(); // Скрытая зависимость
    logger.log('Creating user');
    db.save({ name });
  }
}

// Хорошо: Dependency Injection
class UserService {
  constructor(
    private db: Database,
    private logger: Logger
  ) {}

  createUser(name: string) {
    this.logger.log('Creating user');
    this.db.save({ name });
  }
}

// Явное создание зависимостей
const db = new Database();
const logger = new Logger();
const userService = new UserService(db, logger);
```

### 2. Модульные экспорты

Используйте систему модулей вместо сложных паттернов:

```javascript
// config.js
export const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000
};

// app.js
import { config } from './config.js';
console.log(config.apiUrl);
```

---

## Ключевые выводы

1. **Singleton обеспечивает единственный экземпляр класса** в системе с глобальной точкой доступа

2. **В JavaScript много способов реализации** благодаря гибкости языка и системе модулей

3. **Модульная система — самый естественный способ** для JavaScript/Node.js

4. **Основные проблемы**:
   - Глобальное состояние
   - Race conditions
   - Сложность тестирования
   - Скрытые зависимости

5. **Когда использовать**:
   - Менеджеры конфигурации
   - Логгеры
   - Пулы соединений
   - Кэш-менеджеры
   - Реестры/регистры

6. **Когда избегать**:
   - Когда возможна Dependency Injection
   - Когда важна изоляция тестов
   - Когда состояние должно быть локальным

7. **Лучшая практика**: Предпочитайте модульную систему сложным реализациям с замыканиями, используйте явное внедрение зависимостей где возможно

---

## Дополнительные ресурсы

### Рекомендуемые паттерны для изучения:
- **Factory** — для контролируемого создания объектов
- **Dependency Injection** — для управления зависимостями
- **Module Pattern** — для инкапсуляции кода

### Связанные концепции:
- Система модулей CommonJS и ES6
- Замыкания в JavaScript
- Статические члены классов
- Приватные поля в TypeScript/JavaScript
