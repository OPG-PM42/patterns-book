# AsyncPool — Паттерн асинхронный пул объектов

## Обзор

AsyncPool — это паттерн проектирования, представляющий собой **пул объектов** (Object Pool), адаптированный для работы в асинхронной среде Node.js. Он предназначен для управления ограниченным количеством экземпляров объектов (инстансов) и предоставления механизма ожидания освобождения ресурсов при их исчерпании.

### Основные задачи паттерна

- **Хранение инстансов** — поддержание коллекции переиспользуемых объектов
- **Управление доступностью** — отслеживание свободных и занятых экземпляров
- **Асинхронное ожидание** — предоставление механизма ожидания через Promise, когда свободные инстансы отсутствуют
- **Автоматическая инстанциация** — создание необходимого количества объектов через фабрику

---

## Зачем нужен асинхронный пул?

В современных Node.js приложениях часто возникает необходимость ограничить количество одновременно используемых ресурсов:

- **Подключения к базе данных** — ограниченное количество соединений
- **HTTP-соединения** — контроль одновременных запросов
- **Тяжелые объекты** — переиспользование дорогостоящих в создании экземпляров
- **Управление памятью** — предотвращение утечек при массовых операциях

**Ключевое отличие от синхронного пула**: вместо блокирования или колбеков используется **Promise-контракт** для ожидания освобождения ресурсов.

---

## Архитектура AsyncPool

### Основные компоненты

```javascript
class AsyncPool {
  // Коллекция хранимых инстансов
  instances = [];

  // Флаги доступности: true = свободен, false = занят
  free = [];

  // Очередь ожидающих Promise
  queue = [];

  // Указатель на текущий инстанс для выдачи
  current = 0;

  // Размер пула
  size = 0;

  // Счетчик доступных (свободных) инстансов
  available = 0;
}
```

### Диаграмма состояний

```
┌─────────────────────────────────────┐
│      AsyncPool (size: 10)           │
├─────────────────────────────────────┤
│ instances: [obj0...obj9]            │
│ free:      [T,T,F,F,T,T,T,T,T,T]    │
│ available: 8                        │
│ queue:     [{resolve, timer}, ...]  │
└─────────────────────────────────────┘
         ↑                    ↓
    release()            getInstance()
```

---

## Реализация базового AsyncPool

### 1. Конструктор и инициализация

```javascript
class AsyncPool {
  constructor(factory, size) {
    this.size = size;
    this.available = size;
    this.current = 0;
    this.instances = new Array(size);
    this.free = new Array(size);
    this.queue = [];

    // Инициализация флагов "свободен"
    for (let i = 0; i < size; i++) {
      this.free[i] = true;
      // Вызов фабрики для создания инстансов
      this.instances[i] = factory(i);
    }
  }
}
```

**Ключевые моменты:**

- **Фабрика вместо готовых объектов** — пул сам создает нужное количество инстансов через переданную функцию-фабрику
- **Размер фиксирован** — массивы инициализируются сразу на полный размер
- **Счетчик available** — позволяет быстро проверить наличие свободных элементов без перебора массива

### 2. Метод получения инстанса (getInstance)

```javascript
async getInstance() {
  // Если нет свободных — ставим Promise в очередь
  if (this.available === 0) {
    return new Promise((resolve) => {
      this.queue.push(resolve);
    });
  }

  // Ищем следующий свободный инстанс
  return this.next();
}
```

**Логика работы:**

1. **Проверка доступности** — если `available === 0`, создается промис и его `resolve` добавляется в очередь
2. **Асинхронное ожидание** — вызывающий код получит инстанс только когда кто-то освободит ресурс
3. **Немедленная выдача** — если есть свободные, вызывается `next()` для поиска инстанса

### 3. Поиск следующего свободного элемента (next)

```javascript
next() {
  let instance = null;
  let free = false;

  // Цикл поиска свободного инстанса
  while (!free) {
    instance = this.instances[this.current];
    free = this.free[this.current];

    // Циклическое перемещение указателя
    this.current++;
    if (this.current >= this.size) {
      this.current = 0;
    }

    // Если нашли свободный — помечаем занятым и возвращаем
    if (instance && free) {
      this.free[this.current] = false;
      this.available--;
      return instance;
    }
  }
}
```

**Особенности реализации:**

- **Циклический обход** — указатель `current` двигается по кругу (Round-Robin)
- **Гарантия завершения** — цикл безопасен, так как перед вызовом `next()` проверяется `available > 0`
- **Атомарное изменение состояния** — флаг `free` меняется сразу, счетчик уменьшается

### 4. Освобождение инстанса (release)

```javascript
release(instance) {
  // Проверка: существует ли инстанс в пуле
  const index = this.instances.indexOf(instance);
  if (index === -1) {
    throw new Error('Attempting to release an instance not managed by this pool');
  }

  // Проверка: не пытаются ли вернуть уже свободный инстанс
  if (this.free[index]) {
    throw new Error('Attempting to release an already free instance');
  }

  // Если есть ожидающие в очереди — отдаем им напрямую
  if (this.queue.length > 0) {
    const resolve = this.queue.shift();

    // Отправляем инстанс в промис через setImmediate
    setImmediate(() => {
      resolve(instance);
    });

    // Инстанс остается занятым (переходит к новому владельцу)
    return;
  }

  // Если очередь пуста — помечаем свободным
  this.available++;
  this.free[index] = true;
}
```

**Важные детали:**

1. **Валидация инстанса** — защита от попыток вернуть "чужие" объекты
2. **Защита от двойного освобождения** — предотвращение багов с сохраненными ссылками
3. **Прямая передача ожидающим** — если есть очередь, инстанс сразу переходит следующему потребителю
4. **setImmediate для resolve** — передача кванта времени в event loop, избегание синхронного выполнения

---

## Расширенная версия с таймаутом

### Зачем нужен таймаут?

В реальных приложениях нельзя бесконечно ждать освобождения ресурса:

- **Предотвращение зависаний** — защита от deadlock-ситуаций
- **Контроль времени отклика** — ограничение максимального времени ожидания
- **Graceful degradation** — возможность обработать ошибку и применить альтернативную стратегию

### Модификация конструктора

```javascript
class AsyncPool {
  constructor(factory, options) {
    this.size = options.size;
    this.timeout = options.timeout; // Таймаут в миллисекундах
    this.available = this.size;
    this.current = 0;
    this.instances = new Array(this.size);
    this.free = new Array(this.size);
    this.queue = [];

    for (let i = 0; i < this.size; i++) {
      this.free[i] = true;
      this.instances[i] = factory(i);
    }
  }
}
```

**Изменения:**

- Параметры теперь передаются через объект `options`
- Добавлено поле `timeout` для хранения лимита ожидания

### Модифицированный getInstance с таймером

```javascript
async getInstance() {
  if (this.available === 0) {
    return new Promise((resolve, reject) => {
      // Создаем таймер для отклонения промиса
      const timer = setTimeout(() => {
        reject(new Error('Pool timeout exceeded'));
      }, this.timeout);

      // В очередь сохраняем resolve и timer
      this.queue.push({ resolve, timer });
    });
  }

  return this.next();
}
```

**Механизм работы:**

1. **Создание таймера** — `setTimeout` вызовет `reject` через заданное время
2. **Структура в очереди** — сохраняется и `resolve`, и `timer` для последующей очистки
3. **Отклонение промиса** — если таймаут достигнут, вызывающий код получит ошибку

### Модифицированный release

```javascript
release(instance) {
  const index = this.instances.indexOf(instance);
  if (index === -1) {
    throw new Error('Attempting to release an instance not managed by this pool');
  }

  if (this.free[index]) {
    throw new Error('Attempting to release an already free instance');
  }

  // Если есть ожидающие — отдаем им
  if (this.queue.length > 0) {
    const { resolve, timer } = this.queue.shift();

    // ВАЖНО: очищаем таймер, так как ожидание завершилось успешно
    clearTimeout(timer);

    setImmediate(() => {
      resolve(instance);
    });

    return;
  }

  // Помечаем свободным
  this.available++;
  this.free[index] = true;
}
```

**Критически важно:**

- **clearTimeout(timer)** — предотвращение срабатывания таймера после успешного получения инстанса
- **Извлечение из очереди** — `shift()` удаляет элемент, освобождая память

---

## Альтернатива: использование AbortSignal

### Современный подход к отмене операций

AbortSignal — это стандартный Web API для отмены асинхронных операций:

```javascript
async getInstance() {
  if (this.available === 0) {
    return new Promise((resolve, reject) => {
      // Создаем AbortSignal с таймаутом
      const signal = AbortSignal.timeout(this.timeout);

      // Обработчик отмены
      const listener = () => {
        // Ошибка хранится в signal.reason
        reject(signal.reason);
      };

      // Подписываемся на событие abort
      signal.addEventListener('abort', listener);

      // Сохраняем в очередь все три компонента
      this.queue.push({ resolve, signal, listener });
    });
  }

  return this.next();
}
```

### Очистка при успешном получении

```javascript
release(instance) {
  const index = this.instances.indexOf(instance);
  if (index === -1) {
    throw new Error('Attempting to release an instance not managed by this pool');
  }

  if (this.free[index]) {
    throw new Error('Attempting to release an already free instance');
  }

  if (this.queue.length > 0) {
    const { resolve, signal, listener } = this.queue.shift();

    // Удаляем обработчик события
    signal.removeEventListener('abort', listener);

    setImmediate(() => {
      resolve(instance);
    });

    return;
  }

  this.available++;
  this.free[index] = true;
}
```

**Преимущества AbortSignal:**

- **Стандартизация** — единый API для отмены операций
- **Автоматическое управление** — таймаут встроен в сигнал
- **Garbage Collection** — после `removeEventListener` и удаления из очереди все объекты освобождаются

**Недостатки:**

- **Необходимость хранить listener** — EventTarget требует точную ссылку на функцию для удаления
- **Более сложная структура** — три компонента вместо двух в варианте с setTimeout

---

## Практические примеры использования

### Пример 1: Фабрика соединений

```javascript
// Фабрика создает объекты соединений
const connectionFactory = (index) => {
  return {
    id: `connection-${index}`,
    query: async (sql) => {
      console.log(`[${this.id}] Executing: ${sql}`);
      // Симуляция запроса к БД
      return new Promise((resolve) => {
        setTimeout(() => resolve({ rows: [] }), 100);
      });
    },
    close: () => {
      console.log(`[${this.id}] Closed`);
    }
  };
};

// Создание пула с 10 соединениями
const pool = new AsyncPool(connectionFactory, { size: 10, timeout: 3000 });
```

### Пример 2: Базовое использование без таймаута

```javascript
const pool = new AsyncPool(connectionFactory, 10);
const connections = [];

// Запрашиваем 12 инстансов при размере пула 10
for (let i = 0; i < 12; i++) {
  (async () => {
    try {
      const conn = await pool.getInstance();
      console.log(`Got instance: ${conn.id}`);

      // Первые два сохраняем для позднего освобождения
      if (i < 2) {
        connections.push(conn);
      } else {
        // Остальные сразу возвращаем
        pool.release(conn);
      }
    } catch (error) {
      console.error(`Error: ${error.message}`);
    }
  })();
}

// Через 2 секунды освобождаем первый
setTimeout(() => {
  console.log('Releasing connection 0');
  pool.release(connections[0]);
}, 2000);

// Через 5 секунд освобождаем второй
setTimeout(() => {
  console.log('Releasing connection 1');
  pool.release(connections[1]);
}, 5000);
```

**Вывод программы:**

```
Got instance: connection-0
Got instance: connection-1
Got instance: connection-2
Got instance: connection-3
Got instance: connection-4
Got instance: connection-5
Got instance: connection-6
Got instance: connection-7
Got instance: connection-8
Got instance: connection-9
Releasing connection 0
Got instance: connection-0   // 11-й запрос получил освободившийся инстанс
Releasing connection 1
Got instance: connection-1   // 12-й запрос получил освободившийся инстанс
```

**Объяснение:**

1. Первые 10 запросов сразу получают инстансы (пул заполнен)
2. Запросы 11 и 12 становятся в очередь (`available === 0`)
3. Через 2 секунды освобождается `connection-0`, он сразу передается 11-му запросу
4. Через 5 секунд освобождается `connection-1`, он передается 12-му запросу

### Пример 3: Работа с таймаутом

```javascript
const pool = new AsyncPool(connectionFactory, {
  size: 10,
  timeout: 3000  // Максимум 3 секунды ожидания
});

const connections = [];

for (let i = 0; i < 12; i++) {
  (async () => {
    try {
      const conn = await pool.getInstance();
      console.log(`[${i}] Got instance: ${conn.id}`);

      if (i < 2) {
        connections.push(conn);
      } else {
        pool.release(conn);
      }
    } catch (error) {
      console.error(`[${i}] Error: ${error.message}`);
    }
  })();
}

// Освобождаем через 4 секунды (позже таймаута!)
setTimeout(() => {
  console.log('Releasing connection 0');
  pool.release(connections[0]);
}, 4000);

setTimeout(() => {
  console.log('Releasing connection 1');
  pool.release(connections[1]);
}, 5000);
```

**Вывод программы:**

```
[0] Got instance: connection-0
[1] Got instance: connection-1
[2] Got instance: connection-2
[3] Got instance: connection-3
[4] Got instance: connection-4
[5] Got instance: connection-5
[6] Got instance: connection-6
[7] Got instance: connection-7
[8] Got instance: connection-8
[9] Got instance: connection-9
[10] Error: Pool timeout exceeded     // Таймаут через 3 секунды
Releasing connection 0                 // Освобождение через 4 секунды
[11] Got instance: connection-0        // 12-й запрос дождался
Releasing connection 1
```

**Анализ:**

- Запрос 11 (индекс 10) ждет 3 секунды → таймаут → получает ошибку
- Запрос 12 (индекс 11) ждет 3 секунды → все еще в очереди → через 4 секунды получает инстанс

---

## Расширенные возможности пула

### 1. Динамическое расширение пула

Можно модифицировать пул для создания дополнительных инстансов сверх начального размера:

```javascript
class DynamicAsyncPool extends AsyncPool {
  constructor(factory, options) {
    super(factory, options);
    this.maxSize = options.maxSize || this.size * 2;
    this.factory = factory;
  }

  async getInstance() {
    // Если нет свободных, но не достигнут maxSize — создаем новый
    if (this.available === 0 && this.instances.length < this.maxSize) {
      const newInstance = this.factory(this.instances.length);
      this.instances.push(newInstance);
      this.free.push(false);
      this.size++;
      return newInstance;
    }

    return super.getInstance();
  }

  release(instance) {
    const index = this.instances.indexOf(instance);

    // Если инстанс сверх начального размера и очередь пуста — удаляем
    if (index >= options.size && this.queue.length === 0) {
      this.instances.splice(index, 1);
      this.free.splice(index, 1);
      this.size--;

      // Вызываем cleanup если есть
      if (instance.close) {
        instance.close();
      }

      return;
    }

    super.release(instance);
  }
}
```

**Преимущества:**

- **Адаптивность** — пул растет под нагрузкой
- **Экономия ресурсов** — лишние инстансы удаляются при снижении нагрузки
- **Ограничение роста** — параметр `maxSize` предотвращает бесконтрольное потребление памяти

### 2. Добавление метода add для внешних инстансов

```javascript
add(instance) {
  // Проверяем, что инстанс не в пуле
  if (this.instances.indexOf(instance) !== -1) {
    throw new Error('Instance already in pool');
  }

  // Если есть ожидающие — сразу отдаем им
  if (this.queue.length > 0) {
    const { resolve, timer } = this.queue.shift();
    clearTimeout(timer);

    setImmediate(() => {
      resolve(instance);
    });

    return;
  }

  // Добавляем в пул
  this.instances.push(instance);
  this.free.push(true);
  this.size++;
  this.available++;
}
```

**Применение:**

- Возможность добавлять инстансы извне
- Гибкость в управлении размером пула
- Интеграция с внешними источниками объектов

---

## Ключевые концепции и термины

### Round-Robin (циклический обход)

**Определение:** Алгоритм равномерного распределения нагрузки, при котором указатель циклически перемещается по коллекции.

**Применение в AsyncPool:**
```javascript
this.current++;
if (this.current >= this.size) {
  this.current = 0;  // Возврат к началу
}
```

**Преимущества:**
- Равномерное использование всех инстансов
- Предотвращение "голодания" отдельных объектов
- Простота реализации

### Promise-based контракт

**Определение:** Использование Promise как механизма асинхронного ожидания освобождения ресурса.

**Сравнение с колбеками:**

```javascript
// Колбек-стиль (устаревший)
pool.getInstance((err, instance) => {
  if (err) return console.error(err);
  // Использование instance
});

// Promise-стиль (современный)
try {
  const instance = await pool.getInstance();
  // Использование instance
} catch (error) {
  console.error(error);
}
```

**Преимущества Promise:**
- Поддержка async/await синтаксиса
- Естественная обработка ошибок через try/catch
- Возможность композиции (Promise.all, Promise.race)

### Event Loop и setImmediate

**Зачем использовать setImmediate?**

```javascript
// БЕЗ setImmediate — синхронное выполнение
resolve(instance);
// resolve может вызвать длинную цепочку синхронного кода

// С setImmediate — передача кванта времени
setImmediate(() => {
  resolve(instance);
});
// Другие операции в event loop получат шанс выполниться
```

**Фазы Event Loop:**
```
   ┌───────────────────────────┐
┌─>│           timers          │ — setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │ — системные колбеки
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │ — внутренние операции
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │ — setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │ — socket.on('close', ...)
   └───────────────────────────┘
```

setImmediate выполняется в фазе **check**, позволяя другим операциям завершиться.

---

## Распространенные ошибки и их предотвращение

### Ошибка 1: Забыли вызвать release

```javascript
// НЕПРАВИЛЬНО
const conn = await pool.getInstance();
await conn.query('SELECT * FROM users');
// Забыли вернуть в пул — утечка ресурсов!

// ПРАВИЛЬНО
const conn = await pool.getInstance();
try {
  await conn.query('SELECT * FROM users');
} finally {
  pool.release(conn);  // Гарантированное освобождение
}
```

**Решение:** Всегда использовать try/finally или обернуть в специальный метод:

```javascript
async withInstance(callback) {
  const instance = await this.getInstance();
  try {
    return await callback(instance);
  } finally {
    this.release(instance);
  }
}

// Использование
await pool.withInstance(async (conn) => {
  return await conn.query('SELECT * FROM users');
});
```

### Ошибка 2: Сохранение ссылки и повторное освобождение

```javascript
// НЕПРАВИЛЬНО
const conn = await pool.getInstance();
const savedRef = conn;  // Сохранили ссылку

pool.release(conn);
// ... позже ...
pool.release(savedRef);  // Error: already free instance
```

**Защита в коде:**
```javascript
if (this.free[index]) {
  throw new Error('Attempting to release an already free instance');
}
```

### Ошибка 3: Возврат "чужого" инстанса

```javascript
// НЕПРАВИЛЬНО
const externalConn = createConnection();
pool.release(externalConn);  // Error: not managed by this pool
```

**Защита в коде:**
```javascript
const index = this.instances.indexOf(instance);
if (index === -1) {
  throw new Error('Attempting to release an instance not managed by this pool');
}
```

---

## Оптимизация и лучшие практики

### 1. Выбор правильного размера пула

```javascript
// Формула для пула соединений к БД:
// connections = ((core_count * 2) + effective_spindle_count)

const cpuCount = require('os').cpus().length;
const poolSize = (cpuCount * 2) + 1;

const pool = new AsyncPool(dbConnectionFactory, {
  size: poolSize,
  timeout: 5000
});
```

### 2. Мониторинг состояния пула

```javascript
class MonitoredAsyncPool extends AsyncPool {
  getStats() {
    return {
      size: this.size,
      available: this.available,
      inUse: this.size - this.available,
      queueLength: this.queue.length,
      utilizationPercent: ((this.size - this.available) / this.size * 100).toFixed(2)
    };
  }

  logStats() {
    const stats = this.getStats();
    console.log(`Pool: ${stats.inUse}/${stats.size} in use, ${stats.queueLength} waiting, ${stats.utilizationPercent}% utilization`);
  }
}
```

### 3. Graceful shutdown

```javascript
class GracefulAsyncPool extends AsyncPool {
  async drain() {
    // Ждем освобождения всех инстансов
    while (this.available < this.size) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }

    // Закрываем все инстансы
    for (const instance of this.instances) {
      if (instance.close) {
        await instance.close();
      }
    }

    this.instances = [];
    this.free = [];
    this.size = 0;
    this.available = 0;
  }
}

// Использование
process.on('SIGTERM', async () => {
  console.log('Draining pool...');
  await pool.drain();
  process.exit(0);
});
```

---

## Сравнение с другими паттернами

### AsyncPool vs Semaphore

```javascript
// Семафор — ограничивает количество одновременных операций
class Semaphore {
  constructor(max) {
    this.max = max;
    this.current = 0;
    this.queue = [];
  }

  async acquire() {
    if (this.current < this.max) {
      this.current++;
      return;
    }

    await new Promise(resolve => this.queue.push(resolve));
  }

  release() {
    if (this.queue.length > 0) {
      const resolve = this.queue.shift();
      resolve();
    } else {
      this.current--;
    }
  }
}

// AsyncPool — управляет конкретными экземплярами объектов
```

**Различия:**
- **Семафор** — абстрактный счетчик, не хранит объекты
- **AsyncPool** — управляет жизненным циклом конкретных инстансов

### AsyncPool vs Worker Pool

```javascript
// Worker Pool — пул потоков для выполнения задач
const { Worker } = require('worker_threads');

class WorkerPool {
  constructor(workerScript, poolSize) {
    this.workers = [];
    for (let i = 0; i < poolSize; i++) {
      this.workers.push(new Worker(workerScript));
    }
  }

  async exec(data) {
    const worker = await this.getAvailableWorker();
    return new Promise((resolve, reject) => {
      worker.once('message', resolve);
      worker.once('error', reject);
      worker.postMessage(data);
    });
  }
}
```

**Различия:**
- **Worker Pool** — специфичен для параллельных вычислений
- **AsyncPool** — универсальный паттерн для любых объектов

---

## Интеграция с реальными библиотеками

### Пример: Пул подключений к PostgreSQL

```javascript
const { Client } = require('pg');

const pgConnectionFactory = (index) => {
  const client = new Client({
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    user: 'user',
    password: 'password'
  });

  // Асинхронное подключение
  client.connect();

  return client;
};

const dbPool = new AsyncPool(pgConnectionFactory, {
  size: 20,
  timeout: 10000
});

// Использование
async function getUser(id) {
  const client = await dbPool.getInstance();
  try {
    const result = await client.query('SELECT * FROM users WHERE id = $1', [id]);
    return result.rows[0];
  } finally {
    dbPool.release(client);
  }
}
```

### Пример: Пул HTTP-агентов

```javascript
const https = require('https');

const httpsAgentFactory = () => {
  return new https.Agent({
    keepAlive: true,
    maxSockets: 1,  // Один сокет на агент
    timeout: 5000
  });
};

const agentPool = new AsyncPool(httpsAgentFactory, {
  size: 50,  // 50 одновременных соединений
  timeout: 3000
});

async function fetchData(url) {
  const agent = await agentPool.getInstance();
  try {
    return await new Promise((resolve, reject) => {
      https.get(url, { agent }, (res) => {
        let data = '';
        res.on('data', chunk => data += chunk);
        res.on('end', () => resolve(data));
      }).on('error', reject);
    });
  } finally {
    agentPool.release(agent);
  }
}
```

---

## Тестирование AsyncPool

### Базовый тест функциональности

```javascript
const assert = require('assert');

describe('AsyncPool', () => {
  it('should create instances via factory', async () => {
    let factoryCallCount = 0;
    const factory = () => ({ id: factoryCallCount++ });

    const pool = new AsyncPool(factory, 5);

    assert.strictEqual(pool.size, 5);
    assert.strictEqual(pool.available, 5);
    assert.strictEqual(factoryCallCount, 5);
  });

  it('should provide instances immediately when available', async () => {
    const factory = (i) => ({ id: i });
    const pool = new AsyncPool(factory, 3);

    const inst1 = await pool.getInstance();
    const inst2 = await pool.getInstance();

    assert.strictEqual(pool.available, 1);
    assert.ok(inst1.id !== inst2.id);
  });

  it('should queue requests when pool exhausted', async () => {
    const factory = (i) => ({ id: i });
    const pool = new AsyncPool(factory, 2);

    const inst1 = await pool.getInstance();
    const inst2 = await pool.getInstance();

    let inst3Resolved = false;
    const inst3Promise = pool.getInstance().then(inst => {
      inst3Resolved = true;
      return inst;
    });

    // inst3 должен ждать
    await new Promise(resolve => setImmediate(resolve));
    assert.strictEqual(inst3Resolved, false);

    // Освобождаем inst1
    pool.release(inst1);

    const inst3 = await inst3Promise;
    assert.strictEqual(inst3Resolved, true);
    assert.strictEqual(inst3.id, inst1.id);  // Получили тот же инстанс
  });

  it('should reject on timeout', async () => {
    const factory = (i) => ({ id: i });
    const pool = new AsyncPool(factory, { size: 1, timeout: 100 });

    const inst1 = await pool.getInstance();

    // Второй запрос должен упасть по таймауту
    await assert.rejects(
      pool.getInstance(),
      /Pool timeout exceeded/
    );
  });

  it('should throw when releasing foreign instance', async () => {
    const factory = (i) => ({ id: i });
    const pool = new AsyncPool(factory, 2);

    const foreignInstance = { id: 999 };

    assert.throws(
      () => pool.release(foreignInstance),
      /not managed by this pool/
    );
  });

  it('should throw when releasing already free instance', async () => {
    const factory = (i) => ({ id: i });
    const pool = new AsyncPool(factory, 2);

    const inst = await pool.getInstance();
    pool.release(inst);

    assert.throws(
      () => pool.release(inst),
      /already free instance/
    );
  });
});
```

### Нагрузочный тест

```javascript
async function stressTest() {
  const factory = (i) => ({
    id: i,
    async work() {
      await new Promise(resolve => setTimeout(resolve, Math.random() * 100));
    }
  });

  const pool = new AsyncPool(factory, { size: 10, timeout: 5000 });
  const operations = [];

  // 1000 операций
  for (let i = 0; i < 1000; i++) {
    operations.push((async () => {
      const inst = await pool.getInstance();
      try {
        await inst.work();
      } finally {
        pool.release(inst);
      }
    })());
  }

  const start = Date.now();
  await Promise.all(operations);
  const duration = Date.now() - start;

  console.log(`Completed 1000 operations in ${duration}ms`);
  console.log(`Average: ${(duration / 1000).toFixed(2)}ms per operation`);
}
```

---

## Резюме

### Основные преимущества AsyncPool

✅ **Ограничение ресурсов** — контроль максимального количества одновременно используемых объектов

✅ **Переиспользование** — создание дорогостоящих объектов один раз, многократное использование

✅ **Асинхронное ожидание** — элегантный Promise-контракт вместо блокировки или колбеков

✅ **Защита от ошибок** — валидация возвращаемых инстансов, предотвращение двойного освобождения

✅ **Гибкость** — поддержка таймаутов, расширения пула, мониторинга

### Когда использовать

🎯 **Подключения к базам данных** — ограниченное количество соединений

🎯 **HTTP-клиенты** — контроль одновременных запросов к внешним API

🎯 **Файловые дескрипторы** — ограничение открытых файлов

🎯 **Тяжелые объекты** — парсеры, компиляторы, криптографические контексты

🎯 **Worker threads** — пул потоков для параллельных вычислений

### Когда НЕ использовать

❌ **Легковесные объекты** — накладные расходы пула перевесят выгоду

❌ **Одноразовые объекты** — если объект используется единожды

❌ **Объекты с состоянием** — если сброс состояния невозможен или сложен

❌ **Малая нагрузка** — при редких запросах пул не даст выигрыша

---

## Дополнительные материалы

### Ссылки на исходный код

Продакшн-версия асинхронного пула доступна в репозитории **Metarhia**:
- [AsyncPool в Metarhia](https://github.com/metarhia/metautil)

Демонстрационные примеры из лекции:
- Базовая версия AsyncPool
- AsyncPool с таймаутом через setTimeout
- AsyncPool с AbortSignal

### Рекомендации по углублению

1. **Изучите реализации в популярных библиотеках:**
   - `pg-pool` для PostgreSQL
   - `generic-pool` — универсальная библиотека пулов
   - `tarn` — современная альтернатива с TypeScript

2. **Экспериментируйте с кодом:**
   - Напишите собственную реализацию с нуля
   - Добавьте метрики и мониторинг
   - Реализуйте стратегии вытеснения (LIFO, FIFO, LRU)

3. **Примените на практике:**
   - Создайте пул для вашего проекта
   - Проведите нагрузочное тестирование
   - Сравните производительность с/без пула

---

## Заключение

AsyncPool — это фундаментальный паттерн для эффективного управления ресурсами в асинхронных Node.js приложениях. Понимание его внутреннего устройства позволяет:

- Писать более эффективный код
- Избегать распространенных ошибок с ресурсами
- Оптимизировать производительность приложений
- Проектировать масштабируемые системы

Практикуйтесь в реализации этого паттерна, экспериментируйте с различными стратегиями и адаптируйте его под конкретные задачи вашего проекта.

**Успехов в изучении паттернов асинхронного программирования!** 🚀
