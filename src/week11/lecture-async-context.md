# Паттерн Контекст и Асинхронный контекст в JavaScript

## Введение

Паттерн **Контекст** не входит в классический список паттернов «Банды четырёх», однако является одним из наиболее практичных и широко применяемых приёмов в современной JavaScript-разработке. Его цель — передавать зависимости и состояние между различными абстракциями системы без использования глобальных переменных, глобальных событийных шин (EventEmitter/Observer) или глобальных коллекций.

Контекст позволяет ограничить видимость данных определённой областью: подсистемой, сессией пользователя, одним HTTP-запросом или одним бизнес-процессом. Именно это делает его безопасным и предсказуемым инструментом в отличие от глобального состояния.

---

## Что такое Контекст

Контекст — это структура данных (объект, Map, экземпляр класса), которая содержит зависимости и состояние, необходимые для выполнения некоторой логики. Он передаётся явно — через параметры функций, конструкторы классов или специальные фабрики.

### Проблемы без контекста

Без паттерна Контекст разработчики вынуждены:

- Использовать глобальные переменные (загрязняют пространство имён, провоцируют гонки данных).
- Создавать глобальный EventEmitter/Observer для передачи данных между модулями.
- Записывать состояние в `global` или в статические свойства классов.

Все эти подходы приводят к хрупкому, трудно тестируемому коду.

---

## Ручная реализация контекста

### Вариант 1: Замыкания (функциональный подход)

Фабричная функция принимает контекст и возвращает экземпляр функции, который «захватывает» нужные зависимости через замыкание.

```javascript
// Вспомогательный объект для проверки прав доступа (RBAC)
const createAccessPolicy = (roles) => ({
  check(user, operation) {
    const allowed = roles[user.role];
    if (!allowed || !allowed.includes(operation)) {
      throw new Error(
        `Access denied: user "${user.name}" cannot perform "${operation}"`
      );
    }
  },
});

// Фабрика: создаёт функцию getBalance с доступом к контексту через замыкание
const createGetBalance = (context) => {
  // Деструктурируем контекст один раз — все вложенные функции его видят
  const { console: log, access, user } = context;

  return async (account) => {
    // Проверяем права доступа
    access.check(user, 'read:balance');
    log.log(`[getBalance] User "${user.name}" reads balance for account ${account}`);

    // В реальном приложении здесь был бы запрос к базе данных
    const balances = { acc001: 1500, acc002: 3200 };
    const balance = balances[account] ?? 0;

    return { account, balance };
  };
};

// --- Использование ---
const accessPolicy = createAccessPolicy({
  guest: [],
  user: ['read:balance'],
  admin: ['read:balance', 'read:transactions'],
});

const context = {
  console,           // зависимость: логгер
  access: accessPolicy, // зависимость: политика доступа
  user: { name: 'Markus', role: 'user' }, // состояние: текущий пользователь
};

const getBalance = createGetBalance(context);

getBalance('acc001').then((result) => {
  console.log('Result:', result);
  // Result: { account: 'acc001', balance: 1500 }
});
```

### Вариант 2: ООП — контекст через конструктор (Revealing Constructor)

Паттерн «Открытый конструктор» (Revealing Constructor): контекст передаётся в конструктор и сохраняется как приватное состояние экземпляра.

```javascript
class AccessPolicy {
  #roles;
  constructor(roles) {
    this.#roles = roles;
  }
  check(user, operation) {
    const allowed = this.#roles[user.role];
    if (!allowed || !allowed.includes(operation)) {
      throw new Error(`Access denied for "${user.name}" on "${operation}"`);
    }
  }
}

class AccountService {
  // Контекст сохраняется при создании — это и есть Revealing Constructor
  #context;

  constructor(context) {
    this.#context = context;
  }

  async getBalance(account) {
    const { console: log, access, user } = this.#context;
    access.check(user, 'read:balance');
    log.log(`[AccountService.getBalance] user=${user.name}, account=${account}`);

    const balances = { acc001: 1500, acc002: 3200 };
    return { account, balance: balances[account] ?? 0 };
  }

  async getTransactions(account) {
    const { console: log, access, user } = this.#context;
    access.check(user, 'read:transactions');
    log.log(`[AccountService.getTransactions] user=${user.name}, account=${account}`);

    return { account, transactions: [{ id: 't1', amount: -100 }, { id: 't2', amount: 500 }] };
  }
}

// --- Использование ---
const policy = new AccessPolicy({
  user: ['read:balance'],
  admin: ['read:balance', 'read:transactions'],
});

const ctx = {
  console,
  access: policy,
  user: { name: 'Alice', role: 'admin' },
};

const service = new AccountService(ctx);

service.getBalance('acc001').then(console.log);
service.getTransactions('acc001').then(console.log);
```

### Вариант 3: Фабрика сервисов

Когда сервис состоит из нескольких функций, удобно обернуть их в единую фабрику. Внутри фабрики все функции разделяют один лексический контекст.

```javascript
const createAccountService = (context) => {
  // Деструктурируем один раз для всего модуля/фабрики
  const { console: log, access, user } = context;

  const getBalance = async (account) => {
    access.check(user, 'read:balance');
    log.log(`getBalance: ${user.name} -> ${account}`);
    return { account, balance: 1500 };
  };

  const getTransactions = async (account) => {
    access.check(user, 'read:transactions');
    log.log(`getTransactions: ${user.name} -> ${account}`);
    return { account, transactions: [] };
  };

  // Возвращаем публичный интерфейс сервиса
  return { getBalance, getTransactions };
};

// --- Использование ---
const ctx = {
  console,
  access: { check: () => {} }, // упрощённая заглушка
  user: { name: 'Bob', role: 'admin' },
};

const accountService = createAccountService(ctx);
accountService.getBalance('acc002').then(console.log);
```

---

## Функциональный подход: Pipeline с контекстом и трассировкой

Функциональное программирование позволяет выстроить цепочку асинхронных шагов, каждый из которых получает контекст и возвращает обновлённый контекст внутри Promise. Это напоминает композицию функций, но в прямом (слева направо) порядке.

```javascript
import { randomUUID } from 'node:crypto';

// Генерирует уникальный идентификатор запроса
const makeRequestId = () => `${Date.now()}-${randomUUID()}`;

// Добавляет текущий шаг в массив трассировки внутри контекста
const appendTrace = (context, stepName) => {
  const trace = context.trace ?? [];
  return { ...context, trace: [...trace, stepName] };
};

// Pipeline: принимает массив асинхронных шагов и контекст,
// последовательно прогоняет контекст через каждый шаг
const pipeline =
  (...steps) =>
  (context) =>
    steps.reduce(
      (promise, step) => promise.then((ctx) => step(ctx)),
      Promise.resolve(context)
    );

// --- Шаги пайплайна ---

// Шаг 1: трассировка — добавляет request ID и логирует
const trace = async (context) => {
  const { console: log } = context;
  const requestId = context.requestId ?? makeRequestId();
  const next = appendTrace({ ...context, requestId }, 'trace');
  log.log(`[trace] requestId=${requestId}`);
  return next;
};

// Шаг 2: аутентификация — определяет роль пользователя
const authenticate = async (context) => {
  const { console: log, headers } = context;
  const user = headers?.user
    ? { name: headers.user, role: 'user' }
    : { name: 'anonymous', role: 'guest' };
  log.log(`[authenticate] user=${user.name}, role=${user.role}`);
  return appendTrace({ ...context, user }, 'authenticate');
};

// Шаг 3: проверка прав доступа
const checkAccess = async (context) => {
  const { console: log, user, accessRules } = context;
  const allowed = accessRules[user.role] ?? [];
  if (!allowed.includes('read:balance')) {
    throw new Error(`Access denied for role "${user.role}"`);
  }
  log.log(`[checkAccess] OK for ${user.name}`);
  return appendTrace(context, 'checkAccess');
};

// Шаг 4: получение баланса (бизнес-логика)
const fetchBalance = async (context) => {
  const { console: log, account, user } = context;
  const balance = 1500; // в реальности — запрос к БД
  log.log(`[fetchBalance] account=${account}, balance=${balance}`);
  const result = { status: 200, body: { balance } };
  return appendTrace({ ...context, result }, 'fetchBalance');
};

// --- Сборка и запуск ---
const execute = pipeline(trace, authenticate, checkAccess, fetchBalance);

const initialContext = {
  console,
  headers: { user: 'Markus' },
  account: 'acc001',
  accessRules: {
    guest: [],
    user: ['read:balance'],
    admin: ['read:balance', 'read:transactions'],
  },
};

execute(initialContext).then((finalCtx) => {
  console.log('Trace:', finalCtx.trace);
  console.log('Result:', finalCtx.result);
});
/*
[trace] requestId=1700000000000-xxxxxxxx-...
[authenticate] user=Markus, role=user
[checkAccess] OK for Markus
[fetchBalance] account=acc001, balance=1500
Trace: [ 'trace', 'authenticate', 'checkAccess', 'fetchBalance' ]
Result: { status: 200, body: { balance: 1500 } }
*/
```

---

## Асинхронный контекст

Ручная передача контекста через параметры функций удобна, но требует явного пробрасывания через каждый вызов. В глубоко вложенных или распределённых системах это становится обременительным. Для решения этой проблемы в Node.js существует механизм **асинхронного контекста**.

Исторически в Node.js для этого применялись:

- `domains` — устаревший модуль, эмулировавший контексты (deprecated).
- Сторонние библиотеки «зон» (Zone.js для Angular, аналоги для браузера).

Современное и официальное решение — **`AsyncLocalStorage`** из модуля `node:async_hooks`.

---

## AsyncLocalStorage

`AsyncLocalStorage` позволяет хранить данные, которые автоматически доступны во всех асинхронных операциях, запущенных в рамках одного «контекстного запуска» (`run`). Это работает благодаря тому, что Node.js отслеживает «дерево» асинхронных ресурсов и знает, к какому корневому `run`-вызову относится каждый колбэк.

### Ключевые методы

| Метод | Описание |
|---|---|
| `new AsyncLocalStorage()` | Создаёт хранилище |
| `.run(store, callback)` | Запускает `callback`, связывая с ним `store` |
| `.getStore()` | Возвращает `store` текущего контекста из любого вложенного вызова |
| `.enterWith(store)` | Устанавливает `store` для текущего асинхронного контекста без запуска колбэка |

### Принцип работы

```
asyncLocalStorage.run(store, () => {
  // Здесь и во ВСЕХ вложенных вызовах (даже асинхронных)
  // asyncLocalStorage.getStore() вернёт этот store
});
```

---

## Реализация с AsyncLocalStorage

### Базовый пример: хранение контекста запроса

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';

// Создаём хранилище — один экземпляр на всё приложение
const requestContext = new AsyncLocalStorage();

// Имитация middleware в HTTP-сервере
const handleRequest = (req) => {
  // Формируем контекст запроса
  const store = {
    requestId: `req-${Date.now()}`,
    user: req.user ?? { name: 'anonymous', role: 'guest' },
    console, // логгер (можно заменить на winston, pino и т.д.)
  };

  // Запускаем обработчик внутри контекста
  requestContext.run(store, () => {
    processRequest(req.account);
  });
};

// Вспомогательная функция доступа к контексту — вызывается из любого места
const getContext = () => {
  const store = requestContext.getStore();
  if (!store) throw new Error('No request context found');
  return store;
};

// Функция бизнес-логики — НЕ принимает контекст через параметры
const getBalance = async (account) => {
  const { console: log, user, requestId } = getContext();
  log.log(`[${requestId}] getBalance: user=${user.name}, account=${account}`);
  // Запрос к базе данных...
  return { account, balance: 1500 };
};

// Ещё одна функция, которая тоже читает из контекста
const getTransactions = async (account) => {
  const { console: log, user, requestId } = getContext();
  log.log(`[${requestId}] getTransactions: user=${user.name}, account=${account}`);
  return { account, transactions: [] };
};

const processRequest = async (account) => {
  const balance = await getBalance(account);
  const transactions = await getTransactions(account);
  console.log({ balance, transactions });
};

// --- Имитация двух одновременных запросов ---
handleRequest({ user: { name: 'Markus', role: 'user' }, account: 'acc001' });
handleRequest({ user: { name: 'Alice', role: 'admin' }, account: 'acc002' });

// Каждый вызов handleRequest создаёт свой изолированный store.
// Функции getBalance и getTransactions читают именно свой store,
// даже если запросы выполняются одновременно.
```

### Реалистичный пример: HTTP-сервер с трассировкой

```javascript
import { createServer } from 'node:http';
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

const als = new AsyncLocalStorage();

// Функция-хелпер: получить store или бросить ошибку
const ctx = () => {
  const store = als.getStore();
  if (!store) throw new Error('AsyncLocalStorage store is not initialized');
  return store;
};

// Логгер, который автоматически добавляет requestId из контекста
const logger = {
  log: (message) => {
    const store = als.getStore();
    const prefix = store ? `[${store.requestId}]` : '[no-ctx]';
    console.log(`${prefix} ${message}`);
  },
};

// Сервис — не знает ни о каком контексте напрямую
class UserService {
  async getUser(userId) {
    logger.log(`UserService.getUser(${userId})`);
    // Имитация обращения к БД
    return { id: userId, name: 'Markus', role: 'user' };
  }
}

class BalanceService {
  #userService;
  constructor(userService) {
    this.#userService = userService;
  }

  async getBalance(userId, account) {
    logger.log(`BalanceService.getBalance(${userId}, ${account})`);
    const user = await this.#userService.getUser(userId);
    logger.log(`  -> resolved user: ${user.name}`);
    return { user, account, balance: 2500 };
  }
}

// Middleware: устанавливает контекст для каждого запроса
const withRequestContext = (req, res, next) => {
  const store = {
    requestId: randomUUID(),
    method: req.method,
    url: req.url,
    startTime: Date.now(),
  };

  als.run(store, () => {
    logger.log(`Incoming ${req.method} ${req.url}`);
    next();
  });
};

// Обработчик запроса
const userService = new UserService();
const balanceService = new BalanceService(userService);

const server = createServer((req, res) => {
  withRequestContext(req, res, async () => {
    try {
      const result = await balanceService.getBalance('user-1', 'acc001');
      const duration = Date.now() - ctx().startTime;
      logger.log(`Request completed in ${duration}ms`);
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(result));
    } catch (error) {
      logger.log(`Error: ${error.message}`);
      res.writeHead(500);
      res.end(JSON.stringify({ error: error.message }));
    }
  });
});

server.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});

/*
Пример вывода при двух одновременных запросах:

[a1b2c3d4-...] Incoming GET /balance
[e5f6g7h8-...] Incoming GET /balance
[a1b2c3d4-...] BalanceService.getBalance(user-1, acc001)
[e5f6g7h8-...] BalanceService.getBalance(user-1, acc001)
[a1b2c3d4-...] UserService.getUser(user-1)
[e5f6g7h8-...] UserService.getUser(user-1)
[a1b2c3d4-...]   -> resolved user: Markus
[e5f6g7h8-...]   -> resolved user: Markus
[a1b2c3d4-...] Request completed in 12ms
[e5f6g7h8-...] Request completed in 14ms

Каждый запрос видит только свой requestId — изоляция гарантирована.
*/
```

---

## Вложенные и наследуемые контексты

Иногда нужно создать «дочерний» контекст, который расширяет родительский, не мутируя его. Это важно, когда два бизнес-процесса работают параллельно и используют общий базовый контекст: если один процесс изменит объект контекста, это может повлиять на другой (race condition / data corruption).

### Безопасный паттерн: иммутабельное расширение контекста

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';

const als = new AsyncLocalStorage();

// Создаём базовый контекст (только зависимости, без состояния пользователя)
const createBaseContext = () =>
  Object.freeze({
    console,
    accessRules: {
      guest: [],
      user: ['read:balance'],
      admin: ['read:balance', 'read:transactions'],
    },
  });

// Расширяем базовый контекст пользователем — возвращаем НОВЫЙ объект
const withUser = (baseContext, user) =>
  Object.freeze({ ...baseContext, user });

// Расширяем контекст идентификатором запроса
const withRequestId = (context, requestId) =>
  Object.freeze({ ...context, requestId });

// --- Использование ---
const baseCtx = createBaseContext();

// Контекст для пользователя Alice
const aliceCtx = withUser(baseCtx, { name: 'Alice', role: 'admin' });

// Контекст для конкретного запроса Alice
const requestCtx = withRequestId(aliceCtx, 'req-001');

// Запускаем бизнес-логику в изолированном контексте
als.run(requestCtx, async () => {
  const store = als.getStore();
  store.console.log(`[${store.requestId}] User: ${store.user.name}`);

  // Параллельный запуск двух долгих операций
  // Каждая получает собственный дочерний контекст — они не мешают друг другу
  const childCtx1 = withRequestId(store, 'req-001-transactions');
  const childCtx2 = withRequestId(store, 'req-001-passport');

  await Promise.all([
    als.run(childCtx1, async () => {
      const s = als.getStore();
      // Долгая операция: сбор транзакций за 2 года
      s.console.log(`[${s.requestId}] Collecting transactions...`);
      await new Promise((r) => setTimeout(r, 100)); // имитация задержки
      s.console.log(`[${s.requestId}] Transactions done`);
    }),
    als.run(childCtx2, async () => {
      const s = als.getStore();
      // Быстрая операция: паспортные данные
      s.console.log(`[${s.requestId}] Fetching passport data...`);
      // Даже если эта операция изменит что-то в своём store,
      // это не затронет store первой операции
      s.console.log(`[${s.requestId}] Passport data done`);
    }),
  ]);
});

/*
[req-001] User: Alice
[req-001-transactions] Collecting transactions...
[req-001-passport] Fetching passport data...
[req-001-passport] Passport data done
[req-001-transactions] Transactions done
*/
```

---

## Применение в Node.js

### Типичные сценарии использования контекста и AsyncLocalStorage

**1. Трассировка запросов (Request Tracing)**

В микросервисной архитектуре один бизнес-процесс может охватывать несколько сервисов. Контекст хранит `traceId` / `spanId`, которые передаются между сервисами через HTTP-заголовки (например, `X-Request-ID`, `X-Trace-ID`) и автоматически добавляются во все логи.

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

const traceStorage = new AsyncLocalStorage();

// Обёртка над fetch, которая автоматически добавляет trace-заголовки
const tracedFetch = (url, options = {}) => {
  const store = traceStorage.getStore();
  const headers = {
    ...options.headers,
    ...(store ? { 'X-Trace-ID': store.traceId, 'X-Span-ID': store.spanId } : {}),
  };
  return fetch(url, { ...options, headers });
};

// Запуск обработки запроса с трассировкой
const handleIncomingRequest = (incomingTraceId) => {
  const store = {
    traceId: incomingTraceId ?? randomUUID(), // берём от клиента или создаём новый
    spanId: randomUUID(),                     // spanId уникален для каждого сервиса
  };

  traceStorage.run(store, async () => {
    // Все вызовы tracedFetch внутри автоматически получат заголовки трассировки
    await tracedFetch('http://user-service/api/user/1');
    await tracedFetch('http://balance-service/api/balance/acc001');
  });
};
```

**2. Контекст транзакции базы данных**

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';

const dbTransactionStorage = new AsyncLocalStorage();

// Получить текущее соединение/транзакцию из контекста
const getDbConnection = () => {
  const store = dbTransactionStorage.getStore();
  return store?.connection ?? globalDbPool;
};

// Запустить блок кода внутри транзакции БД
const withTransaction = async (globalDbPool, callback) => {
  const connection = await globalDbPool.getConnection();
  await connection.beginTransaction();

  return dbTransactionStorage.run({ connection }, async () => {
    try {
      const result = await callback();
      await connection.commit();
      return result;
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  });
};

// Репозитории используют getDbConnection() — не знают о транзакции напрямую
const userRepository = {
  async findById(id) {
    const conn = getDbConnection();
    return conn.query('SELECT * FROM users WHERE id = ?', [id]);
  },
  async update(id, data) {
    const conn = getDbConnection();
    return conn.query('UPDATE users SET ? WHERE id = ?', [data, id]);
  },
};
```

**3. Мультитенантность (multi-tenancy)**

```javascript
import { AsyncLocalStorage } from 'node:async_hooks';

const tenantStorage = new AsyncLocalStorage();

const getTenantId = () => {
  const store = tenantStorage.getStore();
  if (!store) throw new Error('Tenant context not set');
  return store.tenantId;
};

// Middleware для Express
const tenantMiddleware = (req, res, next) => {
  const tenantId = req.headers['x-tenant-id'];
  if (!tenantId) {
    res.status(400).json({ error: 'Missing X-Tenant-ID header' });
    return;
  }
  tenantStorage.run({ tenantId }, next);
};

// Сервис автоматически получает tenantId без явной передачи
class ProductService {
  async getProducts() {
    const tenantId = getTenantId();
    // Запрос данных только для текущего тенанта
    console.log(`Fetching products for tenant: ${tenantId}`);
    return []; // результат из БД
  }
}
```

---

## Итог

### Когда использовать ручной контекст (параметры / замыкания / конструктор)

- Небольшие модули с предсказуемой глубиной вызовов.
- Когда важна явность: читатель кода сразу видит, какие зависимости используются.
- В функциональных пайплайнах, где контекст передаётся как иммутабельный объект между шагами.

### Когда использовать AsyncLocalStorage

- HTTP-серверы: контекст запроса (requestId, user, tenant) нужен во множестве вложенных функций.
- Логирование с автоматическим добавлением trace ID без передачи его через все слои.
- Транзакции БД: соединение/транзакция должны быть доступны в любом репозитории без явной передачи.
- Микросервисная архитектура: распределённая трассировка через сервисные границы.

### Риски и как их избежать

| Риск | Решение |
|---|---|
| Гонка данных (race condition): два бизнес-процесса используют один контекст | Создавать дочерние контексты через `Object.freeze({ ...parentCtx, ...extra })` и `als.run(childCtx, ...)` |
| Утечки памяти: store удерживает объекты дольше необходимого | Хранить в store только ссылки, очищать при завершении обработки запроса |
| Неявность: разработчик не видит, что функция читает глобальный store | Документировать предполагаемый контекст, использовать TypeScript типы для store |
| Производительность: накладные расходы AsyncLocalStorage | Минимальны в современном Node.js (v12.17+); не критичны для I/O-bound нагрузки |

### Ключевые выводы

1. **Контекст** — это инструмент передачи зависимостей и состояния без глобальных переменных, ограниченный определённой областью выполнения.
2. **Ручная передача** через замыкания или конструктор — явная, понятная, но требует пробрасывания через каждый уровень.
3. **Функциональный Pipeline** позволяет элегантно компоновать асинхронные шаги, передавая иммутабельный контекст между ними.
4. **AsyncLocalStorage** решает проблему «пробрасывания» автоматически: store доступен из любой глубины вложенности асинхронных вызовов.
5. **Иммутабельность контекста** (`Object.freeze`, spread-копирование) защищает от гонок данных при параллельном выполнении.
6. Паттерн активно применяется в популярных фреймворках: Express, Fastify, Nest.js используют аналогичные механизмы для хранения контекста запроса.
