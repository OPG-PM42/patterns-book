# Архитектура приложений Node.js: Слои и внедрение зависимостей

## Обзор

Лекция посвящена построению многоуровневой архитектуры приложений на Node.js с использованием внедрения зависимостей (Dependency Injection). Мы рассмотрим эволюцию архитектуры от простого Express-приложения до чистой архитектуры с разделением на слои.

---

## 1. Базовое Express приложение

### Структура проекта

Начальное приложение представляет собой простой REST API для работы с пользователями, использующий:
- **Express** - веб-фреймворк
- **body-parser** - парсинг JSON
- **PostgreSQL** (библиотека `pg`) - база данных

### Настройка базы данных

**Файлы структуры БД:**
- `install.sh` - создание пользователя и базы данных
- `structure.sql` - создание таблиц (CREATE TABLE, ALTER TABLE)
- `data.sql` - заполнение данными (INSERT)

```sql
-- Пример из data.sql
-- Пароли хешируются, но в комментариях указаны оригиналы
-- admin: 123456
-- marcus: marcus (логин и пароль совпадают)
```

### Пример Express реализации

```javascript
const express = require('express');
const bodyParser = require('body-parser');
const { Pool } = require('pg');

const app = express();
app.use(bodyParser.json());

// Подключение к базе данных
const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'example',
  user: 'marcus'
});

// GET /user - получить всех пользователей
app.get('/user', async (req, res) => {
  const sql = 'SELECT * FROM users';
  try {
    const { rows } = await pool.query(sql);
    res.status(200).json(rows);
  } catch (err) {
    throw err;
  }
});

// POST /user - создать пользователя
app.post('/user', async (req, res) => {
  const { login, password } = req.body;
  const passwordHash = await hash(password);
  const sql = 'INSERT INTO users (login, password) VALUES ($1, $2) RETURNING id';
  try {
    const { rows } = await pool.query(sql, [login, passwordHash]);
    res.status(201).json({ id: rows[0].id });
  } catch (err) {
    throw err;
  }
});

// GET /user/:id - получить пользователя по ID
app.get('/user/:id', async (req, res) => {
  const id = parseInt(req.params.id, 10);
  const sql = 'SELECT * FROM users WHERE id = $1';
  const { rows } = await pool.query(sql, [id]);
  res.status(200).json(rows[0]);
});

// PUT /user/:id - обновить пользователя
app.put('/user/:id', async (req, res) => {
  // Аналогично GET, но с UPDATE
});

// DELETE /user/:id - удалить пользователя
app.delete('/user/:id', async (req, res) => {
  // DELETE FROM users WHERE id = $1
});

app.listen(8000);
```

### Хеширование паролей

```javascript
// hash.js
const crypto = require('crypto');

const hash = (password) => {
  const salt = crypto.randomBytes(16).toString('base64');
  return new Promise((resolve, reject) => {
    crypto.scrypt(password, salt, 64, (err, derivedKey) => {
      if (err) reject(err);
      const hash = derivedKey.toString('base64');
      resolve(`${salt}:${hash}`);
    });
  });
};

module.exports = { hash };
```

**Проблемы этого подхода:**
- Код связан с деталями HTTP протокола
- Бизнес-логика смешана с транспортным слоем
- Зависимости: ~3.94 МБ (Express + body-parser + pg)
- Размер кода: 2.6 КБ

---

## 2. Отказ от Express - собственный роутинг

### Создание функции parseArgs

Так как body-parser больше не используется, создаем собственную функцию для парсинга JSON из request body:

```javascript
// args.js
const parseArgs = async (req) => {
  const chunks = [];
  for await (const chunk of req) {
    chunks.push(chunk);
  }
  const buffer = Buffer.concat(chunks);
  const data = buffer.toString();
  return JSON.parse(data);
};
```

**Почему так оптимально:**
- Используем `for await` для асинхронной итерации по потоку
- Собираем буферы в массив, а не конкатенируем строки
- Один раз преобразуем буфер в строку
- Парсим JSON только один раз

### Новая структура роутинга

```javascript
// main.js (второй пример)
const http = require('http');
const pg = require('pg');
const { hash } = require('./hash');
const { parseArgs } = require('./args');

const pool = new pg.Pool({
  host: 'localhost',
  port: 5432,
  database: 'example',
  user: 'marcus'
});

// Роутинг как объект
const routing = {
  user: {
    get: async () => {
      const sql = 'SELECT * FROM users';
      const { rows } = await pool.query(sql);
      return rows;
    },

    post: async (login, password) => {
      const passwordHash = await hash(password);
      const sql = 'INSERT INTO users (login, password) VALUES ($1, $2) RETURNING id';
      const { rows } = await pool.query(sql, [login, passwordHash]);
      return { id: rows[0].id };
    },

    // другие методы...
  }
};
```

**Ключевое изменение:** методы теперь принимают параметры предметной области (login, password), а не request/response объекты.

### Простой HTTP сервер

```javascript
http.createServer(async (req, res) => {
  const { method, url } = req;
  const [, entity, id] = url.split('/');

  // Находим сущность в роутинге
  const routing = { user: { /* ... */ } };
  const handler = routing[entity];
  if (!handler) {
    res.statusCode = 404;
    res.end('Not found');
    return;
  }

  // Находим метод
  const methodName = method.toLowerCase();
  const fn = handler[methodName];
  if (!fn) {
    res.statusCode = 404;
    res.end('Not found');
    return;
  }

  // Собираем аргументы
  const args = [];
  if (id) args.push(parseInt(id, 10));

  // Парсим body если есть
  const body = await parseArgs(req);
  args.push(...Object.values(body));

  // Вызываем обработчик
  const result = await fn(...args);
  res.end(JSON.stringify(result));
}).listen(8000);
```

**Результат:**
- Зависимости: 840 КБ (только pg)
- Размер кода: 2.3 КБ
- Бизнес-логика начинает отделяться от протокола

---

## 3. Паттерн Front Controller

Front Controller - это единая точка входа для всех HTTP-запросов.

### Преимущества паттерна

1. Централизованная обработка запросов
2. Единое место для логирования, аутентификации
3. Упрощенный роутинг
4. Абстракция от деталей протокола

### Реализация (16 строк кода)

```javascript
// Простейший front controller
http.createServer(async (req, res) => {
  const { method, url } = req;
  const [, entityName, id] = url.split('/');

  const entity = routing[entityName];
  if (!entity) return void res.end('Not found');

  const handler = entity[method.toLowerCase()];
  if (!handler) return void res.end('Not found');

  // Парсинг сигнатуры функции
  const signature = handler.toString();
  const startIdx = signature.indexOf('(');
  const endIdx = signature.indexOf(')');
  const params = signature.substring(startIdx + 1, endIdx);

  const args = [];
  if (params.includes('id')) args.push(parseInt(id, 10));

  const body = await parseArgs(req);
  args.push(...Object.values(body));

  const result = await handler(...args);
  res.end(JSON.stringify(result));
}).listen(8000);
```

**Важно:** Код больше не работает напрямую с HTTP - он работает с абстракциями (роутинг, сущности, методы).

---

## 4. Генератор CRUD операций

### Проблема

Для каждой таблицы писать одинаковый CRUD код - избыточно.

### Решение: универсальный генератор

```javascript
// db.js
const pg = require('pg');

const pool = new pg.Pool({
  host: 'localhost',
  port: 5432,
  database: 'example',
  user: 'marcus'
});

const crud = (table) => ({
  read: async (id) => {
    const sql = `SELECT * FROM ${table} WHERE id = $1`;
    const { rows } = await pool.query(sql, [id]);
    return rows[0];
  },

  create: async (record) => {
    const keys = Object.keys(record);
    const values = Object.values(record);
    const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
    const sql = `INSERT INTO ${table} (${keys.join(', ')}) VALUES (${placeholders}) RETURNING id`;
    const { rows } = await pool.query(sql, values);
    return { id: rows[0].id };
  },

  update: async (id, record) => {
    const keys = Object.keys(record);
    const values = Object.values(record);
    const updates = keys.map((key, i) => `${key} = $${i + 2}`).join(', ');
    const sql = `UPDATE ${table} SET ${updates} WHERE id = $1`;
    await pool.query(sql, [id, ...values]);
    return { id };
  },

  delete: async (id) => {
    const sql = `DELETE FROM ${table} WHERE id = $1`;
    await pool.query(sql, [id]);
    return { id };
  }
});

module.exports = { crud };
```

### Использование

```javascript
// main.js
const { crud } = require('./db');

const routing = {
  user: crud('users'),
  country: crud('countries'),
  city: crud('cities')
};
```

**Результат:** Теперь можно создавать CRUD для любой таблицы одной строкой!

---

## 5. Кастомизация CRUD

### Проблема

Для некоторых сущностей нужна специальная обработка (например, хеширование паролей).

### Решение: обертки

```javascript
// user.js
const { crud } = require('./db');
const { hash } = require('./hash');

const base = crud('users');

module.exports = {
  read: async (id) => {
    const user = await base.read(id);
    // Не возвращаем пароль клиенту
    return { id: user.id, login: user.login };
  },

  create: async (login, password) => {
    const passwordHash = await hash(password);
    return base.create({ login, password: passwordHash });
  },

  update: async (id, login, password) => {
    const passwordHash = await hash(password);
    return base.update(id, { login, password: passwordHash });
  },

  delete: base.delete,

  // Дополнительный метод
  find: async (mask) => {
    const sql = 'SELECT id, login FROM users WHERE login LIKE $1';
    const { rows } = await pool.query(sql, [`%${mask}%`]);
    return rows;
  }
};
```

### Роутинг с кастомизацией

```javascript
const routing = {
  user: require('./user'),
  country: crud('countries'),
  city: crud('cities')
};
```

---

## 6. Абстракция транспортного слоя

### Цель

Отделить бизнес-логику от протокола передачи данных так, чтобы можно было легко переключаться между HTTP, WebSocket и другими протоколами.

### Структура

```
/lib
  /http.js      - HTTP транспорт
  /ws.js        - WebSocket транспорт
  /static.js    - Сервер статических файлов
/api
  /user.js      - Бизнес-логика пользователей
  /city.js      - Бизнес-логика городов
  /country.js   - Бизнес-логика стран
/main.js        - Точка входа
```

### HTTP транспорт

```javascript
// lib/http.js
const http = require('http');
const { parseArgs } = require('./args');

module.exports = (routing, port) => {
  http.createServer(async (req, res) => {
    const { method, url } = req;
    const [, entityName, id] = url.split('/');

    const entity = routing[entityName];
    if (!entity) {
      res.statusCode = 404;
      return void res.end('Not found');
    }

    const handler = entity[method.toLowerCase()];
    if (!handler) {
      res.statusCode = 404;
      return void res.end('Not found');
    }

    const args = [];
    if (id) args.push(parseInt(id, 10));

    const body = await parseArgs(req);
    args.push(...Object.values(body));

    const result = await handler(...args);
    res.end(JSON.stringify(result));
  }).listen(port);
};
```

### WebSocket транспорт

```javascript
// lib/ws.js
const WebSocket = require('ws');

module.exports = (routing, port) => {
  const wss = new WebSocket.Server({ port });

  wss.on('connection', (ws) => {
    ws.on('message', async (message) => {
      const { entity: entityName, method, args = [] } = JSON.parse(message);

      const entity = routing[entityName];
      if (!entity) return ws.send(JSON.stringify({ error: 'Not found' }));

      const handler = entity[method];
      if (!handler) return ws.send(JSON.stringify({ error: 'Not found' }));

      try {
        const result = await handler(...args);
        ws.send(JSON.stringify({ result }));
      } catch (error) {
        ws.send(JSON.stringify({ error: error.message }));
      }
    });
  });
};
```

### Главный файл с переключением протокола

```javascript
// main.js
const server = require('./lib/http');  // или require('./lib/ws')

const routing = {
  user: require('./api/user'),
  country: require('./api/country'),
  city: require('./api/city')
};

server(routing, 8000);
```

**Важно:** Чтобы переключиться между протоколами, достаточно изменить одну строку импорта!

---

## 7. Автоматическая загрузка API

### Проблема

При добавлении новых сущностей нужно вручную добавлять их в роутинг.

### Решение: динамическая загрузка из папки

```javascript
// main.js
const path = require('path');
const fsp = require('fs').promises;

const apiPath = path.join(process.cwd(), 'api');

const loadRouting = async () => {
  const routing = {};
  const files = await fsp.readdir(apiPath);

  for (const file of files) {
    if (!file.endsWith('.js')) continue;
    const name = path.basename(file, '.js');
    const modulePath = path.join(apiPath, file);
    routing[name] = require(modulePath);
  }

  return routing;
};

// Использование
(async () => {
  const routing = await loadRouting();
  server(routing, 8000);
})();
```

**Результат:** Теперь достаточно создать файл в папке `/api`, и он автоматически станет доступен через API!

---

## 8. Система логирования

### Требования к логгеру

1. Запись в файл и на консоль одновременно
2. Цветной вывод в консоль
3. Разные типы логов (log, error, debug, system, access)
4. Форматирование многострочных сообщений

### Реализация логгера

```javascript
// lib/logger.js
const fs = require('fs');
const util = require('util');

const COLORS = {
  log: '\x1b[37m',    // white
  error: '\x1b[31m',  // red
  debug: '\x1b[33m',  // yellow
  system: '\x1b[36m', // cyan
  access: '\x1b[32m'  // green
};

const RESET = '\x1b[0m';

class Logger {
  constructor(logPath) {
    this.stream = fs.createWriteStream(logPath, { flags: 'a' });
  }

  write(type, message) {
    const date = new Date().toISOString();
    const color = COLORS[type] || '';

    // Преобразуем в строку
    const text = typeof message === 'string'
      ? message
      : util.inspect(message);

    // Убираем переводы строк для файла
    const fileLine = `${date}\t${type}\t${text.replace(/\n/g, '; ')}\n`;
    this.stream.write(fileLine);

    // В консоль с цветом
    console.log(`${color}${date}\t[${type.toUpperCase()}]\t${text}${RESET}`);
  }

  log(message) { this.write('log', message); }
  error(message) { this.write('error', message); }
  debug(message) { this.write('debug', message); }
  system(message) { this.write('system', message); }
  access(message) { this.write('access', message); }

  // Для совместимости с console
  dir(obj) { this.write('log', obj); }

  close() {
    this.stream.end();
  }
}

module.exports = (logPath) => new Logger(logPath);
```

### Использование логгера

```javascript
// main.js
const logger = require('./lib/logger')('./app.log');

// Логгер доступен глобально через Sandbox (см. ниже)
```

---

## 9. Внедрение зависимостей через Sandbox

### Концепция Sandbox

Sandbox - это изолированный контекст выполнения, в который внедряются зависимости.

### Зачем нужен Sandbox?

1. **Изоляция:** модули не имеют доступа к глобальному scope
2. **Внедрение зависимостей:** можем контролировать, что доступно модулям
3. **Безопасность:** ограничиваем возможности модулей
4. **Тестируемость:** легко подменять зависимости

### Реализация простого загрузчика модулей

```javascript
// lib/loader.js
const fs = require('fs');
const vm = require('vm');

const createContext = (sandbox) => {
  const context = vm.createContext(sandbox);
  return context;
};

const runScript = (code, context) => {
  const script = new vm.Script(code);
  const exports = {};
  context.module = { exports };
  context.exports = exports;
  script.runInContext(context);
  return context.module.exports;
};

const loadModule = (filePath, sandbox) => {
  const code = fs.readFileSync(filePath, 'utf8');
  const wrappedCode = `(function(module, exports) {
    ${code}
  })`;
  const context = createContext(sandbox);
  return runScript(wrappedCode, context);
};

module.exports = { loadModule };
```

### Создание Sandbox с зависимостями

```javascript
// main.js
const logger = require('./lib/logger')('./app.log');
const db = require('./lib/db');

// Создаем Sandbox с нужными зависимостями
const sandbox = {
  console: logger,  // Подменяем console на logger
  db,               // Доступ к базе данных
  setTimeout,       // Таймеры
  setInterval,
  Buffer,           // Работа с буферами
  // НЕ даем доступ к process, require, fs и т.д.
};

// Загружаем модули в Sandbox
const { loadModule } = require('./lib/loader');
const user = loadModule('./api/user.js', sandbox);
```

### Использование в API модулях

```javascript
// api/user.js
// Модуль работает в изолированном контексте
// Имеет доступ только к тому, что передано в sandbox

// console - это наш logger
console.log('User module loaded');

// db - это наш database wrapper
const users = await db.query('SELECT * FROM users');

// process, require, fs - недоступны!
// Это обеспечивает безопасность и явное управление зависимостями
```

---

## 10. Mapping HTTP методов на CRUD

### Проблема

HTTP методы (GET, POST, PUT, DELETE) не всегда соответствуют желаемым именам методов в API.

### Решение: таблица маппинга

```javascript
// lib/http.js
const CRUD_MAPPING = {
  get: 'read',
  post: 'create',
  put: 'update',
  delete: 'delete'
};

module.exports = (routing, port) => {
  http.createServer(async (req, res) => {
    const { method, url } = req;
    const [, entityName, id] = url.split('/');

    const entity = routing[entityName];
    if (!entity) return void res.end('Not found');

    // Маппинг HTTP метода на метод CRUD
    const methodName = CRUD_MAPPING[method.toLowerCase()];
    const handler = entity[methodName];

    if (!handler) return void res.end('Not found');

    // остальная логика...
  }).listen(port);
};
```

### Кастомный маппинг

```javascript
// Можно определить свой маппинг для разных сущностей
const CUSTOM_MAPPING = {
  user: {
    get: 'find',
    post: 'register',
    put: 'updateProfile',
    delete: 'remove'
  }
};
```

---

## 11. Query Builder для SQL

### Проблема

Динамическое построение SQL запросов приводит к дублированию кода.

### Решение: простой Query Builder

```javascript
// lib/db.js
const buildSelect = (table, fields = '*', where = {}) => {
  const keys = Object.keys(where);
  const conditions = keys.map((key, i) => `${key} = $${i + 1}`).join(' AND ');
  const sql = `SELECT ${fields} FROM ${table}${conditions ? ' WHERE ' + conditions : ''}`;
  return { sql, values: Object.values(where) };
};

const buildInsert = (table, record) => {
  const keys = Object.keys(record);
  const values = Object.values(record);
  const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
  const sql = `INSERT INTO ${table} (${keys.join(', ')}) VALUES (${placeholders}) RETURNING *`;
  return { sql, values };
};

const buildUpdate = (table, id, record) => {
  const keys = Object.keys(record);
  const values = Object.values(record);
  const updates = keys.map((key, i) => `${key} = $${i + 2}`).join(', ');
  const sql = `UPDATE ${table} SET ${updates} WHERE id = $1 RETURNING *`;
  return { sql, values: [id, ...values] };
};

const buildDelete = (table, id) => {
  const sql = `DELETE FROM ${table} WHERE id = $1`;
  return { sql, values: [id] };
};
```

### Использование Query Builder

```javascript
const crud = (table) => ({
  async read(id) {
    const { sql, values } = buildSelect(table, '*', { id });
    const { rows } = await pool.query(sql, values);
    return rows[0];
  },

  async create(record) {
    const { sql, values } = buildInsert(table, record);
    const { rows } = await pool.query(sql, values);
    return rows[0];
  },

  async update(id, record) {
    const { sql, values } = buildUpdate(table, id, record);
    const { rows } = await pool.query(sql, values);
    return rows[0];
  },

  async delete(id) {
    const { sql, values } = buildDelete(table, id);
    await pool.query(sql, values);
    return { id };
  }
});
```

---

## 12. Клиентская часть - RPC over WebSocket

### Проблема

На клиенте хотим вызывать серверные методы так, как будто они локальные.

### Решение: автогенерация API proxy

```javascript
// client.js
const ws = new WebSocket('ws://localhost:8000');

const createApi = (ws) => {
  const api = {};

  // Список сущностей нужно как-то получить с сервера
  const entities = ['user', 'country', 'city'];

  for (const entity of entities) {
    api[entity] = {};

    const methods = ['read', 'create', 'update', 'delete', 'find'];
    for (const method of methods) {
      api[entity][method] = (...args) => {
        return new Promise((resolve, reject) => {
          const message = JSON.stringify({ entity, method, args });

          ws.send(message);

          ws.onmessage = (event) => {
            const response = JSON.parse(event.data);
            if (response.error) reject(new Error(response.error));
            else resolve(response.result);
          };
        });
      };
    }
  }

  return api;
};

// Использование
const api = createApi(ws);

// Теперь можно вызывать методы как обычные функции!
const user = await api.user.read(3);
console.log(user);

const newUser = await api.user.create('john', 'password123');
console.log(newUser);
```

### Автоматическое получение схемы API

```javascript
// Сервер может отдавать метаданные об API
const getApiSchema = () => {
  const schema = {};
  for (const [name, entity] of Object.entries(routing)) {
    schema[name] = Object.keys(entity);
  }
  return schema;
};

// Клиент запрашивает схему при подключении
ws.on('open', async () => {
  const schema = await api.getSchema();
  const dynamicApi = createApi(ws, schema);
});
```

---

## 13. Итоговая архитектура

### Структура проекта

```
project/
├── api/                    # Бизнес-логика (Domain Layer)
│   ├── user.js            # Методы работы с пользователями
│   ├── country.js         # Методы работы с странами
│   └── city.js            # Методы работы с городами
├── lib/                   # Инфраструктура (Infrastructure Layer)
│   ├── http.js           # HTTP транспорт
│   ├── ws.js             # WebSocket транспорт
│   ├── static.js         # Сервер статики
│   ├── db.js             # CRUD генератор
│   ├── logger.js         # Система логирования
│   ├── loader.js         # Загрузчик модулей
│   └── args.js           # Парсинг аргументов
├── static/                # Статические файлы
│   ├── index.html
│   └── client.js
├── db/                    # SQL скрипты
│   ├── install.sh
│   ├── structure.sql
│   └── data.sql
├── config/                # Конфигурация (будет добавлено)
│   └── database.json
└── main.js               # Точка входа (Application Layer)
```

### Слои архитектуры

```
┌─────────────────────────────────────────────┐
│         Presentation Layer                   │
│  (HTTP, WebSocket, CLI, etc.)               │
│         /lib/http.js, /lib/ws.js            │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│       Application Layer                      │
│  (Orchestration, Routing, DI)               │
│           /main.js                          │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│         Domain Layer                         │
│  (Business Logic, Entities)                 │
│           /api/*.js                         │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│     Infrastructure Layer                     │
│  (Database, Logging, File System)           │
│  /lib/db.js, /lib/logger.js                │
└─────────────────────────────────────────────┘
```

### Размеры компонентов (финальная версия)

| Компонент | Размер | Описание |
|-----------|--------|----------|
| API (бизнес-логика) | ~0.8 КБ | Вся доменная логика |
| DB (генератор CRUD) | ~1.5 КБ | Работа с базой данных |
| HTTP + WebSocket + Static | ~5.8 КБ | Транспортный слой |
| Logger | ~1.5 КБ | Система логирования |
| **Итого фреймворк** | **~7.5 КБ** | Вся инфраструктура |
| **Зависимости** | **1 МБ** | pg + ws (только драйверы!) |

**Для сравнения:** Express приложение занимало ~4 МБ зависимостей и имело меньше функциональности!

---

## 14. Ключевые принципы реализованной архитектуры

### 1. Разделение ответственности (Separation of Concerns)

**Что отделено:**
- Транспортный слой (HTTP, WebSocket) - `/lib/http.js`, `/lib/ws.js`
- Бизнес-логика (CRUD операции) - `/api/*.js`
- Доступ к данным (SQL) - `/lib/db.js`
- Логирование - `/lib/logger.js`
- Криптография - `/lib/hash.js`

### 2. Внедрение зависимостей (Dependency Injection)

**Как реализовано:**
```javascript
// Sandbox содержит все зависимости
const sandbox = {
  console: logger,    // Логгер вместо console
  db: dbModule,       // Доступ к БД
  crypto: hashModule  // Криптография
};

// API модули получают зависимости через контекст
// Они не делают require() - зависимости внедряются извне
```

**Преимущества:**
- Легко тестировать (можно подменить зависимости моками)
- Явное управление зависимостями
- Безопасность (ограниченный доступ к системным ресурсам)

### 3. Инверсия зависимостей (Dependency Inversion)

**Принцип:** Высокоуровневые модули не зависят от низкоуровневых. Оба зависят от абстракций.

```javascript
// Бизнес-логика НЕ знает о HTTP или WebSocket
// Она работает с абстрактным роутингом
const routing = {
  user: {
    read: async (id) => { /* ... */ }
  }
};

// Транспортный слой знает, как маршрутизировать запросы
// Но НЕ знает о бизнес-логике
const server = (routing, port) => { /* ... */ };
```

### 4. Абстракция от деталей реализации

**Что абстрагировано:**

| Абстракция | Детали реализации |
|------------|-------------------|
| Транспорт | HTTP, WebSocket, JSON-RPC |
| База данных | PostgreSQL, MySQL, SQLite |
| Логирование | Файл, консоль, удаленный сервер |
| Криптография | scrypt, bcrypt, pbkdf2 |

**Как переключаться:**
```javascript
// Изменить транспорт
const server = require('./lib/http');  // или './lib/ws'

// Изменить БД (если добавить абстракцию)
const db = require('./lib/db/postgres'); // или './lib/db/mysql'
```

### 5. Паттерн Front Controller

**Реализация:**
- Единая точка входа для всех запросов
- Централизованная обработка роутинга
- Единое место для:
  - Логирования запросов
  - Обработки ошибок
  - Аутентификации (будет добавлено)
  - Авторизации (будет добавлено)

### 6. Convention over Configuration

**Примеры:**
- Файл `/api/user.js` автоматически создает endpoint `/user`
- HTTP GET маппится на метод `read`
- HTTP POST маппится на метод `create`
- Имя файла = имя сущности = endpoint

---

## 15. Практические примеры использования

### Добавление новой сущности

**Задача:** Добавить API для работы с постами.

**Решение:** Создать один файл!

```javascript
// api/post.js
({
  async read(id) {
    const { sql, values } = db.buildSelect('posts', '*', { id });
    const { rows } = await db.pool.query(sql, values);
    return rows[0];
  },

  async create(userId, title, content) {
    const record = { user_id: userId, title, content, created_at: new Date() };
    const { sql, values } = db.buildInsert('posts', record);
    const { rows } = await db.pool.query(sql, values);
    return rows[0];
  },

  async update(id, title, content) {
    const { sql, values } = db.buildUpdate('posts', id, { title, content });
    const { rows } = await db.pool.query(sql, values);
    return rows[0];
  },

  async delete(id) {
    const { sql, values } = db.buildDelete('posts', id);
    await db.pool.query(sql, values);
    return { id };
  },

  // Дополнительные методы
  async findByUser(userId) {
    const sql = 'SELECT * FROM posts WHERE user_id = $1 ORDER BY created_at DESC';
    const { rows } = await db.pool.query(sql, [userId]);
    return rows;
  }
});
```

**Всё!** Теперь доступны endpoints:
- `GET /post/:id` - читать пост
- `POST /post` - создать пост
- `PUT /post/:id` - обновить пост
- `DELETE /post/:id` - удалить пост
- И метод `findByUser` через WebSocket/RPC

### Переключение между HTTP и WebSocket

```javascript
// Вариант 1: HTTP
const server = require('./lib/http');
server(routing, 8000);

// Вариант 2: WebSocket
const server = require('./lib/ws');
server(routing, 8000);

// Вариант 3: Оба одновременно
require('./lib/http')(routing, 8000);
require('./lib/ws')(routing, 8001);
```

### Логирование в бизнес-логике

```javascript
// api/user.js
({
  async create(login, password) {
    console.log('Creating user:', login);  // Попадет в logger

    try {
      const passwordHash = await hash(password);
      const user = await db.create('users', { login, password: passwordHash });

      console.system('New user registered:', user.id);
      return user;
    } catch (error) {
      console.error('Failed to create user:', error);
      throw error;
    }
  }
});
```

---

## 16. Что еще можно улучшить

### 1. Конфигурация

**Проблема:** Параметры подключения к БД захардкожены.

**Решение:**
```javascript
// config/database.json
{
  "host": "localhost",
  "port": 5432,
  "database": "example",
  "user": "marcus",
  "password": ""
}

// main.js
const config = require('./config/database.json');
const pool = new pg.Pool(config);
```

### 2. Валидация входных данных

**Проблема:** Нет проверки корректности данных.

**Решение:** Добавить валидацию схем.

```javascript
// lib/validator.js
const validate = (schema) => (data) => {
  for (const [key, type] of Object.entries(schema)) {
    if (typeof data[key] !== type) {
      throw new Error(`Invalid type for ${key}: expected ${type}`);
    }
  }
  return data;
};

// api/user.js
const userSchema = {
  login: 'string',
  password: 'string'
};

const validateUser = validate(userSchema);

({
  async create(login, password) {
    const data = validateUser({ login, password });
    // ...
  }
});
```

### 3. Аутентификация и авторизация

**Проблема:** Любой может вызывать любые методы.

**Решение:** Middleware для проверки прав.

```javascript
// lib/auth.js
const authenticate = async (token) => {
  // Проверка токена
  const session = await db.read('sessions', { token });
  return session ? session.userId : null;
};

const authorize = (roles) => (handler) => {
  return async (...args) => {
    const { userId } = context; // Из контекста запроса
    const user = await db.read('users', { id: userId });

    if (!roles.includes(user.role)) {
      throw new Error('Access denied');
    }

    return handler(...args);
  };
};

// Использование
({
  delete: authorize(['admin'])(async (id) => {
    // Только админы могут удалять
  })
});
```

### 4. Транзакции

**Проблема:** Нет поддержки транзакций для сложных операций.

**Решение:**
```javascript
// lib/db.js
const transaction = async (callback) => {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
};

// Использование
await transaction(async (client) => {
  await client.query('INSERT INTO orders ...');
  await client.query('UPDATE products SET stock = stock - 1 ...');
});
```

### 5. Кэширование

**Проблема:** Каждый запрос идет в БД.

**Решение:** Добавить слой кэширования.

```javascript
// lib/cache.js
const cache = new Map();

const cached = (key, ttl) => (handler) => {
  return async (...args) => {
    const cacheKey = `${key}:${JSON.stringify(args)}`;

    if (cache.has(cacheKey)) {
      return cache.get(cacheKey);
    }

    const result = await handler(...args);
    cache.set(cacheKey, result);

    setTimeout(() => cache.delete(cacheKey), ttl);

    return result;
  };
};

// Использование
({
  read: cached('user:read', 60000)(async (id) => {
    // Результат кэшируется на 60 секунд
  })
});
```

### 6. Миграции БД

**Проблема:** Нет версионирования изменений схемы БД.

**Решение:** Система миграций.

```javascript
// migrations/001_create_users.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  login VARCHAR(50) UNIQUE NOT NULL,
  password VARCHAR(255) NOT NULL
);

// migrations/002_add_email.sql
ALTER TABLE users ADD COLUMN email VARCHAR(100);

// lib/migrations.js
const runMigrations = async () => {
  const files = await fsp.readdir('./migrations');
  for (const file of files.sort()) {
    const sql = await fsp.readFile(`./migrations/${file}`, 'utf8');
    await pool.query(sql);
    console.log(`Migration ${file} applied`);
  }
};
```

---

## 17. Сравнение с популярными фреймворками

### Express vs. Собственный фреймворк

| Характеристика | Express | Наш фреймворк |
|---------------|---------|---------------|
| Зависимости | ~4 МБ | ~1 МБ |
| Размер кода | ~2.6 КБ | ~7.5 КБ |
| Абстракция транспорта | Нет | Да (HTTP, WS) |
| CRUD генератор | Нет | Да |
| Внедрение зависимостей | Нет | Да (Sandbox) |
| Логирование | Внешнее | Встроенное |
| Настройка | Middleware | Sandbox + DI |
| Производительность | Хорошая | Отличная |

### Философия отличий

**Express:**
- Middleware-based architecture
- Зависимость от экосистемы npm пакетов
- Явная работа с request/response
- Роутинг через методы app.get(), app.post()

**Наш подход:**
- Contract-based architecture
- Минимум зависимостей
- Работа с абстракциями (роутинг как объект)
- Автоматический роутинг через файловую систему

---

## 18. Важные замечания по SQL

### Почему нельзя полностью абстрагироваться от SQL?

**Проблема:** SQL - единственная деталь реализации, которая остается в доменном слое.

**Причины:**

1. **Производительность:** Абстракции вроде ORM генерируют неоптимальные запросы
2. **Специфичные возможности СУБД:** JOIN, подзапросы, агрегация, индексы
3. **Сложные запросы:** Иногда нужен сырой SQL для эффективности

**Компромисс:**
```javascript
// Простые операции - через CRUD генератор
const user = await db.read('users', id);

// Сложные запросы - сырой SQL
const topUsers = await pool.query(`
  SELECT u.*, COUNT(p.id) as post_count
  FROM users u
  LEFT JOIN posts p ON u.id = p.user_id
  GROUP BY u.id
  ORDER BY post_count DESC
  LIMIT 10
`);
```

**Альтернатива:** Использовать универсальный SQL (стандарт SQL-89/92) для переносимости между СУБД, но потерять современные возможности.

---

## 19. Преимущества реализованной архитектуры

### 1. Минимализм

- Всего ~7.5 КБ инфраструктурного кода
- Только необходимые зависимости
- Нет "магии" фреймворков

### 2. Прозрачность

- Весь код написан явно
- Легко понять, как все работает
- Нет скрытых абстракций

### 3. Гибкость

- Легко менять транспортный слой
- Просто добавлять новые сущности
- Кастомизация любой части

### 4. Производительность

- Минимум лишних слоев
- Прямая работа с базовыми API Node.js
- Эффективная работа с буферами и потоками

### 5. Безопасность

- Sandbox изолирует код
- Явное управление зависимостями
- Ограниченный доступ к системным ресурсам

### 6. Тестируемость

- Внедрение зависимостей упрощает моки
- Изолированные модули
- Простота юнит-тестирования

---

## 20. Паттерны проектирования в архитектуре

### Использованные паттерны

1. **Front Controller** - единая точка входа для HTTP запросов
2. **Dependency Injection** - внедрение зависимостей через Sandbox
3. **Factory** - генератор CRUD операций
4. **Adapter** - адаптация разных транспортов к единому интерфейсу
5. **Strategy** - выбор транспорта (HTTP, WebSocket)
6. **Module** - система модулей с изоляцией
7. **Proxy** - клиентский API proxy для RPC
8. **Template Method** - базовый CRUD с возможностью переопределения

### Принципы SOLID

1. **S - Single Responsibility:**
   - Каждый модуль отвечает за одну вещь
   - `/lib/http.js` - только HTTP
   - `/lib/db.js` - только БД

2. **O - Open/Closed:**
   - CRUD генератор открыт для расширения (обертки)
   - Закрыт для модификации

3. **L - Liskov Substitution:**
   - HTTP и WebSocket транспорты взаимозаменяемы
   - Оба реализуют контракт `(routing, port) => void`

4. **I - Interface Segregation:**
   - Модули в Sandbox получают только нужные им зависимости
   - Не весь Node.js API, а только необходимое

5. **D - Dependency Inversion:**
   - Бизнес-логика не зависит от транспорта
   - Оба зависят от абстракции (роутинг)

---

## 21. Дальнейшее развитие архитектуры

### Разделение на три слоя

**Текущее состояние:** 2 слоя (бизнес + инфраструктура)

**Следующий шаг:** 3 слоя

```
┌─────────────────────────────────┐
│   Application Services Layer    │  ← Оркестрация, workflow
│   (Use Cases, Scenarios)        │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│     Domain Services Layer       │  ← Бизнес-логика
│     (Business Rules)            │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│        Domain Model             │  ← Entities, Value Objects
│        (Pure Logic)             │
└─────────────────────────────────┘
```

### Пример разделения

```javascript
// domain/entities/User.js - чистая модель
class User {
  constructor(id, login, email) {
    this.id = id;
    this.login = login;
    this.email = email;
  }

  isValid() {
    return this.login.length >= 3 && this.email.includes('@');
  }
}

// domain/services/UserService.js - доменная логика
class UserService {
  constructor(userRepository, hashService) {
    this.userRepository = userRepository;
    this.hashService = hashService;
  }

  async register(login, password, email) {
    const passwordHash = await this.hashService.hash(password);
    const user = new User(null, login, email);

    if (!user.isValid()) {
      throw new Error('Invalid user data');
    }

    return this.userRepository.create({ login, password: passwordHash, email });
  }
}

// application/UseCases/RegisterUser.js - сценарий
class RegisterUserUseCase {
  constructor(userService, emailService, logger) {
    this.userService = userService;
    this.emailService = emailService;
    this.logger = logger;
  }

  async execute(login, password, email) {
    this.logger.log('Registering user:', login);

    const user = await this.userService.register(login, password, email);
    await this.emailService.sendWelcome(user.email);

    this.logger.log('User registered:', user.id);
    return user;
  }
}
```

---

## Ключевые выводы

### 1. Эволюция архитектуры

Мы прошли путь от монолитного Express приложения до многоуровневой архитектуры:

**Шаг 1:** Express монолит (77 строк, 4 МБ зависимостей)
↓
**Шаг 2:** Отказ от Express (простой HTTP сервер, 840 КБ)
↓
**Шаг 3:** Паттерн Front Controller (16 строк)
↓
**Шаг 4:** Генератор CRUD (универсальный)
↓
**Шаг 5:** Кастомизация CRUD (обертки)
↓
**Шаг 6:** Абстракция транспорта (HTTP + WebSocket)
↓
**Шаг 7:** Автозагрузка API (файловая система)
↓
**Шаг 8:** Логирование (цветной вывод)
↓
**Шаг 9:** Внедрение зависимостей (Sandbox)
↓
**Итог:** Чистая архитектура (7.5 КБ инфраструктуры, 1 МБ зависимостей)

### 2. Главные принципы

1. **Отделяй бизнес-логику от деталей реализации**
2. **Используй внедрение зависимостей**
3. **Абстрагируй транспортный слой**
4. **Генерируй повторяющийся код**
5. **Следуй принципам SOLID**
6. **Минимизируй зависимости**
7. **Делай код прозрачным и понятным**

### 3. Практические преимущества

- Легко добавлять новые API (один файл)
- Просто переключаться между протоколами (одна строка)
- Удобно тестировать (изолированные модули)
- Быстрая разработка (меньше boilerplate кода)
- Высокая производительность (минимум абстракций)

### 4. Что дальше?

- Разделение на Application/Domain/Infrastructure слои
- Добавление аутентификации и авторизации
- Реализация кэширования
- Система миграций БД
- Работа с другими источниками данных (Redis, файлы)

---

## Дополнительные материалы

### Рекомендуемая литература

1. **Clean Architecture** (Robert C. Martin) - принципы чистой архитектуры
2. **Domain-Driven Design** (Eric Evans) - разработка через доменную модель
3. **Patterns of Enterprise Application Architecture** (Martin Fowler) - паттерны корпоративных приложений

### Полезные ссылки

- Node.js Documentation: https://nodejs.org/docs
- PostgreSQL Documentation: https://postgresql.org/docs
- WebSocket Protocol: https://tools.ietf.org/html/rfc6455

### Исходный код примеров

Все примеры из лекции доступны в репозитории. Каждая папка содержит:
- Исходный код
- package.json с зависимостями
- SQL скрипты для БД
- README с инструкциями по запуску

**Рекомендация:** Попробуйте запустить все примеры последовательно, чтобы увидеть эволюцию архитектуры своими глазами.

---

## Практические задания

### Задание 1: Добавить новую сущность

Создайте API для работы с комментариями (comments):
- Поля: id, post_id, user_id, content, created_at
- Методы: read, create, update, delete, findByPost

### Задание 2: Реализовать валидацию

Добавьте валидацию входных данных:
- Проверка типов
- Проверка обязательных полей
- Проверка форматов (email, длина строки)

### Задание 3: Добавить аутентификацию

Реализуйте простую систему аутентификации:
- Создание сессий
- Проверка токенов
- Middleware для защищенных endpoints

### Задание 4: Кэширование

Реализуйте слой кэширования:
- In-memory кэш
- TTL для записей
- Инвалидация кэша при изменениях

### Задание 5: Переключение БД

Добавьте поддержку другой СУБД (MySQL или SQLite):
- Создайте адаптер для новой БД
- Реализуйте тот же интерфейс
- Добавьте возможность выбора БД через конфиг

---

**Конец конспекта**

Эта архитектура демонстрирует, как можно построить мощное и гибкое приложение на Node.js без тяжелых фреймворков, используя простые и понятные абстракции. Главное - следовать принципам разделения ответственности и внедрения зависимостей.
