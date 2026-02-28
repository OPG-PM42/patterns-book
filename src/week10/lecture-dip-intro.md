# Принцип инверсии зависимостей (DIP) — SOLID

## Введение

Принцип инверсии зависимостей (Dependency Inversion Principle, DIP) — один из самых сложных, недопонятых и при этом наиболее полезных принципов SOLID. Именно он открывает путь к построению настоящей архитектуры приложений.

Если остальные четыре принципа SOLID касаются преимущественно структуры отдельных классов и модулей, то DIP выводит нас на уровень выше: он описывает, как должны быть организованы **слои** приложения и как они должны взаимодействовать между собой.

Цель принципа — сделать логику предметной области (доменный слой) переносимой и независимой: её можно запустить на фронтенде, бэкенде, в мобильном приложении или даже встроить в программу, написанную на другом языке, если тот поддерживает JavaScript-рантайм.

---

## Принцип инверсии зависимостей

Формулировка принципа состоит из двух частей:

1. **Модули верхних уровней не должны зависеть от модулей нижних уровней.** Оба должны зависеть от абстракций (интерфейсов).
2. **Абстракции не должны зависеть от деталей.** Детали (конкретные реализации) должны зависеть от абстракций.

### Что такое "верхний" и "нижний" уровень?

В типичном приложении можно выделить несколько слоёв:

```
┌─────────────────────────────┐
│      Доменная модель        │  ← верхний уровень
│   (Employee, Payroll, ...)  │
├─────────────────────────────┤
│     Бизнес-сервисы          │  ← средний уровень
│  (PayrollService, ...)      │
├─────────────────────────────┤
│      Инфраструктура         │  ← нижний уровень
│  (Logger, Storage, Server)  │
└─────────────────────────────┘
```

В архитектуре "луковица" (Onion Architecture) или чистой архитектуре (Clean Architecture) доменная модель находится в центре и **не должна знать ничего** о слоях снаружи: ни о конкретной СУБД, ни о сетевом протоколе, ни о способе логирования, ни об интерфейсе пользователя.

### Почему это важно

Когда верхний уровень напрямую импортирует нижний — между ними возникает жёсткая связь (tight coupling). Стрелки зависимостей направлены вниз. Если нижний уровень изменится (например, вы переходите с HTTP на HTTP/2 или меняете S3-хранилище на локальный диск), вам придётся менять и верхний уровень.

DIP предлагает **инвертировать** эти стрелки: пусть нижний уровень **зависит от интерфейса**, который принадлежит верхнему уровню, а не наоборот.

> Принцип DIP применим не только в ООП. Он одинаково хорошо работает в процедурном, функциональном, реактивном и мультипарадигменном программировании.

---

## До и После

### Ситуация "до": жёсткая связь

Рассмотрим простой файловый сервер на Node.js. Есть три модуля:

- `logger.js` — логирует запросы через `console`
- `storage.js` — читает файлы с диска через `ReadableStream`
- `server.js` — HTTP-сервер, который использует `logger` и `storage`

Каждый модуль напрямую импортирует конкретные реализации друг друга. Зависимости выглядят так:

```
server.js  →  logger.js  →  console (Node.js)
           →  storage.js →  ReadableStream (Node.js)
```

Проблема: `server.js` намертво привязан к конкретному логеру и конкретному хранилищу. Хотите писать логи в файл? Хотите отдавать файлы из S3? Придётся менять код сервера.

### Ситуация "после": инверсия через интерфейсы

После применения DIP мы вводим **интерфейсы** (контракты). Каждый интерфейс принадлежит тому слою, который его **использует**, а не тому, который его реализует.

```
server.js  →  ILogger    ←  logger.js
           →  IStorage   ←  storage.js
           →  IStatic    ←  httpServer.js / http2Server.js
```

Теперь `server.js` зависит только от абстракций. Конкретные реализации — `logger.js`, `storage.js` — сами зависят от этих контрактов. Стрелки инвертированы.

---

## Реализация

### Пример без DIP (нарушение принципа)

```javascript
// logger.js — конкретная реализация через console
class ConsoleLogger {
  log(message) {
    console.log(`[LOG] ${message}`);
  }
}

module.exports = { ConsoleLogger };
```

```javascript
// storage.js — конкретная реализация через файловую систему
const fs = require('fs');

class FileStorage {
  getFile(path) {
    // Возвращает ReadableStream напрямую
    return fs.createReadStream(path);
  }
}

module.exports = { FileStorage };
```

```javascript
// server.js — жёсткая зависимость от конкретных классов
const http = require('http');
const { ConsoleLogger } = require('./logger');   // импорт конкретного класса
const { FileStorage }   = require('./storage');  // импорт конкретного класса

class StaticServer {
  constructor() {
    // Верхний уровень сам создаёт детали реализации — нарушение DIP
    this.logger  = new ConsoleLogger();
    this.storage = new FileStorage();
  }

  serveStatic(req, res) {
    this.logger.log(`Request: ${req.url}`);
    const stream = this.storage.getFile(`.${req.url}`);
    stream.pipe(res);
  }
}

const server = new StaticServer();
http.createServer((req, res) => server.serveStatic(req, res)).listen(3000);
```

**Проблема:** `StaticServer` знает про `ConsoleLogger` и `FileStorage`. Чтобы заменить логер или хранилище, нужно редактировать класс `StaticServer`.

---

### Пример с DIP (принцип соблюдён)

Шаг 1 — определяем интерфейсы (контракты) на уровне того слоя, который их использует.

```javascript
// interfaces.js — контракты принадлежат слою сервера, не реализациям

/**
 * @typedef {Object} ILogger
 * @property {(message: string) => void} log
 */

/**
 * @typedef {Object} IFile
 * @property {() => NodeJS.ReadableStream} createReadStream
 */

/**
 * @typedef {Object} IStorage
 * @property {(path: string) => IFile} getFile
 */

/**
 * @typedef {Object} IStatic
 * @property {(req: any, res: any) => void} serveStatic
 */
```

Шаг 2 — реализации зависят от контракта, а не наоборот.

```javascript
// logger-console.js — реализует интерфейс ILogger через console
class ConsoleLogger {
  /** @param {string} message */
  log(message) {
    console.log(`[LOG] ${message}`);
  }
}

// logger-file.js — альтернативная реализация ILogger
const fs = require('fs');

class FileLogger {
  constructor(logPath) {
    this.logPath = logPath;
  }

  /** @param {string} message */
  log(message) {
    fs.appendFileSync(this.logPath, `[LOG] ${message}\n`);
  }
}

module.exports = { ConsoleLogger, FileLogger };
```

```javascript
// storage-disk.js — реализует IStorage, читает файлы с диска
const fs = require('fs');

class DiskStorage {
  /** @param {string} filePath @returns {IFile} */
  getFile(filePath) {
    return {
      createReadStream: () => fs.createReadStream(filePath),
    };
  }
}

// storage-s3.js — альтернативная реализация IStorage (псевдокод)
class S3Storage {
  constructor(bucket) {
    this.bucket = bucket;
  }

  /** @param {string} key @returns {IFile} */
  getFile(key) {
    return {
      // В реальном коде здесь был бы вызов AWS SDK
      createReadStream: () => this._downloadFromS3(key),
    };
  }

  _downloadFromS3(key) {
    // ... логика получения потока из S3
  }
}

module.exports = { DiskStorage, S3Storage };
```

```javascript
// server.js — зависит только от интерфейсов ILogger и IStorage
class StaticServer {
  /**
   * Зависимости передаются снаружи (Dependency Injection).
   * StaticServer ничего не знает о конкретных классах.
   *
   * @param {ILogger}  logger
   * @param {IStorage} storage
   */
  constructor(logger, storage) {
    this.logger  = logger;
    this.storage = storage;
  }

  /** @param {IStatic} transport */
  serveStatic(req, res) {
    this.logger.log(`Request: ${req.url}`);
    const file = this.storage.getFile(`.${req.url}`);
    file.createReadStream().pipe(res);
  }
}

module.exports = { StaticServer };
```

```javascript
// index.js — точка входа: здесь выбираем конкретные реализации
const http = require('http');
const { StaticServer } = require('./server');
const { ConsoleLogger } = require('./logger-console');
const { DiskStorage }   = require('./storage-disk');

// Можно легко заменить ConsoleLogger на FileLogger,
// а DiskStorage на S3Storage — не трогая server.js
const logger  = new ConsoleLogger();
const storage = new DiskStorage();
const app     = new StaticServer(logger, storage);

http.createServer((req, res) => app.serveStatic(req, res)).listen(3000, () => {
  logger.log('Server started on port 3000');
});
```

### Фабрика логеров (стратегия)

Принцип DIP хорошо сочетается с паттерном **Стратегия** и фабриками:

```javascript
// logger-factory.js — фабрика создаёт нужную реализацию ILogger
const { ConsoleLogger } = require('./logger-console');
const { FileLogger }    = require('./logger-console');

/**
 * @param {'console' | 'file'} type
 * @param {Object}             [options]
 * @returns {ILogger}
 */
function createLogger(type, options = {}) {
  switch (type) {
    case 'file':
      return new FileLogger(options.logPath || 'app.log');
    case 'console':
    default:
      return new ConsoleLogger();
  }
}

module.exports = { createLogger };
```

```javascript
// Использование фабрики в index.js
const { createLogger } = require('./logger-factory');

// Переключение реализации — одна строка, server.js не меняется
const logger = createLogger(process.env.LOG_TARGET || 'console');
```

---

## Когда применять

DIP стоит применять, когда:

- **Логика должна быть переносимой** — один и тот же доменный код нужно запустить в браузере, на сервере или в мобильном приложении.
- **Реализации могут меняться** — хранилище данных, способ логирования, сетевой транспорт, провайдер очереди сообщений.
- **Нужна тестируемость** — при передаче зависимостей через конструктор (Dependency Injection) легко подменить реальную реализацию на заглушку (mock/stub) в тестах.
- **Команда растёт** — разные команды могут независимо разрабатывать разные слои, не мешая друг другу, если между ними есть чёткий контракт (интерфейс).

DIP **не нужен**, если:

- Приложение очень маленькое и реализации никогда не будут меняться.
- Введение интерфейсов создаёт больше сложности, чем решает проблем.

---

## Связанные принципы и паттерны

DIP — самый общий из принципов SOLID. Он является основой для:

| Концепция | Суть |
|---|---|
| **IoC** (Inversion of Control) | Общий принцип: управление созданием зависимостей передаётся внешнему коду |
| **DI** (Dependency Injection) | Конкретный механизм IoC: зависимости передаются в конструктор или метод |
| **Service Locator** | Альтернативный механизм IoC: зависимости запрашиваются из реестра |
| **Стратегия** | Паттерн GoF: алгоритм (реализация) заменяется без изменения контекста |
| **Фабричный метод** | Паттерн GoF: создание объектов делегируется фабрике |

---

## Итог

Принцип инверсии зависимостей переворачивает привычное направление зависимостей в коде:

- **Без DIP:** высокоуровневый код импортирует низкоуровневые детали напрямую. Смена деталей ломает верхний уровень.
- **С DIP:** и верхний, и нижний уровни зависят от абстракции (интерфейса). Интерфейс принадлежит верхнему уровню. Конкретная реализация может быть заменена без изменения бизнес-логики.

Правило большого пальца: **верхний уровень владеет интерфейсом, нижний — реализует его.**

Именно поэтому DIP является фундаментом чистой и луковичной архитектур, а также основой для Dependency Injection и Inversion of Control — тем, что мы рассмотрим в следующем материале.
