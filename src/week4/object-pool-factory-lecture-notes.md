# Паттерны инстанцирования: Пул объектов и Фабрика

## Обзор

Эта лекция посвящена изучению двух фундаментальных паттернов создания объектов в JavaScript/TypeScript:
- **Factory (Фабрика)** - паттерн для абстрагирования процесса создания объектов
- **Object Pool (Пул объектов)** - паттерн для переиспользования объектов и оптимизации памяти

Оба паттерна являются ключевыми инструментами для эффективного управления инстанцированием объектов в современных JavaScript-приложениях.

---

## 1. Инстанцирование через конструктор

### Базовый подход

Самый простой способ создания объектов - использование конструктора класса через оператор `new`:

```javascript
// Простой класс
class Connection {
  constructor(index) {
    this.url = `https://example.com/connection-${index}`;
  }
}

// Создание экземпляра
const conn = new Connection(1);
```

**Ключевые моменты:**
- Конструктор вызывается через оператор `new`
- Каждый вызов создает новый экземпляр в памяти
- Прямое инстанцирование связывает код с конкретным классом

---

## 2. Паттерн Factory (Фабрика)

### Определение

**Фабрика** - это любая функция или метод, который возвращает экземпляр объекта. В JavaScript фабрики более гибкие, чем в других языках, благодаря динамической природе языка.

### Варианты реализации фабрик

#### 2.1 Фабрика как обычная функция

```javascript
// Простая функция-фабрика, возвращающая литерал объекта
function createUser(name, age) {
  return {
    name,
    age,
    greet() {
      console.log(`Hello, I'm ${this.name}`);
    }
  };
}

const user = createUser('Alice', 30);
```

**Преимущества:**
- Не требует использования `new`
- Может возвращать любой тип объекта
- Скрывает детали создания от клиента

#### 2.2 Статический метод класса

```javascript
class Connection {
  static create(index) {
    return new Connection(index);
  }

  constructor(index) {
    this.url = `https://example.com/connection-${index}`;
  }
}

// Использование
const conn = Connection.create(1);
```

**Преимущества:**
- Явное указание на фабричный метод
- Логически связан с классом
- Может содержать дополнительную логику инициализации

#### 2.3 Универсальная фабрика высшего порядка

Можно создать универсальную функцию, которая превращает любой класс в фабрику:

```javascript
// Функция factify - превращает класс в фабрику
const factify = (Class) => (...args) => new Class(...args);

// Применение
class Connection {
  constructor(index) {
    this.url = `https://example.com/connection-${index}`;
  }
}

const createConnection = factify(Connection);

// Использование
const conn1 = createConnection(1);
const conn2 = createConnection(2);
```

**Ключевые особенности:**
- `factify` принимает класс и возвращает функцию
- Возвращаемая функция создает экземпляры без явного использования `new`
- Универсальное решение для любого класса
- Класс передается в замыкание

### 2.4 Фабрика с состоянием (на основе замыкания)

```javascript
// Альтернативная фабрика с использованием замыкания
const alternativeFactory = (() => {
  let index = 0; // Счетчик в замыкании

  return () => {
    return new Connection(index++);
  };
})();

// Использование
const conn1 = alternativeFactory(); // Connection с index = 0
const conn2 = alternativeFactory(); // Connection с index = 1
```

**Особенности:**
- Состояние (`index`) скрыто в замыкании
- Автоматический инкремент счетчика
- Безопасное хранение состояния

#### 2.5 Фабрика как класс

```javascript
class ConnectionFactory {
  constructor() {
    this.index = 0;
  }

  create() {
    return new Connection(this.index++);
  }
}

// Использование
const factory = new ConnectionFactory();
const conn1 = factory.create();
const conn2 = factory.create();
```

**Преимущества:**
- Четкая структура с явным состоянием
- Можно добавлять дополнительные методы
- Легче тестировать и расширять

### Метапрограммирование в фабриках

Фабрики в JavaScript могут использовать различные техники метапрограммирования:

```javascript
// Использование Object.create
function createWithPrototype(proto, props) {
  const obj = Object.create(proto);
  Object.assign(obj, props);
  return obj;
}

// Использование Object.defineProperties
function createWithDescriptors(props) {
  return Object.defineProperties({}, {
    name: {
      value: props.name,
      writable: true,
      enumerable: true
    }
  });
}

// Использование Object.setPrototypeOf
function createWithSetPrototype(obj, proto) {
  Object.setPrototypeOf(obj, proto);
  return obj;
}
```

**Важное замечание:**
- В платформенном коде допустимо использование "магии" и метапрограммирования
- В прикладном коде фабрики должны быть максимально простыми и понятными
- Предпочтительны явные и читаемые решения

---

## 3. Паттерн Object Pool (Пул объектов)

### Определение и назначение

**Пул объектов** - это паттерн, который создает и поддерживает коллекцию переиспользуемых объектов. Основная цель - оптимизация производительности и управление памятью.

### Основные принципы работы

1. **Предварительное создание** - пул создает определенное количество объектов заранее
2. **Выдача по запросу** - объекты выдаются из пула по мере необходимости
3. **Возврат для переиспользования** - использованные объекты возвращаются в пул
4. **Экономия ресурсов** - избегаем накладных расходов на создание новых объектов

### 3.1 Простейший синхронный пул

#### Базовая реализация

```javascript
// Простейший пул объектов
const pool = (factory) => {
  // Коллекция для хранения объектов
  const items = [];

  // Предзаполнение пула
  for (let i = 0; i < 10; i++) {
    items.push(factory());
  }

  // Возвращаем функцию для работы с пулом
  return (instance) => {
    if (instance) {
      // Если передан объект - возвращаем его в пул
      items.push(instance);
      console.log('Returned to pool, size:', items.length);
    } else {
      // Если объект не передан - выдаем из пула
      const item = items.pop();
      console.log('Taken from pool, size:', items.length);
      return item;
    }
  };
};
```

#### Пример использования с массивами

```javascript
// Фабрика для создания массивов из 1000 элементов
const arrayFactory = () => new Array(1000).fill(0);

// Создание пула
const arrayPool = pool(arrayFactory);

// Получение массива из пула
const arr1 = arrayPool();
const result = arr1.reduce((sum, val) => sum + val, 0);
console.log('Result:', result); // 0 (все элементы - нули)

// Получение еще одного массива
const arr2 = arrayPool();

// Возврат массива в пул
arrayPool(arr1);
```

**Ключевая особенность:** Одна и та же функция используется и для получения, и для возврата объектов в пул.

### 3.2 Пул с использованием фабрики классов

#### Класс Connection

```javascript
class Connection {
  constructor(index) {
    this.url = `https://example.com/connection-${index}`;
  }
}
```

#### Фабрика для Connection

```javascript
class ConnectionFactory {
  constructor() {
    this.index = 0;
  }

  create() {
    return new Connection(this.index++);
  }
}

// Создание фабрики и пула
const factory = new ConnectionFactory();
const connectionPool = pool(factory.create.bind(factory));

// Использование
const conn1 = connectionPool(); // Получаем соединение
console.log(conn1.url); // https://example.com/connection-0

const conn2 = connectionPool(); // Получаем еще одно
console.log(conn2.url); // https://example.com/connection-1

connectionPool(conn1); // Возвращаем в пул
```

### 3.3 Пул с автоматическим созданием новых объектов

Если пул исчерпан, можно автоматически создавать новые объекты:

```javascript
const poolWithFactory = (factory) => {
  const items = [];

  // Предварительное заполнение
  for (let i = 0; i < 5; i++) {
    items.push(factory());
  }

  return (instance) => {
    if (instance) {
      // Возврат в пул
      items.push(instance);
      console.log('Returned, pool size:', items.length);
    } else {
      // Попытка взять из пула
      let item = items.pop();

      // Если пул пуст - создаем новый объект
      if (!item) {
        item = factory();
        console.log('Pool empty, creating new instance');
      }

      console.log('Taken, pool size:', items.length);
      return item;
    }
  };
};
```

**Преимущества:**
- Пул не блокирует работу при исчерпании объектов
- Автоматическое расширение при необходимости
- Возможность переиспользования возвращенных объектов

**Потенциальная проблема:**
- Неограниченный рост количества созданных объектов
- Необходимость контроля верхнего лимита

### 3.4 Пул с ограничениями размера

#### Концепция трех уровней

Современные пулы часто используют три уровня размера:

1. **Минимальный размер (min)** - гарантированное минимальное количество объектов
2. **Нормальный размер (normal)** - оптимальное количество, создаваемое при инициализации
3. **Максимальный размер (max)** - верхний лимит, за который пул не должен выходить

```javascript
const poolWithLimits = (factory, min = 5, normal = 10, max = 15) => {
  const items = [];
  let totalCreated = 0;

  // Инициализация пула нормальным размером
  for (let i = 0; i < normal; i++) {
    items.push(factory());
    totalCreated++;
  }

  return (instance) => {
    if (instance) {
      // Возврат в пул (только если не превышен максимум)
      if (items.length < max) {
        items.push(instance);
        console.log(`Returned, pool size: ${items.length}`);
      } else {
        console.log('Pool is full, discarding instance');
      }
    } else {
      // Получение из пула
      let item = items.pop();

      if (!item) {
        // Проверка на максимальный лимит
        if (totalCreated < max) {
          item = factory();
          totalCreated++;
          console.log(`Created new, total: ${totalCreated}`);
        } else {
          // Достигнут максимум - возвращаем undefined
          console.log('Max limit reached');
          return undefined;
        }
      }

      console.log(`Taken, pool size: ${items.length}`);
      return item;
    }
  };
};
```

#### Пример с типизированными массивами

```javascript
// Фабрика для буферов 4KB (для обработки видео/криптографии)
const bufferFactory = () => new Uint8Array(4096);

// Создание пула с ограничениями
const bufferPool = poolWithLimits(bufferFactory, 5, 10, 15);

// Использование
for (let i = 0; i < 20; i++) {
  const buffer = bufferPool();
  if (buffer) {
    console.log(`Got buffer of size: ${buffer.length}`);
    // ... работа с буфером ...

    // Возврат в пул после использования
    bufferPool(buffer);
  } else {
    console.log('No buffer available');
  }
}
```

**Применение:**
- Обработка видеоконтента
- Криптографические операции
- Сетевые буферы для передачи данных
- Любые операции с большими блоками памяти

### 3.5 Пул, встроенный в класс

Фабрику можно встроить прямо в класс как статический метод:

```javascript
class Connection {
  static index = 0;
  static urlBase = 'https://example.com/connection-';

  static create() {
    return new Connection(this.index++);
  }

  constructor(index) {
    this.url = Connection.urlBase + index;
  }
}

// Пул, использующий встроенную фабрику
const poolBound = () => {
  const items = [];

  // Используем статический метод класса
  for (let i = 0; i < 5; i++) {
    items.push(Connection.create());
  }

  return (instance) => {
    if (instance) {
      items.push(instance);
    } else {
      return items.pop() || Connection.create();
    }
  };
};

const connPool = poolBound();
```

**Особенности:**
- Пул специализирован для конкретного типа объектов
- Меньше гибкости, но более простой API
- Фабрика и класс тесно связаны

---

## 4. Асинхронные пулы объектов

### Зачем нужны асинхронные пулы?

В реальных приложениях часто возникает ситуация, когда:
- Пул временно исчерпан
- Создание нового объекта нежелательно (лимит достигнут)
- Нужно дождаться возврата объекта в пул

### Основная концепция

```javascript
const asyncPool = (factory, size = 10) => {
  const items = [];
  const waiting = []; // Очередь ожидающих

  // Предзаполнение
  for (let i = 0; i < size; i++) {
    items.push(factory());
  }

  return {
    // Асинхронное получение объекта
    async acquire() {
      // Если есть доступный объект - возвращаем сразу
      if (items.length > 0) {
        return items.pop();
      }

      // Иначе - создаем промис и ждем
      return new Promise((resolve, reject) => {
        const timeout = setTimeout(() => {
          // Удаляем из очереди по таймауту
          const index = waiting.indexOf(resolve);
          if (index > -1) waiting.splice(index, 1);
          reject(new Error('Timeout waiting for pool item'));
        }, 5000); // Таймаут 5 секунд

        waiting.push({ resolve, timeout });
      });
    },

    // Возврат объекта в пул
    release(instance) {
      // Если кто-то ждет - отдаем ему
      if (waiting.length > 0) {
        const { resolve, timeout } = waiting.shift();
        clearTimeout(timeout);
        resolve(instance);
      } else {
        // Иначе - возвращаем в пул
        items.push(instance);
      }
    }
  };
};
```

#### Пример использования

```javascript
const connectionPool = asyncPool(() => new Connection(Date.now()), 3);

async function useConnection() {
  try {
    // Получаем соединение (ждем, если пул пуст)
    const conn = await connectionPool.acquire();

    // Используем соединение
    console.log('Using connection:', conn.url);
    await doSomethingWithConnection(conn);

    // Возвращаем в пул
    connectionPool.release(conn);
  } catch (error) {
    console.error('Failed to acquire connection:', error);
  }
}

// Запуск множества параллельных операций
Promise.all([
  useConnection(),
  useConnection(),
  useConnection(),
  useConnection(), // Этот будет ждать
  useConnection(), // И этот тоже
]);
```

### Ключевые особенности асинхронных пулов

**Преимущества:**
- Контролируемое ожидание освобождения ресурсов
- Не создаем избыточные объекты
- Graceful handling при исчерпании пула

**Обязательные элементы:**
- **Таймауты** - чтобы не ждать вечно
- **Очередь ожидания** - для справедливого распределения
- **Отмена ожидания** - при таймауте или ошибке

**Применение:**
- Пулы подключений к базам данных
- Пулы HTTP-соединений
- Пулы воркеров/тредов
- Любые ограниченные ресурсы

---

## 5. Практические примеры и паттерны использования

### 5.1 Пул подключений к базе данных

```javascript
class DatabaseConnection {
  constructor(connectionString) {
    this.connectionString = connectionString;
    this.connected = false;
  }

  async connect() {
    // Имитация подключения
    this.connected = true;
  }

  async disconnect() {
    this.connected = false;
  }

  async query(sql) {
    if (!this.connected) throw new Error('Not connected');
    // Выполнение запроса
  }
}

// Асинхронная фабрика
async function createConnection() {
  const conn = new DatabaseConnection('postgresql://localhost/mydb');
  await conn.connect();
  return conn;
}

// Создание пула
const dbPool = asyncPool(createConnection, 5);

// Использование
async function executeQuery(sql) {
  const connection = await dbPool.acquire();
  try {
    const result = await connection.query(sql);
    return result;
  } finally {
    dbPool.release(connection);
  }
}
```

### 5.2 Пул буферов для обработки данных

```javascript
// Пул для обработки больших файлов по частям
const bufferSize = 64 * 1024; // 64 KB
const bufferPool = pool(() => Buffer.alloc(bufferSize), 10, 20, 50);

async function processLargeFile(filePath) {
  const fileHandle = await fs.open(filePath, 'r');

  try {
    let bytesRead;
    do {
      const buffer = bufferPool(); // Получаем буфер из пула

      if (!buffer) {
        await new Promise(resolve => setTimeout(resolve, 100));
        continue;
      }

      bytesRead = await fileHandle.read(buffer, 0, bufferSize);

      // Обработка данных
      await processChunk(buffer.slice(0, bytesRead));

      // Возврат буфера в пул
      bufferPool(buffer);
    } while (bytesRead > 0);
  } finally {
    await fileHandle.close();
  }
}
```

### 5.3 Пул воркеров для параллельной обработки

```javascript
const { Worker } = require('worker_threads');

function createWorker() {
  return new Worker('./worker.js');
}

const workerPool = asyncPool(createWorker, 4); // 4 воркера

async function processTask(data) {
  const worker = await workerPool.acquire();

  return new Promise((resolve, reject) => {
    worker.once('message', (result) => {
      workerPool.release(worker);
      resolve(result);
    });

    worker.once('error', (error) => {
      workerPool.release(worker);
      reject(error);
    });

    worker.postMessage(data);
  });
}

// Обработка множества задач
const tasks = Array.from({ length: 100 }, (_, i) => ({ id: i, data: '...' }));
const results = await Promise.all(tasks.map(task => processTask(task)));
```

---

## 6. Сравнение подходов

### Конструктор vs Фабрика vs Пул

| Аспект | Конструктор | Фабрика | Пул объектов |
|--------|-------------|---------|--------------|
| **Простота** | Очень простой | Средняя | Сложная |
| **Гибкость** | Низкая | Высокая | Средняя |
| **Производительность** | Базовая | Базовая | Высокая |
| **Использование памяти** | Неконтролируемое | Неконтролируемое | Контролируемое |
| **Переиспользование** | Нет | Нет | Да |
| **Сложность кода** | Минимальная | Низкая | Средняя/Высокая |

### Когда использовать каждый подход

**Конструктор (`new`):**
- Простые объекты без сложной инициализации
- Объекты с коротким жизненным циклом
- Когда производительность не критична

**Фабрика:**
- Нужна абстракция от процесса создания
- Создание требует сложной логики
- Требуется создавать разные типы объектов
- Необходимо скрыть детали реализации

**Пул объектов:**
- Создание объектов дорогостоящее (DB connections, network sockets)
- Ограниченные ресурсы (память, подключения)
- Высокая частота создания/уничтожения объектов
- Необходимо контролировать количество экземпляров

---

## 7. Функциональный стиль vs ООП

### Замыкания vs Классы для хранения состояния

**Подход с замыканием (функциональный):**

```javascript
const alternativeFactory = (() => {
  let index = 0; // Мутируемое состояние в замыкании
  return () => new Connection(index++);
})();
```

**Подход с классом (ООП):**

```javascript
class Factory {
  constructor() {
    this.index = 0; // Явное состояние
  }
  create() {
    return new Connection(this.index++);
  }
}
```

### Критика функционального подхода

**Замечание из лекции:**
> "Настоящие функциональщики скажут: 'Ну вы же храните тут в замыкании значения. Мутируете. Какое тут может быть функциональное программирование?'"

**Реальность:**
- Мутирующее замыкание НЕ является функциональным программированием
- Это просто способ создания контекста с приватным состоянием
- В прикладном коде предпочтительны более явные подходы (классы)
- В платформенном коде допустима большая "магия"

---

## 8. Лучшие практики

### Для фабрик

1. **Держите фабрики простыми** в прикладном коде
2. **Используйте статические методы** для простых случаев
3. **Создавайте отдельные классы-фабрики** для сложной логики
4. **Избегайте излишнего метапрограммирования** в бизнес-логике

### Для пулов объектов

1. **Всегда устанавливайте лимиты** (min, normal, max)
2. **Используйте таймауты** в асинхронных пулах
3. **Очищайте объекты** перед возвратом в пул
4. **Логируйте метрики пула** для мониторинга
5. **Реализуйте graceful shutdown** для корректного закрытия

### Управление ресурсами

```javascript
// Хорошая практика: очистка при возврате в пул
const poolWithCleanup = (factory, cleanup) => {
  const items = [];

  for (let i = 0; i < 10; i++) {
    items.push(factory());
  }

  return (instance) => {
    if (instance) {
      cleanup(instance); // Очистка перед возвратом
      items.push(instance);
    } else {
      return items.pop() || factory();
    }
  };
};

// Пример с соединениями
const connPool = poolWithCleanup(
  () => new Connection(),
  (conn) => {
    conn.reset(); // Сброс состояния
    conn.clearCache(); // Очистка кеша
  }
);
```

---

## 9. Заключение

### Ключевые выводы

1. **Фабрики** предоставляют гибкость в создании объектов и абстрагируют детали инстанцирования

2. **Пулы объектов** критически важны для:
   - Оптимизации производительности
   - Контроля использования памяти
   - Управления ограниченными ресурсами

3. **Асинхронные пулы** необходимы для:
   - Работы с сетевыми ресурсами
   - Управления подключениями к БД
   - Координации параллельных операций

4. **Выбор подхода** зависит от:
   - Стоимости создания объектов
   - Требований к производительности
   - Ограничений ресурсов
   - Сложности системы

### Дальнейшее изучение

Темы для углубленного изучения:
- Асинхронные пулы с приоритетами
- Стратегии вытеснения в пулах (LRU, LFU)
- Мониторинг и метрики пулов
- Интеграция пулов с системами управления ресурсами
- Распределенные пулы в микросервисной архитектуре

---

## Дополнительные ресурсы

### Рекомендуемые материалы

- Отдельная лекция по асинхронным пулам (упоминается в лекции)
- Примеры кода в репозиториях с различными реализациями пулов
- Документация по Worker Threads для пулов воркеров
- Паттерны управления ресурсами в Node.js

### Практические задания

1. Реализовать пул с метриками (количество выдач, возвратов, созданий)
2. Создать асинхронный пул с очередью приоритетов
3. Разработать пул подключений с автоматическим переподключением
4. Имплементировать пул с различными стратегиями очистки
