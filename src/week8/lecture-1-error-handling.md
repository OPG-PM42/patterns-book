# Обработка ошибок и исключений в Node.js / JavaScript

---

## Введение

Обработка ошибок — одна из наиболее недооценённых дисциплин в разработке серверных
приложений. Написать код, который работает в штатном режиме, относительно просто.
Написать код, который корректно ведёт себя при сбоях, не теряет ошибки, не роняет
процесс и оставляет систему в согласованном состоянии — принципиально сложнее.

В Node.js это особенно критично: один процесс обслуживает тысячи одновременных
пользователей. Ошибка, не обработанная должным образом, может уронить весь сервер
или оставить ресурсы (соединения, файловые дескрипторы, блокировки) навсегда занятыми.

Лекция охватывает три уровня:

- **Классификация ошибок** — code errors, soft errors, operational errors
- **Паттерны обработки** — fail fast, return early, error cause chaining, AggregateError
- **Архитектура** — graceful shutdown, изоляция, логирование, стратегии деплоя

---

## 1. Три типа ошибок

Прежде чем выбирать инструменты обработки ошибок, необходимо понять: не все ошибки
одинаковы. Смешивание разных типов ошибок в единую стратегию — один из наиболее
распространённых архитектурных просчётов.

### 1.1. Code Errors — ошибки в коде

Code errors — это ошибки программиста: обращение к несуществующему методу,
вызов не-функции как функции, несоответствие типов, нарушение контракта API.

```javascript
// code-errors.js
// Типичные примеры code errors

// 1. Вызов метода у undefined
function getUserName(user) {
  // Если user не передан — TypeError: Cannot read properties of undefined
  return user.name.toUpperCase();
}

// Правильно: явная проверка контракта (fail fast)
function getUserNameSafe(user) {
  if (user === null || user === undefined) {
    throw new TypeError('getUserName: аргумент user обязателен');
  }
  if (typeof user.name !== 'string') {
    throw new TypeError(`getUserName: user.name должен быть строкой, получен ${typeof user.name}`);
  }
  return user.name.toUpperCase();
}

// 2. Вызов не-функции
function processItems(items, callback) {
  if (typeof callback !== 'function') {
    // Бросаем немедленно, не дожидаясь момента вызова callback
    throw new TypeError(`processItems: callback должен быть функцией, получен ${typeof callback}`);
  }
  return items.map(callback);
}

// 3. Асинхронный контракт — сложнее всего описать декларативно
// По сигнатуре невозможно понять: callback вызывается синхронно или асинхронно?
// Это покрывается только тестами.
function fetchData(id, callback) {
  // Антипаттерн: иногда синхронный, иногда асинхронный (Zalgo)
  const cached = cache.get(id);
  if (cached) {
    callback(null, cached); // синхронно!
  } else {
    db.find(id, callback); // асинхронно!
  }
}

// Правильно: всегда асинхронный
function fetchDataCorrect(id, callback) {
  const cached = cache.get(id);
  if (cached) {
    // queueMicrotask гарантирует асинхронность даже при кэш-хите
    queueMicrotask(() => callback(null, cached));
  } else {
    db.find(id, callback);
  }
}
```

TypeScript помогает поймать часть code errors на этапе компиляции, но не является
полноценным контрактным программированием. Например, TypeScript не может выразить:
асинхронные конструкторы, ковариантность коллбэков, инварианты предметной области.
То, что TypeScript не покрывает — закрывается тестами.

---

### 1.2. Soft Errors — ошибки предметной области

Soft errors — ожидаемые ситуации в бизнес-логике, которые не являются ошибками кода.
Система работает корректно; просто условия не позволяют выполнить операцию.

Примеры:
- Запрашивается 100 м кабеля, но на складе только 30 м
- Пользователь запрашивает отчёт за месяц, который ещё не завершился
- Сотрудник хочет уйти в отпуск, но у него запланированы работы

**Антипаттерн: множество пустых классов-исключений**

```javascript
// ПЛОХО: copy-paste классы, неудобны в поддержке
class InsufficientStockError extends Error {}
class ReportNotReadyError extends Error {}
class ScheduleConflictError extends Error {}
class InvalidPeriodError extends Error {}
// ... ещё 46 классов, все одинаковые
```

**Правильно: единый класс с кодом ошибки**

```javascript
// domain-error.js
// Один универсальный класс для всех ошибок предметной области.
// Код ошибки — строка, как в Node.js (err.code === 'ENOENT' и т.д.)

export class DomainError extends Error {
  constructor(code, message, context = {}) {
    super(message);
    this.name = 'DomainError';
    this.code = code;         // строковый код: 'INSUFFICIENT_STOCK', 'REPORT_NOT_READY'
    this.context = context;   // произвольные данные для логирования
  }
}

// Использование
import { DomainError } from './domain-error.js';

function reserveCable(warehouseStock, requestedMeters) {
  if (requestedMeters > warehouseStock) {
    return {
      ok: false,
      error: new DomainError(
        'INSUFFICIENT_STOCK',
        `Запрошено ${requestedMeters} м, на складе ${warehouseStock} м`,
        { requested: requestedMeters, available: warehouseStock }
      ),
    };
  }
  return { ok: true, reserved: requestedMeters };
}

// Вызывающий код ветвится по коду ошибки — без instanceof
const result = reserveCable(30, 100);
if (!result.ok) {
  if (result.error.code === 'INSUFFICIENT_STOCK') {
    console.log(`Нет нужного количества. Доступно: ${result.error.context.available} м`);
  }
}
```

Soft-ошибки **возвращаются**, а не бросаются (`throw`). Это разграничение принципиально:
`throw` предназначен для ситуаций, из которых текущий поток управления не может
восстановиться. Нехватка товара — нормальная рабочая ситуация, а не катастрофа.

---

### 1.3. Operational Errors — ошибки инфраструктуры

Operational errors — сбои среды выполнения: DNS недоступен, сокет не открылся,
нет подключения к базе данных, диск отключился, кончилась оперативная память.

Причина — не ошибка кода и не нарушение бизнес-правил. Сломалась инфраструктура.

```javascript
// operational-retry.js
// Стратегия автоматического повтора с экспоненциальным backoff.
//
// Почему экспоненциальный, а не фиксированный интервал?
// Постоянная "долбёжка" сервиса, который и так перегружен,
// только усугубляет ситуацию. Экспоненциальный backoff
// даёт инфраструктуре время на восстановление.

import { setTimeout as delay } from 'node:timers/promises';

export async function withRetry(operation, options = {}) {
  const {
    maxAttempts = 5,
    baseDelayMs = 200,    // начальная задержка
    maxDelayMs = 30_000,  // потолок задержки (30 секунд)
    factor = 2,           // основание экспоненты
    signal,               // AbortSignal для принудительной остановки
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (err) {
      lastError = err;

      // Прерываемся немедленно, если получили сигнал отмены
      if (signal?.aborted) throw err;

      // На последней попытке не ждём — сразу бросаем ошибку
      if (attempt === maxAttempts) break;

      // Экспоненциальный backoff: 200ms, 400ms, 800ms, 1600ms...
      // Добавляем случайный jitter, чтобы несколько инстансов
      // не отправляли retry одновременно (thundering herd problem)
      const exponentialDelay = baseDelayMs * Math.pow(factor, attempt - 1);
      const jitter = Math.random() * baseDelayMs;
      const waitMs = Math.min(exponentialDelay + jitter, maxDelayMs);

      console.warn(`Попытка ${attempt}/${maxAttempts} провалилась: ${err.message}. Повтор через ${Math.round(waitMs)} мс`);
      await delay(waitMs, undefined, { signal });
    }
  }

  throw lastError;
}

// Пример: подключение к базе данных с retry
async function connectToDatabase() {
  return withRetry(
    () => db.connect('postgresql://localhost/mydb'),
    { maxAttempts: 5, baseDelayMs: 500 }
  );
}
```

**Стратегия кэша при недоступности источника:**

```javascript
// cache-fallback.js
// Если актуальные данные получить не удалось — отдаём кэшированные
// и планируем обновление кэша позже.
//
// Пользователь получает ответ (пусть устаревший), а не ошибку.
// Инженеры получают алерт о деградации сервиса.

const cache = new Map();

async function getProductCatalog() {
  try {
    const data = await fetchFromDatabase('SELECT * FROM products');
    cache.set('catalog', { data, updatedAt: Date.now() });
    return data;
  } catch (err) {
    const cached = cache.get('catalog');

    if (cached) {
      console.warn('База недоступна, отдаём кэш от', new Date(cached.updatedAt).toISOString());

      // Планируем обновление кэша через 10 минут
      setTimeout(() => getProductCatalog().catch(console.error), 10 * 60 * 1000);

      return cached.data;
    }

    // Кэша нет — ошибка неизбежна, эскалируем её
    throw new Error('Сервис каталога недоступен, кэш отсутствует', { cause: err });
  }
}
```

---

## 2. Паттерны работы с ошибками

### 2.1. Error Cause — причинная цепочка ошибок

Свойство `cause` объекта `Error` позволяет выстраивать цепочки причин. Это стандарт
ECMAScript 2022, доступный начиная с Node.js 16.9.

```javascript
// error-cause.js
// Цепочка причин: операционная ошибка -> soft-ошибка бизнес-логики.
//
// Без cause: в логах видна только верхняя ошибка.
// С cause: полная цепочка, от первопричины до высокоуровневого эффекта.

import { DomainError } from './domain-error.js';

async function placeOrder(orderId, productId, quantity) {
  let warehouseData;

  try {
    warehouseData = await warehouse.getStock(productId);
  } catch (infraErr) {
    // Операционная ошибка становится причиной бизнес-ошибки
    throw new DomainError(
      'ORDER_FAILED',
      `Не удалось оформить заказ ${orderId}: склад недоступен`,
      { orderId, productId },
      // { cause } — стандартный второй аргумент Error в ES2022
    );
    // Правильная запись с cause:
    // throw Object.assign(
    //   new DomainError('ORDER_FAILED', `Не удалось оформить заказ ${orderId}`),
    //   { cause: infraErr }
    // );
  }

  if (warehouseData.stock < quantity) {
    return {
      ok: false,
      error: new DomainError('INSUFFICIENT_STOCK', 'Недостаточно товара', {
        available: warehouseData.stock,
        requested: quantity,
      }),
    };
  }

  return { ok: true };
}

// Правильный способ создания ошибки с cause через конструктор:
function createOrderError(message, cause) {
  return new Error(message, { cause });
}

// Извлечение полной цепочки причин для логирования
function formatErrorChain(err, depth = 0) {
  const indent = '  '.repeat(depth);
  let output = `${indent}${err.name}: ${err.message}`;
  if (err.cause) {
    output += '\n' + `${indent}Причина:`;
    output += '\n' + formatErrorChain(err.cause, depth + 1);
  }
  return output;
}

// Пример вывода цепочки:
// Error: Не удалось оформить заказ #42
//   Причина:
//   Error: connect ECONNREFUSED 127.0.0.1:5432
//     Причина:
//     Error: Таймаут подключения (5000ms)
```

---

### 2.2. AggregateError — агрегация нескольких ошибок

`AggregateError` встроен в JavaScript (Node.js 15+). Позволяет собрать несколько
независимых ошибок в один объект — например, при параллельной валидации.

```javascript
// aggregate-error.js
// Валидация заявки на отпуск.
// Несколько условий проверяются независимо, все нарушения
// возвращаются пользователю сразу, а не по одному.

export class DomainError extends Error {
  constructor(code, message, context = {}) {
    super(message);
    this.name = 'DomainError';
    this.code = code;
    this.context = context;
  }
}

async function validateVacationRequest(employeeId, startDate, endDate) {
  // Запускаем все проверки параллельно
  const [hasScheduledWork, hasSufficientBalance, isApproverAvailable] = await Promise.all([
    checkScheduledWork(employeeId, startDate, endDate),
    checkVacationBalance(employeeId, startDate, endDate),
    checkApproverAvailability(startDate, endDate),
  ]);

  const errors = [];

  if (hasScheduledWork) {
    errors.push(new DomainError(
      'SCHEDULED_WORK_CONFLICT',
      'На период отпуска запланированы работы',
      { employeeId, startDate, endDate }
    ));
  }

  if (!hasSufficientBalance) {
    errors.push(new DomainError(
      'INSUFFICIENT_VACATION_BALANCE',
      'Недостаточно дней отпуска',
      { employeeId }
    ));
  }

  if (!isApproverAvailable) {
    errors.push(new DomainError(
      'APPROVER_UNAVAILABLE',
      'Руководитель недоступен в указанный период',
      { startDate, endDate }
    ));
  }

  if (errors.length > 0) {
    // Все причины отказа — в одном объекте
    return {
      ok: false,
      error: new AggregateError(errors, 'Заявка на отпуск не прошла валидацию'),
    };
  }

  return { ok: true };
}

// Обработка на стороне вызывающего кода
const result = await validateVacationRequest('emp-42', '2025-08-01', '2025-08-14');

if (!result.ok) {
  const { error } = result;
  console.error(error.message);
  // Перебираем все причины
  for (const reason of error.errors) {
    console.error(`  [${reason.code}] ${reason.message}`);
  }
}

// Вывод:
// Заявка на отпуск не прошла валидацию
//   [SCHEDULED_WORK_CONFLICT] На период отпуска запланированы работы
//   [INSUFFICIENT_VACATION_BALANCE] Недостаточно дней отпуска
```

`AggregateError` может выступать как `cause` для ошибки более высокого уровня:
операционные и soft-ошибки агрегируются, а затем передаются как причина в верхнеуровневую
ошибку.

---

### 2.3. Fail Fast vs Return Early

Два внешне похожих паттерна решают разные проблемы:

| Паттерн | Тип ошибки | Механизм | Когда применять |
|---------|-----------|----------|-----------------|
| **Fail Fast** | Code error | `throw` | Нарушение контракта, недопустимое состояние |
| **Return Early** | Soft error | `return { ok, error }` | Ожидаемые бизнес-ситуации |

```javascript
// fail-fast-vs-return-early.js

import { DomainError } from './domain-error.js';

// --- Fail Fast: для code errors ---
// Контракт нарушен — бросаем немедленно, не уходим вглубь функции.
function parseOrderConfig(rawConfig) {
  // Fail fast: проверяем контракт на входе
  if (rawConfig === null || rawConfig === undefined) {
    throw new TypeError('parseOrderConfig: rawConfig обязателен');
  }
  if (typeof rawConfig.warehouseId !== 'string') {
    throw new TypeError('parseOrderConfig: warehouseId должен быть строкой');
  }
  if (!Number.isInteger(rawConfig.quantity) || rawConfig.quantity <= 0) {
    throw new RangeError('parseOrderConfig: quantity должен быть положительным целым числом');
  }

  return {
    warehouseId: rawConfig.warehouseId.trim(),
    quantity: rawConfig.quantity,
    priority: rawConfig.priority ?? 'normal',
  };
}

// --- Return Early: для soft errors ---
// Условие не выполнено, но это нормальная рабочая ситуация.
async function processOrder(order) {
  // Проверка 1: товар в наличии?
  const stock = await getStock(order.warehouseId, order.productId);
  if (stock < order.quantity) {
    // Возвращаем раньше — не уходим вглубь функции
    return {
      ok: false,
      error: new DomainError('INSUFFICIENT_STOCK', 'Недостаточно товара на складе', {
        available: stock,
        requested: order.quantity,
      }),
    };
  }

  // Проверка 2: склад принимает заказы?
  const warehouseStatus = await getWarehouseStatus(order.warehouseId);
  if (warehouseStatus !== 'active') {
    return {
      ok: false,
      error: new DomainError('WAREHOUSE_INACTIVE', `Склад ${order.warehouseId} не принимает заказы`),
    };
  }

  // Все проверки прошли — выполняем основную операцию
  const reservation = await createReservation(order);
  return { ok: true, reservationId: reservation.id };
}
```

---

### 2.4. Result / Either — функциональный подход

В функциональном программировании функция никогда не бросает исключение —
она возвращает «коробочку», описывающую два возможных исхода. В JavaScript
наиболее распространённая реализация этого паттерна — `Promise`.

```javascript
// result-pattern.js
// Реализация паттерна Result без сторонних библиотек.
//
// Promise — это встроенная реализация Either/Result в JavaScript.
// Три внутренних состояния (pending, fulfilled, rejected) выражаются
// наружу через разветвление потока: .then() / .catch().

// Простая реализация Result-объекта
const Result = {
  ok: (value) => ({ ok: true, value }),
  err: (error) => ({ ok: false, error }),
};

// Оборачиваем async-функцию, чтобы она никогда не бросала исключение
async function safeAsync(fn) {
  try {
    const value = await fn();
    return Result.ok(value);
  } catch (err) {
    return Result.err(err);
  }
}

// Пример использования
async function loadUserProfile(userId) {
  const dbResult = await safeAsync(() => db.users.findById(userId));

  if (!dbResult.ok) {
    // Операционная ошибка БД трансформируется в soft-ошибку
    return Result.err(
      new DomainError('USER_NOT_FOUND', `Пользователь ${userId} не найден`, {
        cause: dbResult.error,
      })
    );
  }

  const user = dbResult.value;

  if (!user) {
    return Result.err(
      new DomainError('USER_NOT_FOUND', `Пользователь ${userId} не существует`)
    );
  }

  return Result.ok(user);
}

// Цепочка без вложенных try/catch
async function handleProfileRequest(req, res) {
  const result = await loadUserProfile(req.params.id);

  if (!result.ok) {
    const { code } = result.error;
    const status = code === 'USER_NOT_FOUND' ? 404 : 500;
    // Сериализуем ошибку: только code и message, без stack trace
    return res.status(status).json({ error: { code, message: result.error.message } });
  }

  res.json({ user: result.value });
}
```

---

## 3. Критические секции и таймауты

При работе с пулами соединений, мьютексами, семафорами — любая критическая секция
обязана иметь таймаут. Без таймаута запрос, ожидающий ресурс из пула, зависнет навсегда,
удерживая открытый HTTP-запрос и постепенно исчерпывая все слоты Event Loop.

```javascript
// timeout-wrapper.js
// Обёртка, добавляющая таймаут к любой Promise-операции.
//
// Используется вместе с Promise.race(): первый завершившийся Promise
// (либо результат операции, либо таймаут) определяет итог.

import { setTimeout as delay } from 'node:timers/promises';

export function withTimeout(promise, ms, label = 'операция') {
  let timeoutId;

  const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => {
      reject(new Error(`Таймаут: ${label} не завершилась за ${ms} мс`));
    }, ms);
  });

  return Promise.race([promise, timeoutPromise]).finally(() => {
    clearTimeout(timeoutId);
  });
}

// Использование с пулом соединений
async function queryWithTimeout(sql, params) {
  // Получаем соединение из пула с таймаутом 5 секунд
  const connection = await withTimeout(
    pool.acquire(),
    5000,
    'получение соединения из пула'
  );

  try {
    // Выполняем запрос с таймаутом 30 секунд
    return await withTimeout(
      connection.query(sql, params),
      30_000,
      `SQL-запрос: ${sql.slice(0, 50)}`
    );
  } finally {
    // finally выполняется даже при ошибке или таймауте —
    // соединение возвращается в пул в любом случае
    pool.release(connection);
  }
}

// Современный способ через AbortSignal.timeout (Node.js 17.3+)
async function fetchWithAbort(url) {
  const signal = AbortSignal.timeout(10_000); // 10 секунд

  const response = await fetch(url, { signal });
  return response.json();
}
```

Node.js-специфика: метод `.finally()` у Promise выполняется всегда — и после
`.then()`, и после `.catch()`. Это стандартный механизм для освобождения ресурсов:
снятия блокировок, возврата соединений в пул, сброса флагов занятости.

---

## 4. Graceful Shutdown

Node.js-процесс обслуживает одновременно тысячи запросов, scheduled-задачи, таймеры,
WebSocket-соединения. При обнаружении неустранимой ошибки нельзя завершить процесс
мгновенно — это грубо обрывает все активные запросы.

Graceful shutdown — это управляемое завершение: новые запросы не принимаются,
активные успевают завершиться, ресурсы освобождаются корректно.

```javascript
// graceful-shutdown.js
// Полная реализация graceful shutdown для HTTP-сервера.
//
// Алгоритм:
// 1. Поймать сигнал завершения (SIGTERM, SIGINT)
// 2. Перестать принимать новые соединения (server.close)
// 3. Дать активным запросам время завершиться
// 4. Принудительно завершить если timeout истёк

import http from 'node:http';

const server = http.createServer(requestHandler);

// Счётчик активных запросов
let activeRequests = 0;
let isShuttingDown = false;

server.on('request', (req, res) => {
  activeRequests++;

  // Если сервер уже завершается — возвращаем 503
  if (isShuttingDown) {
    res.writeHead(503, {
      'Connection': 'close',
      'Retry-After': '30',
    });
    res.end('Сервер перезапускается, повторите запрос позже');
    activeRequests--;
    return;
  }

  res.on('finish', () => {
    activeRequests--;
  });
});

async function gracefulShutdown(signal) {
  console.log(`[${signal}] Начинаем graceful shutdown...`);
  isShuttingDown = true;

  // Шаг 1: перестаём принимать новые соединения
  await new Promise((resolve) => server.close(resolve));
  console.log('Новые соединения не принимаются');

  // Шаг 2: ждём завершения активных запросов (максимум 60 секунд)
  const SHUTDOWN_TIMEOUT_MS = 60_000;
  const startTime = Date.now();

  while (activeRequests > 0) {
    if (Date.now() - startTime > SHUTDOWN_TIMEOUT_MS) {
      console.warn(`Таймаут: ${activeRequests} запросов не завершились, принудительно останавливаем`);
      break;
    }
    console.log(`Ожидаем ${activeRequests} активных запросов...`);
    await new Promise((resolve) => setTimeout(resolve, 1000));
  }

  // Шаг 3: освобождаем ресурсы
  await db.pool.end();       // закрываем пул соединений БД
  await cache.quit();        // закрываем Redis-клиент
  console.log('Ресурсы освобождены. Процесс завершён.');

  process.exit(0);
}

// SIGTERM — стандартный сигнал завершения (Docker, Kubernetes, systemd)
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));

// SIGINT — Ctrl+C в терминале
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// uncaughtException — неперехваченное синхронное исключение
// ВАЖНО: к этому моменту состояние процесса уже неизвестно.
// Единственное безопасное действие — залогировать и завершить процесс.
process.on('uncaughtException', (err, origin) => {
  console.error('Неперехваченное исключение:', err);
  console.error('Источник:', origin);
  // Не пытаемся продолжить работу — выполняем graceful shutdown
  gracefulShutdown('uncaughtException').catch(() => process.exit(1));
});

// unhandledRejection — Promise завершился с ошибкой, у которой нет .catch()
process.on('unhandledRejection', (reason, promise) => {
  console.error('Необработанный rejection:', reason);
  // В Node.js 15+ unhandledRejection по умолчанию завершает процесс.
  // Явно логируем, чтобы не терять контекст.
});

server.listen(3000, () => {
  console.log('Сервер запущен на порту 3000');
});
```

**Когда перезапускать процесс.** Graceful shutdown применяется только при реально
неустранимом состоянии: corruption памяти, неперехваченное исключение в критическом
пути. На операционных ошибках (нет подключения к БД) — восстанавливаемся через retry,
не перезапуская процесс. Перезапуск не исправит code-ошибки: код не изменится.

---

## 5. Изоляция ошибок

Изоляция — механизм защиты одних пользователей от ошибок других. Уровни изоляции
от наиболее тяжёлого к наиболее лёгкому:

| Уровень | Механизм | Изолирует | Стоимость |
|---------|----------|-----------|-----------|
| Отдельные серверы / контейнеры | Docker, K8s | Память, CPU, Event Loop, безопасность | Высокая |
| Worker-процессы | `child_process`, `cluster` | Память, возможен перезапуск без влияния на основной процесс | Средняя |
| Worker-треды | `worker_threads` | Event Loop (CPU-задачи не вытесняют I/O) | Средняя |
| `vm.createContext()` | `node:vm` | Данные и код в рамках одного процесса | Низкая |
| Замыкания | — | Только данные | Минимальная |

```javascript
// worker-isolation.js
// Worker-тред для CPU-интенсивных задач.
//
// Зачем: если хэширование пароля (bcrypt) выполняется в основном потоке,
// оно блокирует Event Loop на ~100 мс. Все остальные пользователи
// не получают ответ всё это время.
// Worker-тред изолирует блокирующую задачу — основной Event Loop свободен.

// main.js
import { Worker } from 'node:worker_threads';
import { fileURLToPath } from 'node:url';
import path from 'node:path';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

function hashPasswordInWorker(password) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      path.join(__dirname, 'hash-worker.js'),
      { workerData: { password } }
    );

    // Таймаут: если воркер завис — не ждём вечно
    const timeout = setTimeout(() => {
      worker.terminate();
      reject(new Error('Таймаут хэширования пароля'));
    }, 10_000);

    worker.on('message', (hash) => {
      clearTimeout(timeout);
      resolve(hash);
    });

    worker.on('error', (err) => {
      clearTimeout(timeout);
      reject(err);
    });
  });
}

// hash-worker.js
import { workerData, parentPort } from 'node:worker_threads';
import bcrypt from 'bcrypt';

// Ошибка в воркере не влияет на основной процесс
try {
  const hash = await bcrypt.hash(workerData.password, 12);
  parentPort.postMessage(hash);
} catch (err) {
  // Ошибку передаём в основной поток через механизм 'error'
  throw err;
}
```

Принцип выбора уровня изоляции: применять наименее тяжёлую абстракцию, которая
решает задачу. Запускать отдельный процесс на каждый запрос оправдано только при
высоких требованиях безопасности (untrusted code execution). Для большинства
задач достаточно worker-треда или правильно построенного замыкания.

---

## 6. Логирование и консолидация ошибок

### 6.1. Сериализация ошибок для передачи по сети

Ошибки можно сериализовать в JSON и передавать по HTTP, WebSocket, SSE. При этом
действует строгое правило: **stack trace клиенту не передаётся**. Stack trace содержит
пути к файлам, имена переменных и другие сведения о внутреннем устройстве сервера.

```javascript
// error-serialization.js
// Безопасная сериализация ошибок для отправки клиенту и для логирования.
//
// Два разных представления одной ошибки:
//   - clientSafe: только то, что можно показать пользователю
//   - logEntry: полный контекст для внутреннего логирования

import { randomUUID } from 'node:crypto';

export function serializeErrorForClient(err) {
  // Никогда не передаём stack trace клиенту
  return {
    error: {
      code: err.code ?? 'INTERNAL_ERROR',
      message: err.message,
      // requestId позволяет пользователю сообщить инженерам,
      // какой именно запрос вызвал проблему
      requestId: err.requestId,
    }
  };
}

export function serializeErrorForLog(err, context = {}) {
  const entry = {
    timestamp: new Date().toISOString(),
    level: 'error',
    code: err.code ?? 'UNKNOWN',
    message: err.message,
    stack: err.stack,
    context,
  };

  // Рекурсивно сериализуем цепочку причин
  if (err.cause) {
    entry.cause = serializeErrorForLog(err.cause);
  }

  if (err instanceof AggregateError) {
    entry.errors = err.errors.map((e) => serializeErrorForLog(e));
  }

  return entry;
}

// Middleware для Express / Fastify
export function errorMiddleware(err, req, res, next) {
  const requestId = req.headers['x-request-id'] ?? randomUUID();

  // Полный лог — только внутри сервера
  console.error(JSON.stringify(serializeErrorForLog(err, {
    requestId,
    userId: req.user?.id,
    path: req.path,
    method: req.method,
  })));

  // Клиенту — только безопасная часть
  const status = err.statusCode ?? 500;
  res
    .status(status)
    .json(serializeErrorForClient({ ...err, requestId }));
}
```

---

### 6.2. Трекинг асинхронных контекстов

Задача: когда в логах появляется ошибка, нужно знать — от какого пользователя
пришёл запрос, в рамках какой сессии, какой бизнес-процесс выполнялся.

**AsyncLocalStorage** — встроенный механизм Node.js для хранения контекста
в рамках асинхронной цепочки вызовов. Он автоматически «путешествует» через
`await`, `Promise.then`, колбэки таймеров.

```javascript
// async-context.js
// AsyncLocalStorage автоматически propagate-ирует контекст
// через все асинхронные операции в рамках одного запроса.

import { AsyncLocalStorage } from 'node:async_hooks';
import { randomUUID } from 'node:crypto';

// Создаём хранилище один раз для всего приложения
export const requestContext = new AsyncLocalStorage();

// Middleware: создаём контекст для каждого запроса
export function contextMiddleware(req, res, next) {
  const context = {
    requestId: req.headers['x-request-id'] ?? randomUUID(),
    sessionId: req.session?.id,
    userId: req.user?.id,
    businessProcessId: req.headers['x-business-process-id'],
    startTime: Date.now(),
  };

  // Весь код, вызванный из next(), получит доступ к context
  requestContext.run(context, next);
}

// Где угодно в приложении — без передачи контекста через параметры
export function getLogger() {
  return {
    error(message, extra = {}) {
      const ctx = requestContext.getStore() ?? {};
      // Каждая запись лога автоматически содержит контекст запроса
      console.error(JSON.stringify({
        level: 'error',
        message,
        requestId: ctx.requestId,
        userId: ctx.userId,
        sessionId: ctx.sessionId,
        timestamp: new Date().toISOString(),
        ...extra,
      }));
    }
  };
}

// Глубоко в стеке вызовов — контекст доступен автоматически
async function processPayment(amount) {
  const logger = getLogger();
  try {
    await paymentGateway.charge(amount);
  } catch (err) {
    // requestId попадёт в лог без явной передачи
    logger.error('Ошибка платёжного шлюза', { amount, cause: err.message });
    throw err;
  }
}
```

**Важно:** `AsyncLocalStorage` — дорогая операция с точки зрения производительности.
В высоконагруженных системах применяется более эффективная альтернатива.

**Front Controller Pattern — эффективная альтернатива AsyncLocalStorage:**

```javascript
// front-controller.js
// Идентификатор запроса хранится в замыкании фронт-контроллера
// и явно пробрасывается вглубь через параметры.
// Минус: больше boilerplate. Плюс: значительно эффективнее по ресурсам.

import { randomUUID } from 'node:crypto';

async function handleRequest(req, res) {
  // Контекст живёт в замыкании этого вызова
  const ctx = {
    requestId: req.headers['x-request-id'] ?? randomUUID(),
    userId: req.user?.id,
    sessionId: req.session?.id,
  };

  try {
    // ctx явно передаётся на каждый уровень
    const order = await orderService.createOrder(req.body, ctx);
    res.json({ ok: true, orderId: order.id });
  } catch (err) {
    // Все ошибки эскалируются до фронт-контроллера
    // Здесь контекст известен — логируем с полным контекстом
    console.error(JSON.stringify({
      level: 'error',
      message: err.message,
      stack: err.stack,
      requestId: ctx.requestId,
      userId: ctx.userId,
    }));

    res.status(500).json({ error: { code: 'INTERNAL_ERROR' } });
  }
}

// orderService.js — ctx передаётся явно
async function createOrder(data, ctx) {
  const stock = await warehouse.getStock(data.productId, ctx);
  // ...
}
```

---

### 6.3. UUID для трекинга между сервисами

В микросервисной архитектуре один запрос может проходить через несколько сервисов.
Чтобы свести логи из всех сервисов воедино, каждый запрос маркируется UUID.

```javascript
// distributed-tracing.js
// Простой distributed tracing без сторонних зависимостей.
// Стандарт: W3C Trace Context (заголовок traceparent)

import { randomUUID } from 'node:crypto';

// Клиент: при обращении к другому сервису передаём trace-заголовки
async function callUserService(userId, ctx) {
  const response = await fetch(`http://user-service/users/${userId}`, {
    headers: {
      // Передаём request ID — другой сервис запишет его в свои логи
      'x-request-id': ctx.requestId,
      // Business process ID — связывает несколько запросов одного бизнес-процесса
      'x-business-process-id': ctx.businessProcessId,
      // Стандарт W3C Trace Context
      'traceparent': `00-${ctx.traceId}-${randomUUID().replace(/-/g, '')}-01`,
    },
  });

  if (!response.ok) {
    const body = await response.json().catch(() => ({}));
    throw new Error(`UserService вернул ${response.status}`, {
      cause: new Error(body.error?.message ?? 'Неизвестная ошибка'),
    });
  }

  return response.json();
}

// Сервер: при получении запроса — восстанавливаем контекст из заголовков
export function extractContext(req) {
  return {
    requestId: req.headers['x-request-id'] ?? randomUUID(),
    businessProcessId: req.headers['x-business-process-id'],
    traceId: req.headers['traceparent']?.split('-')[1] ?? randomUUID().replace(/-/g, ''),
  };
}
```

---

### 6.4. Стратегии и сроки хранения логов

Логирование имеет прямое влияние на производительность. Само по себе
(особенно синхронная запись в stdout) может стать узким местом.

```javascript
// buffered-logger.js
// Буферизованное логирование: ошибки записываются пачками, не по одной.
// Экономия на I/O: вместо N системных вызовов write() — один.

export class BufferedLogger {
  #buffer = [];
  #flushIntervalMs;
  #maxBufferSize;
  #timer = null;

  constructor({ flushIntervalMs = 1000, maxBufferSize = 100 } = {}) {
    this.#flushIntervalMs = flushIntervalMs;
    this.#maxBufferSize = maxBufferSize;
    this.#scheduleFlush();
  }

  log(entry) {
    this.#buffer.push({ ...entry, timestamp: new Date().toISOString() });

    // Принудительно сбрасываем при достижении максимального размера
    if (this.#buffer.length >= this.#maxBufferSize) {
      this.#flush();
    }
  }

  #flush() {
    if (this.#buffer.length === 0) return;

    const entries = this.#buffer.splice(0); // атомарно забираем все накопленные
    // Один системный вызов вместо N
    process.stdout.write(entries.map((e) => JSON.stringify(e)).join('\n') + '\n');
  }

  #scheduleFlush() {
    this.#timer = setInterval(() => this.#flush(), this.#flushIntervalMs);
    // unref(): таймер не препятствует завершению процесса
    this.#timer.unref();
  }

  async shutdown() {
    clearInterval(this.#timer);
    this.#flush(); // сбрасываем остаток перед завершением
  }
}
```

**Сроки хранения логов** — определяются не техническими, а правовыми и бизнес-соображениями:

| Тип ошибки / события | Срок хранения | Обоснование |
|-----------------------|---------------|-------------|
| Code errors | ~1 месяц | Актуальны до исправления и деплоя |
| Operational errors | ~2 недели | Нужны для анализа инфраструктурных сбоев |
| Soft errors (безопасность: неверный пароль, перебор ID, SQL-инъекции) | 1 год и более | Расследование инцидентов безопасности |
| Бизнес-события (транзакции, медицинские диагнозы) | 10+ лет | Нормативные требования, аудит |

---

## 7. Стратегии деплоя и управление ошибками в production

Обработка ошибок не заканчивается в коде — она продолжается в стратегиях выкатки
новых версий. Цель: минимизировать число пользователей, столкнувшихся с ошибкой
после деплоя.

### 7.1. Blue-Green Deployment

```
Инфраструктура:  [Load Balancer]
                  /             \
          [Blue cluster]    [Green cluster]
          (старая версия)   (новая версия)
```

Трафик переключается мгновенно между кластерами. При проблемах — переключаем
обратно, не трогая код.

```javascript
// feature-flags.js
// Feature Flags: код задеплоен, но функция выключена.
// Включаем для 1% пользователей, смотрим на ошибки — потом для 100%.
// Не требует перезапуска сервера.

const flags = new Map([
  ['new-checkout-flow', { enabled: false, rolloutPercent: 0 }],
  ['ai-recommendations', { enabled: true, rolloutPercent: 10 }],
]);

export function isFeatureEnabled(featureName, userId) {
  const flag = flags.get(featureName);
  if (!flag || !flag.enabled) return false;

  if (flag.rolloutPercent >= 100) return true;

  // Детерминированное решение: один пользователь всегда видит одну версию
  const hash = simpleHash(featureName + userId);
  return (hash % 100) < flag.rolloutPercent;
}

function simpleHash(str) {
  let hash = 0;
  for (const char of str) hash = (hash * 31 + char.charCodeAt(0)) & 0xffffffff;
  return Math.abs(hash);
}

// Использование
async function handleCheckout(req, res) {
  if (isFeatureEnabled('new-checkout-flow', req.user.id)) {
    return newCheckoutHandler(req, res);
  }
  return legacyCheckoutHandler(req, res);
}
```

### 7.2. Canary Releases

Часть пользователей переводится на новую версию. Ошибки у этой группы не влияют
на остальных. Решение об откате принимается автоматически (по метрикам ошибок)
или вручную.

### 7.3. Hot Fix и Rollback

```javascript
// hot-reload.js
// Hot fix: перезагружаем один изменённый модуль без перезапуска сервера.
// Используется для быстрого исправления в production.
//
// ВАЖНО: только для CommonJS (require). ESM-модули не поддерживают
// удаление из кэша стандартными средствами.

import { watch } from 'node:fs';
import { createRequire } from 'node:module';
import path from 'node:path';

const require = createRequire(import.meta.url);

// Отслеживаем изменения в директории модулей
watch('./modules', { recursive: true }, (eventType, filename) => {
  if (!filename?.endsWith('.cjs')) return;

  const modulePath = path.resolve('./modules', filename);

  console.log(`Обнаружено изменение: ${filename}. Перезагружаем модуль...`);

  try {
    // Удаляем модуль из кэша require
    delete require.cache[modulePath];
    // При следующем вызове require модуль загрузится заново
    const freshModule = require(modulePath);
    moduleRegistry.set(filename, freshModule);
    console.log(`Модуль ${filename} успешно перезагружен`);
  } catch (err) {
    console.error(`Ошибка перезагрузки ${filename}:`, err.message);
    // Оставляем старую версию модуля в registry — rollback автоматический
  }
});
```

---

## Итоги

### Ключевые принципы

1. **Классифицируйте ошибки до написания кода.** Code errors — бросать (`throw`) немедленно.
   Soft errors — возвращать через `Result`. Operational errors — применять retry с backoff.

2. **Не теряйте контекст.** Используйте `error.cause` для выстраивания цепочки причин.
   Без этого лог покажет только симптом, а не первопричину.

3. **Один класс ошибки с кодом** вместо 50 пустых подклассов. Код ошибки — строка,
   как в Node.js core API (`err.code`).

4. **Таймаут у каждой критической секции.** Пул соединений, мьютекс, внешний HTTP-запрос —
   всё должно иметь таймаут. `Promise.finally()` — для гарантированного освобождения ресурсов.

5. **Graceful shutdown, а не аварийное завершение.** `process.on('SIGTERM')` +
   дождаться завершения активных запросов + освободить ресурсы.

6. **Stack trace — только внутри сервера.** Клиент получает только `code` и `message`.
   Логи с полным stack trace и контекстом — только во внутренние системы.

7. **UUID для трекинга запросов** между сервисами. Один бизнес-процесс — один UUID,
   по которому логи из нескольких сервисов сводятся воедино.

8. **Выбирайте уровень изоляции под задачу.** Worker-тред для CPU-задач. Отдельный
   процесс только при высоких требованиях безопасности. Замыкание — когда достаточно
   изоляции данных.

### Производительность

- `throw` исторически подавлял JIT-оптимизации в V8; современные версии значительно
  улучшили ситуацию, но бросать исключения в hot path всё равно не следует.
- `AsyncLocalStorage` — удобен, но дорог по CPU. В high-throughput системах
  предпочтительнее явная передача контекста (front controller pattern).
- Буферизованное логирование снижает нагрузку на I/O: пишем пачками, не построчно.

### Сроки хранения логов

- Code errors: ~1 месяц
- Operational errors: ~2 недели
- Security events: 1+ год
- Business events: 10+ лет

---
