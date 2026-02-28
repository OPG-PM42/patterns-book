# Принцип инверсии зависимостей (DIP) — Разбор практического примера

> **SOLID / Dependency Inversion Principle**
> Разбор реального кода на JavaScript: до рефакторинга и после.

---

## Введение

Принцип инверсии зависимостей (Dependency Inversion Principle, DIP) — один из самых
сложных, часто недопонимаемых и при этом самых полезных принципов SOLID. Именно он
открывает путь к построению слоистой архитектуры приложений.

Формальная формулировка:

1. **Модули верхних уровней не должны зависеть от модулей нижних уровней.** И те и
   другие должны зависеть от абстракций (интерфейсов).
2. **Абстракции не должны зависеть от деталей.** Детали (конкретные реализации)
   должны зависеть от абстракций.

Ключевая идея: интерфейс является **собственностью того, кто его вызывает**, а не
того, кто его реализует. Это разворачивает стрелки зависимостей снизу вверх, делая
код подменяемым.

В этой лекции мы разберём конкретный пример: файл-сервер на Node.js, который
логирует работу и читает файлы из хранилища.

---

## Исходный код (до рефакторинга)

До применения DIP все модули жёстко импортируют друг друга по имени файла и
используют конкретные классы напрямую. Зависимости направлены только вниз — от
высокоуровневого сервера к низкоуровневым деталям.

### Структура проекта (до)

```
src/
  logger.js      — создаёт console, пишет в stdout
  storage.js     — читает файлы с диска через fs
  server.js      — импортирует logger и storage напрямую
  main.js        — точка входа
```

### logger.js (до)

```javascript
// logger.js — конкретная реализация, прибита к stdout
'use strict';

// Используем глобальный объект console напрямую.
// Нет никакого контракта — просто класс платформы.
const logger = console;

module.exports = { logger };
```

### storage.js (до)

```javascript
// storage.js — два класса жёстко связаны между собой
'use strict';

const fs = require('node:fs');
const path = require('node:path');

// PrepFile — подготавливает метаданные файла
class PrepFile {
  constructor(filePath) {
    this.filePath = filePath;
    this.extension = path.extname(filePath);
  }
}

// Storage — читает файлы с диска. Знает про PrepFile напрямую.
class Storage {
  get(filePath) {
    const file = new PrepFile(filePath); // жёсткая связь с PrepFile
    const stream = fs.createReadStream(file.filePath);
    return { extension: file.extension, stream };
  }
}

module.exports = { Storage };
```

### server.js (до)

```javascript
// server.js — прибит гвоздями и к Storage, и к logger
'use strict';

const http = require('node:http');
const { Storage } = require('./storage'); // конкретный класс
const { logger } = require('./logger');   // конкретный объект

// Сервер сам создаёт зависимости внутри себя.
// Подменить Storage или logger без изменения этого файла невозможно.
function serveStatic(req, res) {
  const storage = new Storage(); // зависимость создаётся здесь же
  logger.log(`Request: ${req.url}`);

  const { extension, stream } = storage.get(`.${req.url}`);
  res.setHeader('Content-Type', `text/${extension.slice(1)}`);
  stream.pipe(res);
}

const server = http.createServer(serveStatic);
module.exports = { server };
```

### main.js (до)

```javascript
// main.js — просто запускает сервер
'use strict';

const { server } = require('./server');

server.listen(3000, () => {
  console.log('Server started on port 3000');
});
```

---

## Проблемы

Перечислим конкретные нарушения DIP в коде выше.

| Проблема | Следствие |
|---|---|
| `server.js` создаёт `new Storage()` внутри себя | Невозможно подменить хранилище (файловая система → S3, БД, другой сервер) без изменения `server.js` |
| `server.js` импортирует `logger` напрямую | Невозможно подменить способ логирования (stdout → файл → Pino → агрегатор логов) |
| `storage.js` жёстко зависит от `PrepFile` | Два класса нижнего уровня связаны между собой; сложно тестировать по отдельности |
| Нет никаких интерфейсов/контрактов | Компилятор и среда выполнения не могут проверить соответствие зависимостей |

Главный симптом: **все стрелки зависимостей направлены вниз**. Верхний уровень
(сервер) знает обо всём, что находится ниже.

```
main.js
  └─> server.js
        ├─> storage.js  ──> fs (Node.js platform)
        └─> logger.js   ──> console (Node.js platform)
```

---

## Рефакторинг с DIP

Решение: **ввести интерфейсы-контракты** и **перенести создание зависимостей в
`main.js`**. Модули больше не создают зависимости сами — они получают их снаружи
(dependency injection через замыкание или конструктор).

### Шаг 1. Определяем интерфейсы (контракты)

В JavaScript нет встроенного синтаксиса интерфейсов, поэтому контракты описываются
через JSDoc или TypeScript `.d.ts`. Приведём оба варианта.

```javascript
// interfaces.js — описание контрактов (JSDoc-вариант)
'use strict';

/**
 * @typedef {Object} IFile
 * @property {string} extension  — расширение файла, например '.html'
 * @property {import('stream').Readable} stream — поток с содержимым
 */

/**
 * @typedef {Object} IStorage
 * @property {function(string): Promise<IFile>} get — получить файл по пути
 */

/**
 * @typedef {Object} ILogger
 * @property {function(...any): void} log   — информационное сообщение
 * @property {function(...any): void} error — сообщение об ошибке
 */

/**
 * @typedef {function(IStorage, ILogger): import('http').RequestListener} ICreateServer
 * Фабрика, принимающая зависимости и возвращающая HTTP-обработчик.
 */

module.exports = {};
```

```typescript
// interfaces.d.ts — то же самое в TypeScript
import { Readable } from 'node:stream';
import { RequestListener } from 'node:http';

export interface IFile {
  extension: string;
  stream: Readable;
}

export interface IStorage {
  get(filePath: string): Promise<IFile>;
}

export interface ILogger {
  log(...args: unknown[]): void;
  error(...args: unknown[]): void;
}

// Интерфейс самого сервера — функция, возвращающая обработчик запросов
export type ICreateServer = (storage: IStorage, logger: ILogger) => RequestListener;
```

### Шаг 2. Рефакторинг logger.js — стратегия логирования

Вместо одного жёсткого логгера создаём фабрику `createLogger`, которая возвращает
объект, реализующий `ILogger`. Фабрику можно вызвать с разными стратегиями.

```javascript
// logger.js (после рефакторинга)
'use strict';

const fs = require('node:fs');
const { PassThrough } = require('node:stream');

/**
 * Стратегия 1: логирование только в stdout.
 * Использует встроенный класс Console как контракт платформы.
 *
 * @returns {Console} объект, реализующий интерфейс ILogger
 */
function createStdoutLogger() {
  // Console — класс платформы с высокой обратной совместимостью.
  // Он достаточно стабилен, чтобы использовать его как интерфейс.
  return new console.Console(process.stdout, process.stderr);
}

/**
 * Стратегия 2: логирование в stdout И в файл одновременно.
 * Использует PassThrough-стрим, который разветвляет поток данных.
 *
 * @param {string} logFilePath — путь к файлу для записи логов
 * @returns {Console} объект, реализующий интерфейс ILogger
 */
function createFileLogger(logFilePath) {
  // PassThrough позволяет подписать несколько получателей на один поток.
  const passThrough = new PassThrough();
  const fileStream = fs.createWriteStream(logFilePath, { flags: 'a' });

  // Первый pipe — в файл
  passThrough.pipe(fileStream);
  // Второй pipe — в stdout
  passThrough.pipe(process.stdout);

  // Console принимает любой Writable-стрим, не только process.stdout.
  // Это и есть использование платформенного класса как абстракции.
  return new console.Console(passThrough, passThrough);
}

module.exports = { createStdoutLogger, createFileLogger };
```

### Шаг 3. Рефакторинг storage.js — один класс, чистый интерфейс

Убираем `PrepFile` как отдельный класс. Вместо создания объекта `new PrepFile`
возвращаем простую структуру, описанную интерфейсом `IFile`.

```javascript
// storage.js (после рефакторинга)
'use strict';

const fs = require('node:fs');
const path = require('node:path');

/**
 * FileStorage — конкретная реализация IStorage.
 * Читает файлы с диска через fs.createReadStream.
 *
 * Реализует контракт IStorage: метод get(filePath) -> Promise<IFile>
 */
class FileStorage {
  /**
   * @param {string} filePath — абсолютный или относительный путь к файлу
   * @returns {Promise<IFile>}
   */
  async get(filePath) {
    const extension = path.extname(filePath);
    const stream = fs.createReadStream(filePath);
    // Возвращаем структуру, описанную интерфейсом IFile.
    // FileStorage ничего не знает о том, кто вызовет этот метод.
    return { extension, stream };
  }
}

/**
 * S3Storage — альтернативная реализация IStorage (заглушка для примера).
 * Завтра можно заменить FileStorage на S3Storage без изменения server.js.
 *
 * @implements {IStorage}
 */
class S3Storage {
  constructor(bucketName) {
    this.bucketName = bucketName;
  }

  async get(filePath) {
    // Здесь был бы вызов AWS SDK или minio-client.
    // Для примера — заглушка с пустым стримом.
    const { PassThrough } = require('node:stream');
    const extension = path.extname(filePath);
    const stream = new PassThrough();
    stream.end(`[S3 content of ${filePath} from bucket ${this.bucketName}]`);
    return { extension, stream };
  }
}

module.exports = { FileStorage, S3Storage };
```

### Шаг 4. Рефакторинг server.js — зависимости приходят снаружи

Сервер больше не создаёт зависимости. Они приходят через параметры фабричной
функции (инъекция через замыкание). Это эквивалентно конструктору класса.

```javascript
// server.js (после рефакторинга)
'use strict';

const http = require('node:http');

/**
 * createServer — фабрика HTTP-сервера.
 * Реализует контракт ICreateServer.
 *
 * @param {IStorage} storage — любая реализация IStorage
 * @param {ILogger}  logger  — любая реализация ILogger
 * @returns {import('http').Server}
 *
 * server.js НЕ импортирует ни Storage, ни logger напрямую.
 * Он работает только с интерфейсами — не знает о конкретных реализациях.
 */
function createServer(storage, logger) {
  // Зависимости хранятся в замыкании — аналог приватных полей класса.
  async function serveStatic(req, res) {
    try {
      logger.log(`[${new Date().toISOString()}] Request: ${req.url}`);

      const filePath = `.${req.url}`;
      const file = await storage.get(filePath); // используем IStorage

      const mimeType = getMimeType(file.extension);
      res.setHeader('Content-Type', mimeType);
      file.stream.pipe(res);
    } catch (err) {
      logger.error(`Error serving ${req.url}:`, err.message);
      res.statusCode = 404;
      res.end('Not found');
    }
  }

  return http.createServer(serveStatic);
}

function getMimeType(extension) {
  const map = {
    '.html': 'text/html',
    '.css':  'text/css',
    '.js':   'application/javascript',
    '.json': 'application/json',
    '.txt':  'text/plain',
  };
  return map[extension] ?? 'application/octet-stream';
}

module.exports = { createServer };
```

### Шаг 5. main.js — единственная точка сборки зависимостей

Теперь `main.js` — это **единственное место**, где мы знаем о конкретных
реализациях. Все зависимости создаются здесь и внедряются в модули.

```javascript
// main.js (после рефакторинга)
'use strict';

const { createServer }       = require('./server');
const { FileStorage }        = require('./storage');
const { createFileLogger }   = require('./logger');

// --- Стратегия 1: файловое хранилище ---
const storage = new FileStorage();

// --- Стратегия 2: логирование в файл и stdout ---
const logger = createFileLogger('./app.log');

// --- Внедрение зависимостей ---
// Сервер получает обе зависимости снаружи. Он не создаёт их сам.
// Завтра мы заменим FileStorage на S3Storage — server.js не изменится.
const server = createServer(storage, logger);

server.listen(3000, () => {
  logger.log('Server started on port 3000');
});
```

---

## Итоговый код

### Полная структура проекта (после)

```
src/
  interfaces.d.ts  — контракты: IFile, IStorage, ILogger, ICreateServer
  logger.js        — фабрики: createStdoutLogger, createFileLogger
  storage.js       — реализации: FileStorage, S3Storage
  server.js        — фабрика: createServer(storage, logger)
  main.js          — точка сборки: создаёт реализации, внедряет зависимости
```

### Как выглядят зависимости теперь

**До рефакторинга** — все стрелки вниз:

```
main.js
  └──> server.js  (знает о storage.js и logger.js)
         ├──> storage.js  ──> fs
         └──> logger.js   ──> console
```

**После рефакторинга** — реализации указывают вверх на интерфейсы:

```
main.js  (единственный, кто знает обо всех реализациях)
  ├── создаёт FileStorage  ──реализует──> IStorage  <──зависит── server.js
  ├── создаёт FileLogger   ──реализует──> ILogger   <──зависит── server.js
  └── передаёт их в createServer()
```

Стрелки от `FileStorage` и `FileLogger` теперь идут **вверх** к интерфейсам,
которые принадлежат слою сервера. Сервер зависит только от абстракций.

### Демонстрация подмены зависимостей без изменения server.js

```javascript
// main.js — альтернативная конфигурация (server.js не трогаем)
'use strict';

const { createServer }       = require('./server');
const { S3Storage }          = require('./storage');     // другая реализация
const { createStdoutLogger } = require('./logger');       // другая стратегия

// Меняем хранилище с файловой системы на S3:
const storage = new S3Storage('my-bucket');

// Меняем логирование с файла на stdout:
const logger = createStdoutLogger();

// server.js не знает и не узнает о замене — он работает с IStorage и ILogger
const server = createServer(storage, logger);

server.listen(3000, () => {
  logger.log('Server started with S3 storage and stdout logger');
});
```

---

## Итог

| Аспект | До DIP | После DIP |
|---|---|---|
| Зависимости | Жёсткие (import конкретного файла) | Через интерфейсы (контракты) |
| Создание зависимостей | В каждом модуле (`new Storage()`) | Только в `main.js` |
| Подменяемость | Невозможна без изменения кода | Меняем реализацию в `main.js` |
| Тестируемость | Сложно — нужен реальный FS и сервер | Легко — передаём mock-объекты |
| Направление стрелок | Только вниз | Снизу вверх (к интерфейсам) |

**Что даёт DIP:**

- Модуль `server.js` можно протестировать, передав любой объект с методами `get()`
  и `log()` — без реального файлового хранилища и без реального HTTP.
- Хранилище можно заменить с локального диска на S3/MinIO/PostgreSQL/HTTP, не
  трогая `server.js`.
- Логгер можно заменить с `console` на Pino, Winston или агрегатор логов (Elastic,
  Datadog) — также без изменений в `server.js`.

**Важное уточнение:** DIP не сводится только к внедрению зависимостей (Dependency
Injection). DI — это лишь один из механизмов реализации DIP наряду с сервис-локатором
и DI-контейнером. Суть принципа — в **формулировании контрактов** (интерфейсов)
между модулями и в том, что интерфейс принадлежит вызывающей стороне, а не
реализующей.
