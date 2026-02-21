# Node.js: Слои и Слабое Зацепление на примере конфигурации и транспорта

## Обзор

Эта лекция посвящена архитектуре Node.js приложений с акцентом на разделение по слоям (Layers) и принцип слабого зацепления (Low Coupling). Мы рассмотрим практические примеры рефакторинга, включая вынос конфигурации из приложения, абстрагирование транспортных слоев (HTTP и WebSocket) и применение принципа инъекции зависимостей (Dependency Injection) через различные механизмы.

**Основные темы:**
- Вынос конфигурации в отдельный модуль
- Абстрагирование транспортов
- Слабое зацепление модулей
- Инъекция зависимостей через замыкания
- Отказ от кастомного лоадера в пользу CommonJS

---

## Часть 1: Вынос конфигурации из приложения

### Проблема: Конфигурация распределена по коду

Раньше конфигурация была встроена прямо в код приложения. Это создавало проблемы:
- Сильное зацепление (tight coupling) между модулями и конфигурацией
- Сложность тестирования
- Невозможность легко изменять параметры
- Модули импортируют конфигурацию повсеместно

### Решение: Централизованная конфигурация

Все настройки выносятся в один файл конфигурации (например, `config.js`).

#### Пример структуры конфига

```javascript
// config.js
'use strict';

module.exports = {
  // Порт для отдачи статики
  static: {
    port: 8000
  },

  // Порт для API
  api: {
    port: 8001,
    transport: 'http' // или 'ws' для WebSocket
  },

  // Настройки Sandbox для изоляции прикладного кода
  sandbox: {
    timeout: 5000,
    displayErrors: false
  },

  // Подключение к базе данных
  db: {
    host: 'localhost',
    port: 5432,
    database: 'application',
    user: 'user',
    password: 'password'
  }
};
```

### Принцип единственной точки входа

**Важно:** Конфиг должен импортироваться **только в одном месте** — в точке входа приложения (`main.js`).

```javascript
// main.js
'use strict';

const config = require('./config.js');

// Дальше конфиг передается через зависимости,
// а не импортируется повсеместно
```

**Почему это важно:**
- Централизованное управление конфигурацией
- Легко отследить, где и как используются настройки
- Упрощается тестирование (можно подменить конфиг)
- Снижается зацепление между модулями

---

## Часть 2: Инъекция зависимостей через замыкания

### Концепция

Вместо импорта конфигурации в каждом модуле, мы **инжектируем** нужные части конфигурации через замыкания (closures).

### Пример 1: Инъекция настроек в Sandbox loader

**До рефакторинга:**
```javascript
// loader.js
'use strict';

const config = require('./config.js'); // Плохо: импорт конфига

module.exports = (filename) => {
  const timeout = config.sandbox.timeout;
  // ... код
};
```

**После рефакторинга:**
```javascript
// loader.js
'use strict';

// Экспортируем функцию, которая принимает options
module.exports = (options) => {
  // Возвращаем функцию-загрузчик
  return (filename) => {
    const { timeout } = options; // options попадает в замыкание

    const sandbox = {
      console,
      setTimeout,
      // ... остальные API
    };

    const context = vm.createContext(sandbox);
    // ... код запуска в sandbox
  };
};
```

**Использование в main.js:**
```javascript
// main.js
const config = require('./config.js');
const createLoader = require('./loader.js');

// Передаем только нужную часть конфига
const loader = createLoader(config.sandbox);

// Теперь loader содержит настройки в замыкании
```

**Преимущества:**
- Модуль `loader.js` не знает про существование файла `config.js`
- Настройки попадают в замыкание и доступны внутри функции
- Модуль получает **только то, что ему нужно**, а не весь конфиг

### Пример 2: Инъекция настроек базы данных

```javascript
// db.js
'use strict';

const pg = require('pg');

// Экспортируем функцию высшего порядка
module.exports = (options) => {
  // Создаем пул подключений с переданными настройками
  const pool = new pg.Pool(options);

  // Возвращаем функцию для создания CRUD операций
  const crud = (pool) => {
    return (tableName) => {
      return {
        async create(record) {
          const keys = Object.keys(record).join(', ');
          const values = Object.values(record);
          const params = values.map((_, i) => `$${i + 1}`).join(', ');
          const sql = `INSERT INTO ${tableName} (${keys}) VALUES (${params})`;
          const result = await pool.query(sql, values);
          return result.rows[0];
        },

        async read(id) {
          const sql = `SELECT * FROM ${tableName} WHERE id = $1`;
          const result = await pool.query(sql, [id]);
          return result.rows[0];
        },

        async update(id, record) {
          const keys = Object.keys(record);
          const values = Object.values(record);
          const set = keys.map((key, i) => `${key} = $${i + 2}`).join(', ');
          const sql = `UPDATE ${tableName} SET ${set} WHERE id = $1`;
          await pool.query(sql, [id, ...values]);
        },

        async delete(id) {
          const sql = `DELETE FROM ${tableName} WHERE id = $1`;
          await pool.query(sql, [id]);
        }
      };
    };
  };

  // Вызываем crud с пулом и возвращаем результат
  return crud(pool);
};
```

**Использование:**
```javascript
// main.js
const config = require('./config.js');
const db = require('./db.js');

// Инжектируем настройки БД
const createCrud = db(config.db);

// Создаем CRUD для конкретной таблицы
const usersCrud = createCrud('users');

// Используем
await usersCrud.create({ name: 'John', email: 'john@example.com' });
```

**Цепочка вызовов:**
1. `db(config.db)` — передаем настройки, создаем пул, возвращаем функцию `crud`
2. `createCrud('users')` — передаем имя таблицы, возвращаем объект с методами CRUD
3. Пул и имя таблицы находятся в замыкании

---

## Часть 3: Инъекция логгера

### Проблема использования глобального console

Во многих модулях используется глобальный объект `console`, который:
- Невозможно переопределить или подменить
- Не позволяет централизованно управлять логированием
- Усложняет тестирование

### Решение: Инъекция кастомного логгера

```javascript
// logger.js
'use strict';

const pino = require('pino');

const logger = pino({
  level: 'info',
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true
    }
  }
});

// Создаем обертку с интерфейсом console
module.exports = {
  log: (...args) => logger.info(...args),
  error: (...args) => logger.error(...args),
  warn: (...args) => logger.warn(...args),
  debug: (...args) => logger.debug(...args),
};
```

**Использование в транспорте:**
```javascript
// transport/ws.js
'use strict';

const { WebSocket } = require('ws');

module.exports = (routing, port, console) => {
  const ws = new WebSocket.Server({ port });

  ws.on('connection', (connection) => {
    console.log('WebSocket client connected'); // Используем инжектированный логгер

    connection.on('message', async (message) => {
      const { method, args } = JSON.parse(message);
      console.log(`Call: ${method}`, args);

      const handler = routing[method];
      if (!handler) {
        console.error(`Method not found: ${method}`);
        return;
      }

      const result = await handler(...args);
      connection.send(JSON.stringify(result));
    });
  });

  console.log(`WebSocket server on port ${port}`);
};
```

**В main.js:**
```javascript
// main.js
const logger = require('./logger.js');
const transport = require('./transport/ws.js');

// Инжектируем логгер в транспорт
transport(routing, config.api.port, logger);
```

**Альтернатива — перекрытие имени:**
```javascript
// static.js
module.exports = (path, port, console) => {
  // console теперь указывает на переданный логгер,
  // а не на глобальный объект

  const server = http.createServer((req, res) => {
    console.log(`${req.method} ${req.url}`);
    // ...
  });

  server.listen(port);
  console.log(`Static server on port ${port}`);
};
```

---

## Часть 4: Абстрагирование транспортов

### Цель

Сделать приложение независимым от конкретного транспорта (HTTP или WebSocket). Переключение транспорта должно происходить изменением **одного параметра в конфигурации**.

### Структура папки transport

```
transport/
├── http.js    # HTTP транспорт
└── ws.js      # WebSocket транспорт
```

### Контракт транспорта

Оба транспорта должны экспортировать функцию с одинаковой сигнатурой:

```javascript
module.exports = (routing, port, console) => {
  // Инициализация транспорта
  // ...
};
```

**Параметры:**
- `routing` — объект с методами API (маршрутизация)
- `port` — порт для запуска сервера
- `console` — логгер для вывода информации

### Реализация HTTP транспорта

```javascript
// transport/http.js
'use strict';

const http = require('http');

module.exports = (routing, port, console) => {
  const HEADERS = {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type',
  };

  const server = http.createServer(async (req, res) => {
    // Обработка CORS preflight
    if (req.method === 'OPTIONS') {
      res.writeHead(200, HEADERS);
      res.end();
      return;
    }

    // Парсинг URL: /api/interfaceName/methodName
    const url = req.url.split('/');
    const interfaceName = url[2];
    const methodName = url[3];

    console.log(`HTTP Call: ${interfaceName}.${methodName}`);

    // Получаем тело запроса
    let body = '';
    req.on('data', chunk => {
      body += chunk.toString();
    });

    req.on('end', async () => {
      const args = JSON.parse(body);
      const method = routing[methodName];

      if (!method) {
        res.writeHead(404, HEADERS);
        res.end(JSON.stringify({ error: 'Method not found' }));
        return;
      }

      try {
        const result = await method(...args);
        res.writeHead(200, HEADERS);
        res.end(JSON.stringify(result));
      } catch (error) {
        console.error(error);
        res.writeHead(500, HEADERS);
        res.end(JSON.stringify({ error: error.message }));
      }
    });
  });

  server.listen(port);
  console.log(`HTTP API server on port ${port}`);
};
```

### Реализация WebSocket транспорта

```javascript
// transport/ws.js
'use strict';

const { WebSocketServer } = require('ws');

module.exports = (routing, port, console) => {
  const wss = new WebSocketServer({ port });

  wss.on('connection', (connection) => {
    console.log('WebSocket client connected');

    connection.on('message', async (message) => {
      const packet = JSON.parse(message);
      const { method, args } = packet;

      console.log(`WS Call: ${method}`, args);

      const handler = routing[method];

      if (!handler) {
        console.error(`Method not found: ${method}`);
        connection.send(JSON.stringify({ error: 'Method not found' }));
        return;
      }

      try {
        const result = await handler(...args);
        connection.send(JSON.stringify({ result }));
      } catch (error) {
        console.error(error);
        connection.send(JSON.stringify({ error: error.message }));
      }
    });

    connection.on('close', () => {
      console.log('WebSocket client disconnected');
    });
  });

  console.log(`WebSocket API server on port ${port}`);
};
```

### Динамическая загрузка транспорта в main.js

```javascript
// main.js
'use strict';

const config = require('./config.js');
const logger = require('./logger.js');

// Динамически загружаем транспорт из конфига
const transportName = config.api.transport; // 'http' или 'ws'
const transport = require(`./transport/${transportName}.js`);

// Загружаем API (routing)
const routing = {};
const apiPath = './api';
const files = fs.readdirSync(apiPath);

for (const file of files) {
  const modulePath = path.join(apiPath, file);
  const methods = require(modulePath);
  Object.assign(routing, methods);
}

// Запускаем транспорт с инжекцией зависимостей
transport(routing, config.api.port, logger);
```

**Переключение транспорта:**
```javascript
// config.js
module.exports = {
  api: {
    port: 8001,
    transport: 'http' // Меняем на 'ws' для WebSocket
  }
};
```

---

## Часть 5: Абстрагирование транспорта на клиенте

### Цель

Клиентское приложение должно уметь работать с разными транспортами без изменения бизнес-логики.

### Scaffold — динамическое создание API

**Идея:** На клиенте мы создаем прокси-объект, который имитирует серверное API. Вызов методов автоматически превращается в сетевые запросы.

```javascript
// client.js
'use strict';

const scaffolds = {
  http: (url) => (structure) => {
    const api = {};
    for (const interfaceName in structure) {
      api[interfaceName] = {};
      const methods = structure[interfaceName];

      for (const methodName of methods) {
        api[interfaceName][methodName] = (...args) => {
          return fetch(`${url}/${interfaceName}/${methodName}`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(args)
          }).then(res => res.json());
        };
      }
    }
    return api;
  },

  ws: (url) => (structure) => {
    const socket = new WebSocket(url);
    const api = {};
    const calls = new Map();
    let callId = 0;

    socket.addEventListener('message', (event) => {
      const packet = JSON.parse(event.data);
      const { id, result, error } = packet;
      const { resolve, reject } = calls.get(id);

      if (error) reject(new Error(error));
      else resolve(result);

      calls.delete(id);
    });

    for (const interfaceName in structure) {
      api[interfaceName] = {};
      const methods = structure[interfaceName];

      for (const methodName of methods) {
        api[interfaceName][methodName] = (...args) => {
          return new Promise((resolve, reject) => {
            const id = callId++;
            calls.set(id, { resolve, reject });

            const packet = { id, method: methodName, args };
            socket.send(JSON.stringify(packet));
          });
        };
      }
    }

    return new Promise((resolve) => {
      socket.addEventListener('open', () => resolve(api));
    });
  }
};

// Определение транспорта по URL
const scaffold = (url, structure) => {
  const protocol = url.startsWith('ws://') || url.startsWith('wss://') ? 'ws' : 'http';
  const transport = scaffolds[protocol];
  return transport(url)(structure);
};

// Использование
const api = await scaffold('http://localhost:8001', {
  test: ['say']
});

// Вызов метода — прозрачно, без знания о транспорте
const result = await api.test.say('Hello');
console.log(result); // { status: 'OK' }
```

**Переключение транспорта:**
```javascript
// Для HTTP
const api = await scaffold('http://localhost:8001', structure);

// Для WebSocket (меняем только URL)
const api = await scaffold('ws://localhost:8001', structure);
```

---

## Часть 6: Слои и Low Coupling

### Модель ISO/OSI как аналогия

В сетевой модели ISO/OSI каждый слой взаимодействует **только с соседними слоями**. Например:
- **Application Layer** на клиенте общается с **Application Layer** на сервере
- Но физически данные проходят через все нижележащие слои (Transport, Network, Data Link, Physical)

**Преимущества:**
- Каждый слой решает свою задачу
- Слои не знают о деталях реализации других слоев
- Можно заменить реализацию слоя без изменения других

### Применение в Node.js приложении

```
┌─────────────────────────────────────┐
│   Business Logic (API methods)      │  ← Прикладной уровень
├─────────────────────────────────────┤
│   Routing (метод → обработчик)      │  ← Уровень маршрутизации
├─────────────────────────────────────┤
│   Transport (HTTP / WebSocket)      │  ← Транспортный уровень
├─────────────────────────────────────┤
│   Infrastructure (Config, Logger)   │  ← Инфраструктурный уровень
└─────────────────────────────────────┘
```

**Принцип Low Coupling:**
- Бизнес-логика **не знает** про транспорт
- Транспорт **не знает** про конфигурацию (получает только порт)
- Маршрутизация **не знает** про логирование (получает логгер как зависимость)

### Пример: API метод не зависит от транспорта

```javascript
// api/test.js
'use strict';

module.exports = {
  async say(message) {
    console.log('Received message:', message);
    return { status: 'OK', echo: message };
  }
};
```

**Этот код работает одинаково при любом транспорте!**

---

## Часть 7: Отказ от кастомного лоадера

### Зачем нужен был лоадер?

Кастомный лоадер позволял:
- Автоматически загружать модули из папки
- Инжектировать зависимости в контекст модуля через VM sandbox
- Изолировать прикладной код

### Переход на CommonJS

В примере **C** лоадер удален, используется стандартный `require`.

**До (с лоадером):**
```javascript
// main.js
const loader = require('./loader.js');

const apiPath = './api';
const files = fs.readdirSync(apiPath);

const api = {};
for (const file of files) {
  const modulePath = path.join(apiPath, file);
  const module = loader(modulePath);
  Object.assign(api, module);
}
```

**После (без лоадера):**
```javascript
// main.js
const apiPath = './api';
const files = fs.readdirSync(apiPath);

const routing = {};
for (const file of files) {
  const modulePath = path.join(apiPath, file);
  const methods = require(modulePath);
  Object.assign(routing, methods);
}
```

**Последствия:**
- Код проще и понятнее
- Минус один уровень абстракции
- Модули выполняются в общем контексте Node.js
- Но теряется изоляция

---

## Часть 8: MIME-типы для статического сервера

При отдаче статических файлов нужно указывать правильный `Content-Type`.

### Словарь MIME-типов

```javascript
// static.js
const MIME_TYPES = {
  html: 'text/html; charset=UTF-8',
  js: 'application/javascript; charset=UTF-8',
  css: 'text/css',
  png: 'image/png',
  ico: 'image/x-icon',
  svg: 'image/svg+xml',
  json: 'application/json',
};
```

### Использование

```javascript
// static.js
'use strict';

const http = require('http');
const fs = require('fs');
const path = require('path');

const MIME_TYPES = {
  html: 'text/html; charset=UTF-8',
  js: 'application/javascript; charset=UTF-8',
  css: 'text/css',
  png: 'image/png',
  ico: 'image/x-icon',
  svg: 'image/svg+xml',
};

const HEADERS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type',
};

module.exports = (staticPath, port, console) => {
  const server = http.createServer((req, res) => {
    const filePath = path.join(staticPath, req.url === '/' ? 'index.html' : req.url);
    const ext = path.extname(filePath).substring(1);
    const mimeType = MIME_TYPES[ext] || 'application/octet-stream';

    fs.readFile(filePath, (err, data) => {
      if (err) {
        res.writeHead(404, { 'Content-Type': 'text/plain' });
        res.end('Not Found');
        return;
      }

      res.writeHead(200, { ...HEADERS, 'Content-Type': mimeType });
      res.end(data);
    });
  });

  server.listen(port);
  console.log(`Static server on port ${port}`);
};
```

---

## Часть 9: Способы инъекции зависимостей

В лекции были рассмотрены **несколько способов** внедрения зависимостей (Dependency Injection).

### Способ 1: Через замыкание (Closure Injection)

```javascript
// module.js
module.exports = (dependency) => {
  return () => {
    // dependency доступна в замыкании
    dependency.doSomething();
  };
};

// main.js
const createModule = require('./module.js');
const module = createModule(someDependency);
```

**Плюсы:**
- Простота
- Зависимость попадает в замыкание и всегда доступна
- Нет глобального состояния

### Способ 2: Через require (Singleton)

```javascript
// logger.js
const pino = require('pino');
const logger = pino();
module.exports = logger;

// module1.js
const logger = require('./logger.js'); // Получает singleton
logger.info('Module 1');

// module2.js
const logger = require('./logger.js'); // Тот же экземпляр
logger.info('Module 2');
```

**Плюсы:**
- Автоматическое создание синглтона через кэш `require`
- Не нужно передавать зависимость явно

**Минусы:**
- Сильное зацепление
- Сложно тестировать (нужно чистить кэш `require`)

### Способ 3: Через конструктор класса (Constructor Injection)

```javascript
// Database.js
class Database {
  constructor(config, logger) {
    this.config = config;
    this.logger = logger;
    this.pool = new Pool(config);
  }

  async query(sql, params) {
    this.logger.info('Query:', sql);
    return this.pool.query(sql, params);
  }
}

module.exports = Database;

// main.js
const Database = require('./Database.js');
const db = new Database(config.db, logger);
```

**Плюсы:**
- Явная передача зависимостей
- Легко тестировать (можно передать моки)
- ООП подход

### Способ 4: Через Sandbox (Context Injection)

```javascript
// loader.js
const vm = require('vm');

module.exports = (filename, context) => {
  const code = fs.readFileSync(filename, 'utf8');
  const sandbox = { ...context, console, setTimeout };
  vm.runInContext(code, vm.createContext(sandbox));
};

// main.js
const context = {
  db: database,
  logger: logger,
  config: config
};

loader('./app.js', context);
```

**Плюсы:**
- Полная изоляция
- Контроль над доступным API

**Минусы:**
- Сложность
- Производительность

---

## Резюме

### Что было сделано в лекции:

1. **Вынесли конфигурацию** в отдельный файл
2. **Создали единую точку входа** для импорта конфига (main.js)
3. **Применили инъекцию зависимостей через замыкания** для передачи:
   - Настроек Sandbox в loader
   - Настроек БД в db модуль
   - Логгера в транспорты
   - Порта в HTTP/WebSocket серверы
4. **Абстрагировали транспорты:**
   - Создали папку `transport/` с модулями `http.js` и `ws.js`
   - Унифицировали контракт транспорта: `(routing, port, console) => {}`
   - Реализовали динамическую загрузку транспорта из конфига
5. **Реализовали scaffold на клиенте** для автоматического создания API-прокси
6. **Показали переход от кастомного лоадера к CommonJS**
7. **Применили принцип Low Coupling:**
   - Бизнес-логика не знает про транспорт
   - Транспорт не знает про конфигурацию
   - Модули получают только необходимые зависимости

### Домашнее задание

**Задание 5:** Сделать подключаемый фреймворк (по аналогии с транспортом)

**Задание 7:** Сделать подключаемый логгер (Pino или другой):
- Создать обертку, реализующую интерфейс `console`
- Инжектировать во все модули через замыкание
- Обеспечить возможность переключения между нативным `console` и кастомным логгером

### Ключевые принципы

**Low Coupling (Слабое зацепление):**
- Модули зависят от абстракций, а не от конкретных реализаций
- Каждый модуль получает только то, что ему нужно
- Изменения в одном модуле не затрагивают другие

**Layered Architecture (Слоистая архитектура):**
- Четкое разделение ответственности между слоями
- Слои взаимодействуют только через определенные интерфейсы
- Возможность заменить реализацию слоя без изменения других

**Dependency Injection (Инъекция зависимостей):**
- Зависимости передаются извне, а не создаются внутри модуля
- Упрощает тестирование и переиспользование кода
- Можно реализовать через замыкания, конструкторы, или контекст

---

## Дополнительные примеры

### Пример: Полный цикл запроса через слои

**Клиент (Browser):**
```javascript
const api = await scaffold('http://localhost:8001', {
  test: ['say']
});

const result = await api.test.say('Hello');
console.log(result); // { status: 'OK', echo: 'Hello' }
```

**Сервер — Транспортный слой (HTTP):**
```javascript
// transport/http.js
// Получает запрос POST /api/test/say
// Парсит тело: ["Hello"]
// Вызывает routing.say("Hello")
// Отправляет ответ: {"status": "OK", "echo": "Hello"}
```

**Сервер — Бизнес-логика:**
```javascript
// api/test.js
module.exports = {
  async say(message) {
    return { status: 'OK', echo: message };
  }
};
```

**Поток данных:**
```
Client → HTTP Request → Transport Layer → Routing → Business Logic
                                                          ↓
Client ← HTTP Response ← Transport Layer ← Return Value ←
```

### Пример: Переключение с HTTP на WebSocket

**Изменение на сервере:**
```javascript
// config.js
module.exports = {
  api: {
    transport: 'ws' // Было: 'http'
  }
};
```

**Изменение на клиенте:**
```javascript
// Было:
const api = await scaffold('http://localhost:8001', structure);

// Стало:
const api = await scaffold('ws://localhost:8001', structure);
```

**Бизнес-логика не меняется!** Вызовы остаются прежними:
```javascript
await api.test.say('Hello'); // Работает с любым транспортом
```

---

## Заключение

Применение принципов **Layers** и **Low Coupling** позволяет создавать гибкие, легко тестируемые и поддерживаемые приложения. Инъекция зависимостей через замыкания — элегантный способ снизить зацепление в JavaScript/Node.js приложениях, сохраняя при этом простоту кода.

В следующих лекциях мы:
- Перейдем на ES Modules
- Реализуем подключаемые фреймворки
- Добавим работу с базами данных
- Спроектируем API для мессенджера
- Разработаем полноценное приложение на базе созданной инфраструктуры
