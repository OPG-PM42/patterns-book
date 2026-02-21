# Паттерн Adapter (Адаптер) из GoF для JavaScript и TypeScript

## Обзор

Паттерн **Adapter** (Адаптер) — один из самых распространённых и важных паттернов проектирования из книги "Gang of Four". Он решает фундаментальную задачу преобразования одного контракта (интерфейса) в другой, позволяя совместно работать несовместимым абстракциям.

## Назначение паттерна

Паттерн Adapter используется для:

- **Скрытия одного контракта внутри** и **выдачи другого контракта наружу**
- Стыковки двух абстракций с разными интерфейсами
- Преобразования несовместимых контрактов для их взаимодействия
- Развязывания кода и повышения гибкости системы

**Ключевая идея**: Адаптер боксирует (оборачивает) одну абстракцию в "коробочку" и предоставляет другой интерфейс для работы с ней.

## Теоретические основы

### Что может быть адаптировано?

В JavaScript/TypeScript адаптировать можно любую абстракцию:
- Функции и процедуры
- Классы и прототипы
- Модули и компоненты
- Функторы и функциональные объекты
- Автоматы и стримы
- Любые другие абстракции

### Альтернативные названия

Паттерн Adapter также может называться:
- **Wrapper** (Обёртка)
- **Boxing** (Боксирование)

### Способы реализации

Адаптер может быть реализован тремя основными способами:

1. **Наследование (Extends)** — адаптер расширяет существующий класс
2. **Композиция** — адаптер создаёт экземпляр внутри себя
3. **Агрегация** — адаптер получает экземпляр извне

## Способы реализации адаптера

### 1. Наследование (Extends)

При использовании наследования адаптер одновременно реализует оба интерфейса.

#### Пример: Array to Queue Adapter с наследованием

```javascript
// Адаптер преобразует интерфейс Array в интерфейс Queue
class ArrayToQueueAdapter extends Array {
  // Queue интерфейс: enqueue, dequeue, count

  enqueue(item) {
    // Используем push из Array
    this.push(item);
  }

  dequeue() {
    // Используем shift из Array (удаляет первый элемент)
    return this.shift();
  }

  get count() {
    // Используем length из Array
    return this.length;
  }
}

// Использование
const queue = new ArrayToQueueAdapter();
queue.enqueue('first');
queue.enqueue('second');
queue.enqueue('third');

console.log(queue.dequeue()); // 'first'
console.log(queue.count);      // 2
```

**Важное замечание о терминологии:**
- В ООП это называется **расширение** (extends)
- В теории типов это называется **сужение**
- При extends область определения типа/класса всегда **сужается**
- Array используется повсеместно, ArrayToQueueAdapter — реже, следовательно его область применения уже

### 2. Агрегация

При агрегации адаптер получает объект извне и сохраняет его внутри.

#### Пример: Array to Queue Adapter с агрегацией

```javascript
class ArrayToQueueAdapter {
  #array;

  constructor(array) {
    // Получаем массив извне
    this.#array = array;
  }

  enqueue(item) {
    this.#array.push(item);
  }

  dequeue() {
    return this.#array.shift();
  }

  get count() {
    return this.#array.length;
  }
}

// Использование
const arr = []; // Создаём массив снаружи
const queue = new ArrayToQueueAdapter(arr); // Передаём в адаптер

queue.enqueue('first');
queue.enqueue('second');
console.log(queue.count); // 2
```

### 3. Функциональная реализация (без классов)

Адаптер можно реализовать без использования классов, применяя замыкания.

#### Пример: Array to Queue Adapter как функция

```javascript
// Функциональная реализация адаптера
const arrayToQueueAdapter = (array) => {
  // Возвращаем объект с методами Queue интерфейса
  return {
    enqueue(item) {
      array.push(item);
    },

    dequeue() {
      return array.shift();
    },

    get count() {
      return array.length;
    }
  };
};

// Использование
const arr = [];
const queue = arrayToQueueAdapter(arr);

queue.enqueue('first');
queue.enqueue('second');
console.log(queue.count); // 2
console.log(queue.dequeue()); // 'first'
```

**Преимущества функциональной реализации:**
- Объекты всегда имеют одинаковый интерфейс
- Одинаковая последовательность создания полей и методов
- Отличная оптимизация в V8 благодаря стабильной "форме объекта"
- Нет накладных расходов на классы и наследование

## Практические примеры из реального кода

### Пример 1: HashMapFS — Адаптер файловой системы к интерфейсу Map

Этот пример показывает, как можно скрыть работу с файловой системой за интерфейсом стандартного Map.

```javascript
class HashMapFS {
  #fs;
  #path;

  constructor(fs, path) {
    // Агрегация: получаем FS интерфейс извне
    this.#fs = fs;
    this.#path = path;
  }

  // Реализуем интерфейс Map
  set(key, value) {
    const filepath = `${this.#path}/${key}`;
    this.#fs.writeFileSync(filepath, JSON.stringify(value));
  }

  get(key) {
    const filepath = `${this.#path}/${key}`;
    try {
      const data = this.#fs.readFileSync(filepath, 'utf8');
      return JSON.parse(data);
    } catch (err) {
      return undefined;
    }
  }

  has(key) {
    const filepath = `${this.#path}/${key}`;
    try {
      this.#fs.accessSync(filepath);
      return true;
    } catch {
      return false;
    }
  }

  delete(key) {
    const filepath = `${this.#path}/${key}`;
    try {
      this.#fs.unlinkSync(filepath);
      return true;
    } catch {
      return false;
    }
  }

  get size() {
    const files = this.#fs.readdirSync(this.#path);
    return files.length;
  }

  keys() {
    return this.#fs.readdirSync(this.#path);
  }

  clear() {
    const files = this.#fs.readdirSync(this.#path);
    for (const file of files) {
      this.#fs.unlinkSync(`${this.#path}/${file}`);
    }
  }
}

// Использование
import fs from 'fs';

const hashMap = new HashMapFS(fs, './data');

hashMap.set('user1', { name: 'Alice', age: 30 });
hashMap.set('user2', { name: 'Bob', age: 25 });

console.log(hashMap.get('user1')); // { name: 'Alice', age: 30 }
console.log(hashMap.has('user2')); // true
console.log(hashMap.size);          // 2
```

**Преимущества такого подхода:**

1. **Развязка кода** — модуль, использующий Map, не знает о Node.js и файловой системе
2. **Взаимозаменяемость** — можно легко заменить реализацию:
   - `HashMapMemory` — хранение в памяти
   - `HashMapRedis` — хранение в Redis
   - `HashMapMongo` — хранение в MongoDB
   - `HashMapS3` — хранение в Amazon S3
3. **Переносимость** — код работает в разных окружениях (Node.js, Lambda, браузер)
4. **Тестируемость** — можно передать mock объект вместо настоящего fs

### Пример 2: Promisify — Классический адаптер контрактов

`promisify` — один из самых известных адаптеров в JavaScript, преобразующий callback-контракт в Promise-контракт.

#### Базовая реализация promisify

```javascript
// Простейшая версия promisify
const promisify = (fn) => {
  return (...args) => {
    return new Promise((resolve, reject) => {
      // Создаём callback для передачи в функцию
      const callback = (err, data) => {
        if (err) {
          reject(err);
        } else {
          resolve(data);
        }
      };

      // Вызываем исходную функцию с callback
      fn(...args, callback);
    });
  };
};

// Использование
import fs from 'fs';

const readFile = promisify(fs.readFile);

// Теперь можем использовать async/await
async function loadConfig() {
  try {
    const data = await readFile('./config.json', 'utf8');
    return JSON.parse(data);
  } catch (err) {
    console.error('Ошибка чтения файла:', err);
  }
}
```

#### Promisify с timeout

Расширенная версия с поддержкой таймаута:

```javascript
const promisifyWithTimeout = (fn) => {
  return (...args) => {
    // Создаём Promise с доступом к resolve/reject
    let resolve, reject;
    const promise = new Promise((res, rej) => {
      resolve = res;
      reject = rej;
    });

    // Состояние промиса
    let pending = true;
    let timer = null;

    // Извлекаем options из последнего аргумента
    const lastArg = args[args.length - 1];
    const options = typeof lastArg === 'object' ? lastArg : {};
    const timeout = options.timeout;

    // Устанавливаем таймер, если указан timeout
    if (timeout) {
      timer = setTimeout(() => {
        if (!pending) return;

        pending = false;
        timer = null;
        reject(new Error('Timed out'));
      }, timeout);
    }

    // Создаём callback
    const callback = (err, data) => {
      if (!pending) return; // Промис уже разрешён

      // Отменяем таймер
      if (timer) {
        clearTimeout(timer);
        timer = null;
      }

      pending = false;

      if (err) {
        reject(err);
      } else {
        resolve(data);
      }
    };

    // Вызываем исходную функцию
    fn(...args, callback);

    return promise;
  };
};

// Использование с timeout
const readFile = promisifyWithTimeout(fs.readFile);

try {
  // Файл должен быть прочитан за 1000 мс
  const data = await readFile('./large-file.txt', 'utf8', { timeout: 1000 });
  console.log(data);
} catch (err) {
  console.error(err.message); // "Timed out" если превышен timeout
}
```

#### Promisify с раздельными options

Более сложная версия, где options промисификации передаются отдельно:

```javascript
const promisifyAdvanced = (fn, options = {}) => {
  return (...args) => {
    let resolve, reject;
    const promise = new Promise((res, rej) => {
      resolve = res;
      reject = rej;
    });

    let pending = true;
    let timer = null;

    const { timeout } = options;

    if (timeout) {
      timer = setTimeout(() => {
        if (!pending) return;
        pending = false;
        timer = null;
        reject(new Error('Timed out'));
      }, timeout);
    }

    const callback = (err, data) => {
      if (!pending) return;

      if (timer) {
        clearTimeout(timer);
        timer = null;
      }

      pending = false;

      if (err) {
        reject(err);
      } else {
        resolve(data);
      }
    };

    fn(...args, callback);

    return promise;
  };
};

// Использование
const read = promisifyAdvanced(fs.readFile);

// Вызываем с параметрами для readFile и отдельно с timeout
const data = await read('./file.txt', 'utf8', { timeout: 1000 });
```

#### Promisify с AbortController

Современная версия с поддержкой отмены через AbortSignal:

```javascript
const promisifyWithAbort = (fn) => {
  return (...args) => {
    const lastArg = args[args.length - 1];
    const options = typeof lastArg === 'object' ? lastArg : {};
    const signal = options.signal;

    return new Promise((resolve, reject) => {
      // Проверяем, не отменён ли уже сигнал
      if (signal?.aborted) {
        reject(new Error('Operation aborted'));
        return;
      }

      // Подписываемся на событие отмены
      const abortHandler = () => {
        reject(new Error('Operation aborted'));
      };

      signal?.addEventListener('abort', abortHandler);

      const callback = (err, data) => {
        // Отписываемся от события
        signal?.removeEventListener('abort', abortHandler);

        if (err) {
          reject(err);
        } else {
          resolve(data);
        }
      };

      fn(...args, callback);
    });
  };
};

// Использование с AbortController
const read = promisifyWithAbort(fs.readFile);

// Вариант 1: Сигнал уже отменён
const ac1 = new AbortController();
ac1.abort(); // Сразу отменяем

try {
  await read('./file.txt', 'utf8', { signal: ac1.signal });
} catch (err) {
  console.error(err.message); // "Operation aborted"
}

// Вариант 2: Отмена через timeout
const ac2 = AbortController.timeout(1000); // AbortSignal с таймаутом

try {
  await read('./large-file.txt', 'utf8', { signal: ac2.signal });
} catch (err) {
  console.error(err.message); // "Operation aborted" если файл читается > 1 сек
}
```

### Пример 3: Адаптер setInterval к AsyncIterator

Преобразование таймера в асинхронный итератор для использования с `for await...of`.

```javascript
class Timer {
  #interval;
  #queue = [];
  #waiting = null;

  constructor(interval) {
    this.#interval = interval;

    // Запускаем таймер
    const timerId = setInterval(() => {
      const timestamp = Date.now();

      // Если есть ожидающий Promise, разрешаем его
      if (this.#waiting) {
        this.#waiting.resolve({ value: timestamp, done: false });
        this.#waiting = null;
      } else {
        // Иначе добавляем в очередь
        this.#queue.push(timestamp);
      }
    }, interval);

    // Сохраняем timerId для очистки
    this.#timerId = timerId;
  }

  // Реализация async iterator протокола
  async next() {
    // Если в очереди есть значения, возвращаем первое
    if (this.#queue.length > 0) {
      return { value: this.#queue.shift(), done: false };
    }

    // Иначе создаём Promise и ждём следующего тика
    return new Promise((resolve) => {
      this.#waiting = { resolve };
    });
  }

  // Symbol.asyncIterator делает объект асинхронно итерируемым
  [Symbol.asyncIterator]() {
    return this;
  }

  // Метод для остановки таймера
  stop() {
    clearInterval(this.#timerId);
    if (this.#waiting) {
      this.#waiting.resolve({ done: true });
    }
  }
}

// Использование
const timer = new Timer(1000); // Тик каждую секунду

// Проходим по таймеру циклом
for await (const timestamp of timer) {
  console.log('Tick:', new Date(timestamp).toISOString());

  // Останавливаем через 5 секунд
  if (timestamp > Date.now() - 5000) {
    timer.stop();
    break;
  }
}
```

**Преимущества:**
- Возможность использовать `for await...of` для работы с таймерами
- Совместимость со Stream Composition API
- Единообразный интерфейс для работы с асинхронными данными

### Пример 4: Адаптер setInterval к EventTarget

Преобразование таймера в EventTarget для работы с событиями.

```javascript
class TimerEventTarget extends EventTarget {
  #interval;
  #timerId;

  constructor(interval) {
    super();
    this.#interval = interval;
  }

  start() {
    this.#timerId = setInterval(() => {
      // Создаём и отправляем событие
      const event = new CustomEvent('tick', {
        detail: { timestamp: Date.now() }
      });
      this.dispatchEvent(event);
    }, this.#interval);
  }

  stop() {
    clearInterval(this.#timerId);
  }
}

// Использование
const timer = new TimerEventTarget(1000);

timer.addEventListener('tick', (event) => {
  console.log('Tick:', event.detail.timestamp);
});

timer.start();

// Останавливаем через 5 секунд
setTimeout(() => timer.stop(), 5000);
```

### Пример 5: EventTarget to AsyncIterator

Преобразование EventTarget в асинхронный итератор.

```javascript
class TargetIterator {
  #target;
  #eventName;
  #queue = [];
  #waiting = null;

  constructor(target, eventName) {
    this.#target = target;
    this.#eventName = eventName;

    // Подписываемся на события
    this.#target.addEventListener(eventName, (event) => {
      if (this.#waiting) {
        // Если есть ожидающий next(), разрешаем промис
        this.#waiting.resolve({ value: event, done: false });
        this.#waiting = null;
      } else {
        // Добавляем событие в очередь
        this.#queue.push(event);
      }
    });
  }

  async next() {
    // Если в очереди есть события, возвращаем первое
    if (this.#queue.length > 0) {
      return { value: this.#queue.shift(), done: false };
    }

    // Создаём Promise и ждём следующего события
    return new Promise((resolve) => {
      this.#waiting = { resolve };
    });
  }

  [Symbol.asyncIterator]() {
    return this;
  }
}

// Использование
const eventTarget = new EventTarget();

// Создаём асинхронный итератор для событий 'tick'
const iterator = new TargetIterator(eventTarget, 'tick');

// Отправляем события через setInterval
setInterval(() => {
  const event = new CustomEvent('tick', {
    detail: { timestamp: Date.now() }
  });
  eventTarget.dispatchEvent(event);
}, 1000);

// Обрабатываем события через for await...of
for await (const event of iterator) {
  console.log('Received event:', event.detail.timestamp);
}
```

**Применение:**
- Преобразование событийной модели в асинхронную итерацию
- Удобная работа с потоками событий
- Возможность использования async/await синтаксиса

## Комбинирование паттернов

Адаптер часто используется в комбинации с другими паттернами:

### Adapter + Revealing Constructor

```javascript
class ReadableStreamAdapter {
  #stream;

  constructor(readFunction) {
    // Revealing Constructor Pattern
    // readFunction инжектится в конструктор
    this.#stream = new ReadableStream({
      async start(controller) {
        try {
          const data = await readFunction();
          controller.enqueue(data);
          controller.close();
        } catch (err) {
          controller.error(err);
        }
      }
    });
  }

  // Адаптируем к AsyncIterator
  async *[Symbol.asyncIterator]() {
    const reader = this.#stream.getReader();
    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        yield value;
      }
    } finally {
      reader.releaseLock();
    }
  }
}
```

### Adapter + Strategy

```javascript
// Паттерн Strategy для выбора реализации Map
const mapStrategies = {
  memory: () => new Map(),

  fs: (fs, path) => new HashMapFS(fs, path),

  redis: (client) => ({
    async set(key, value) {
      await client.set(key, JSON.stringify(value));
    },
    async get(key) {
      const data = await client.get(key);
      return data ? JSON.parse(data) : undefined;
    },
    // ... другие методы Map
  }),

  s3: (s3Client, bucket) => ({
    async set(key, value) {
      await s3Client.putObject({
        Bucket: bucket,
        Key: key,
        Body: JSON.stringify(value)
      });
    },
    // ... другие методы Map
  })
};

// Фабрика с паттерном Strategy
function createHashMap(strategy, ...args) {
  const factory = mapStrategies[strategy];
  if (!factory) {
    throw new Error(`Unknown strategy: ${strategy}`);
  }
  return factory(...args);
}

// Использование
import fs from 'fs';

// Выбираем стратегию в runtime
const config = process.env.MAP_STORAGE || 'memory';

const hashMap = createHashMap(config, fs, './data');
// Код работает одинаково независимо от стратегии
hashMap.set('key', 'value');
```

## Преимущества использования адаптера

### 1. Развязка кода (Decoupling)

```javascript
// БЕЗ адаптера - код привязан к конкретной реализации
class UserService {
  saveUser(user) {
    // Жёстко привязаны к файловой системе Node.js
    fs.writeFileSync(`./users/${user.id}.json`, JSON.stringify(user));
  }

  loadUser(id) {
    return JSON.parse(fs.readFileSync(`./users/${id}.json`, 'utf8'));
  }
}

// С адаптером - код работает с абстракцией
class UserService {
  #storage;

  constructor(storage) {
    this.#storage = storage; // Любая реализация Map интерфейса
  }

  saveUser(user) {
    this.#storage.set(user.id, user);
  }

  loadUser(id) {
    return this.#storage.get(id);
  }
}

// Можем легко менять реализацию
const service1 = new UserService(new Map()); // В памяти
const service2 = new UserService(new HashMapFS(fs, './users')); // В файлах
const service3 = new UserService(new HashMapRedis(redis)); // В Redis
```

### 2. Переносимость между окружениями

```javascript
// Один и тот же код работает везде
class DataProcessor {
  #cache;

  constructor(cache) {
    this.#cache = cache;
  }

  async process(data) {
    // Одинаковый код для разных окружений
    const cached = await this.#cache.get(data.id);
    if (cached) return cached;

    const result = await this.heavyComputation(data);
    await this.#cache.set(data.id, result);
    return result;
  }
}

// В Node.js
const processor1 = new DataProcessor(new HashMapFS(fs, './cache'));

// В AWS Lambda
const processor2 = new DataProcessor(new HashMapS3(s3, 'my-bucket'));

// В браузере
const processor3 = new DataProcessor(new Map());
```

### 3. Упрощение тестирования

```javascript
// Mock реализация для тестов
class HashMapMock {
  #data = new Map();
  #calls = [];

  set(key, value) {
    this.#calls.push({ method: 'set', key, value });
    this.#data.set(key, value);
  }

  get(key) {
    this.#calls.push({ method: 'get', key });
    return this.#data.get(key);
  }

  getCalls() {
    return this.#calls;
  }
}

// В тестах
describe('UserService', () => {
  it('should save user', () => {
    const storage = new HashMapMock();
    const service = new UserService(storage);

    service.saveUser({ id: 1, name: 'Alice' });

    const calls = storage.getCalls();
    expect(calls).toHaveLength(1);
    expect(calls[0]).toEqual({
      method: 'set',
      key: 1,
      value: { id: 1, name: 'Alice' }
    });
  });
});
```

### 4. Скрытие сложности

```javascript
// Вся сложность работы с EventTarget скрыта в адаптере
class EventProcessor {
  async processEvents(eventSource) {
    // Простой и понятный код бизнес-логики
    for await (const event of eventSource) {
      await this.handleEvent(event);
    }
  }

  async handleEvent(event) {
    // Обработка события
    console.log('Processing:', event.type);
  }
}

// Использование
const target = new EventTarget();
const iterator = new TargetIterator(target, 'data');

const processor = new EventProcessor();
processor.processEvents(iterator);
// Бизнес-логика не знает про EventTarget, addEventListener и т.д.
```

## Терминология и концепции

### Композиция vs Агрегация

**Композиция** — адаптер создаёт объект внутри себя:

```javascript
class ArrayQueueComposition {
  #array;

  constructor() {
    // Создаём массив ВНУТРИ
    this.#array = [];
  }

  enqueue(item) {
    this.#array.push(item);
  }
}
```

**Агрегация** — адаптер получает объект извне:

```javascript
class ArrayQueueAggregation {
  #array;

  constructor(array) {
    // Получаем массив СНАРУЖИ
    this.#array = array;
  }

  enqueue(item) {
    this.#array.push(item);
  }
}
```

### Boxing (Боксирование)

Термин "boxing" означает помещение объекта в "коробку" (wrapper):

```javascript
// Боксируем примитивное значение
const boxed = { value: 42 };

// Боксируем функцию в объект
const boxedFunction = {
  execute: () => console.log('Hello')
};

// Боксируем класс в функцию
const boxedClass = () => {
  const instance = new SomeClass();
  return {
    method: () => instance.method()
  };
};
```

### Extends — расширение или сужение?

**Важная концепция из теории типов:**

```typescript
// В ООП говорят "расширение"
class ExtendedArray extends Array {
  // Добавляем новые методы
  sum() {
    return this.reduce((a, b) => a + b, 0);
  }
}

// Но в теории типов это "сужение"!
// Array может содержать любые элементы
// ExtendedArray — более специализированный тип

// Множество всех возможных Array > множество всех возможных ExtendedArray
// Следовательно, тип СУЖАЕТСЯ
```

## Хранение состояния в адаптерах

Существует множество способов хранения состояния:

### 1. Замыкания

```javascript
function createAdapter() {
  // Состояние в замыкании
  let state = { count: 0 };

  return {
    increment() {
      state.count++;
    },
    getCount() {
      return state.count;
    }
  };
}
```

### 2. Приватные поля (Private Fields)

```javascript
class Adapter {
  // Приватные поля
  #state = { count: 0 };

  increment() {
    this.#state.count++;
  }

  getCount() {
    return this.#state.count;
  }
}
```

### 3. Публичные поля

```javascript
class Adapter {
  // Публичные поля
  state = { count: 0 };

  increment() {
    this.state.count++;
  }
}
```

### 4. Символы (Symbols)

```javascript
const stateSymbol = Symbol('state');

class Adapter {
  constructor() {
    // Состояние в символе
    this[stateSymbol] = { count: 0 };
  }

  increment() {
    this[stateSymbol].count++;
  }
}
```

### 5. WeakMap

```javascript
const states = new WeakMap();

class Adapter {
  constructor() {
    // Состояние в WeakMap
    states.set(this, { count: 0 });
  }

  increment() {
    const state = states.get(this);
    state.count++;
  }
}
```

## Стандартные vs Кастомные интерфейсы

### Предпочитайте стандартные интерфейсы

**Хорошо** — использование стандартных интерфейсов:

```javascript
// Iterable — стандартный интерфейс
class MyCollection {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
    yield 3;
  }
}

// Можно использовать с for...of
for (const item of new MyCollection()) {
  console.log(item);
}
```

**Плохо** — изобретение велосипеда:

```javascript
// Кастомный интерфейс итерации
class MyCollection {
  forEach(callback) {
    callback(1);
    callback(2);
    callback(3);
  }
}

// Нельзя использовать со стандартными конструкциями
const collection = new MyCollection();
collection.forEach(item => console.log(item)); // Работает
// for (const item of collection) {} // НЕ работает!
```

### Примеры стандартных интерфейсов

1. **Iterable / Iterator** — синхронная итерация
2. **AsyncIterable / AsyncIterator** — асинхронная итерация
3. **Promise / Thenable** — асинхронные операции
4. **EventTarget** — событийная модель
5. **ReadableStream / WritableStream** — потоковая обработка
6. **Map / Set** — коллекции
7. **AbortSignal** — отмена операций

## Исторический контекст

### Эволюция асинхронных контрактов в JavaScript

```javascript
// 1. Callbacks (первое поколение)
fs.readFile('./file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error(err);
  } else {
    console.log(data);
  }
});

// 2. Promises (второе поколение)
readFile('./file.txt', 'utf8')
  .then(data => console.log(data))
  .catch(err => console.error(err));

// 3. Async/Await (третье поколение)
try {
  const data = await readFile('./file.txt', 'utf8');
  console.log(data);
} catch (err) {
  console.error(err);
}

// 4. Async Iterators (четвёртое поколение)
for await (const chunk of readableStream) {
  console.log(chunk);
}

// Адаптеры позволяют переходить между этими поколениями!
```

### Другие асинхронные абстракции

- **Deferreds** — предшественники Promise (использовались в jQuery)
- **Observables** — реактивное программирование (RxJS)
- **Async Generators** — комбинация async/await и generators
- **Disposables** — управление ресурсами

## Распространённость паттерна

Adapter — один из **самых распространённых паттернов** в JavaScript по следующим причинам:

1. **Постоянные несовпадения контрактов** в экосистеме JavaScript
2. **Эволюция языка** — новые контракты появляются, старые остаются
3. **Множество окружений** — браузер, Node.js, Deno, Bun
4. **Интеграция библиотек** — разные библиотеки используют разные интерфейсы
5. **Межплатформенность** — один код для разных платформ

### Примеры повсеместного использования

```javascript
// В Node.js
util.promisify()      // Callback → Promise
stream.pipeline()     // Стыковка стримов
Buffer.from()         // Различные форматы → Buffer

// В браузерах
fetch()               // XMLHttpRequest → Promise
new Response()        // Stream → Response
new FormData()        // Объект → FormData

// Библиотеки
axios                 // Адаптер для XHR и fetch
bluebird.promisify()  // Callback → Promise
lodash.wrap()         // Функция → обёрнутая функция
```

## Взаимодействие с другими языками

Знание паттернов и терминологии позволяет общаться с разработчиками из других экосистем:

- **Java** — интерфейсы, адаптеры классов
- **C#** — расширения, делегаты, адаптеры
- **C++** — шаблоны адаптеров, proxy
- **Python** — декораторы, wrapper
- **PHP** — trait, adapter
- **Kotlin** — extension functions
- **Go** — interface{}, type embedding

**Важно:** Терминология помогает создать общий язык для обсуждения архитектуры и дизайна.

## Оптимизация и производительность

### V8 оптимизации для функциональных адаптеров

```javascript
// Этот код ОЧЕНЬ эффективен для V8
function createAdapter(data) {
  // Всегда возвращаем объект с одинаковой структурой
  return {
    get() { return data; },
    set(value) { data = value; },
    transform(fn) { return fn(data); }
  };
}

// V8 создаёт "hidden class" (форму объекта)
// Все объекты имеют одинаковую форму
// Оптимизации inline caching работают максимально эффективно

const adapter1 = createAdapter(10);
const adapter2 = createAdapter(20);
const adapter3 = createAdapter(30);

// Все три объекта имеют ОДИНАКОВУЮ внутреннюю структуру
// V8 может применить агрессивную оптимизацию
```

### Антипаттерны, вызывающие деоптимизацию

```javascript
// ПЛОХО - динамическое добавление свойств
function createBadAdapter(data) {
  const obj = { get() { return data; } };

  // Изменяем структуру после создания
  if (typeof data === 'string') {
    obj.length = () => data.length; // Деоптимизация!
  }

  return obj;
}
```

## Ключевые выводы

### Когда использовать Adapter

✅ **Используйте адаптер когда:**
- Нужно состыковать два несовместимых интерфейса
- Хотите скрыть детали реализации за стандартным интерфейсом
- Требуется поддержка нескольких реализаций одного интерфейса
- Необходимо изолировать зависимости от конкретных библиотек/платформ
- Нужно сделать код более тестируемым

❌ **НЕ используйте адаптер когда:**
- Интерфейсы уже совместимы
- Добавление адаптера создаёт избыточную сложность
- Можно просто изменить исходный код

### Золотые правила

1. **Скрывайте сложность за адаптерами** — бизнес-логика должна быть чистой
2. **Используйте стандартные интерфейсы** — Iterable, Promise, EventTarget и т.д.
3. **Комбинируйте с другими паттернами** — Strategy, Factory, Revealing Constructor
4. **Думайте о тестируемости** — адаптер должен легко мокироваться
5. **Помните о производительности** — стабильная структура объектов = быстрый код

## Практические рекомендации

### Чек-лист при проектировании адаптера

- [ ] Определён чёткий контракт входа (что адаптируем)
- [ ] Определён чёткий контракт выхода (во что адаптируем)
- [ ] Выбран способ реализации (extends/composition/aggregation)
- [ ] Решено где хранить состояние (closure/fields/WeakMap)
- [ ] Учтена обработка ошибок
- [ ] Продумана очистка ресурсов (если нужна)
- [ ] Написаны тесты для всех методов адаптера
- [ ] Проверена совместимость с целевым интерфейсом

### Паттерны именования

```javascript
// Хорошие имена адаптеров
promisify()           // Verb - описывает преобразование
ArrayToQueue          // SourceToTarget - показывает направление
HashMapFS             // TargetStorage - цель + реализация
TargetIterator        // SourceAdapter - что адаптируем

// Плохие имена
Adapter               // Слишком общее
Helper                // Непонятное назначение
Utils                 // Неинформативное
```

## Дополнительные материалы

### Связанные паттерны

- **Facade** — упрощает интерфейс сложной системы
- **Proxy** — контролирует доступ к объекту
- **Decorator** — добавляет функциональность объекту
- **Bridge** — разделяет абстракцию и реализацию
- **Strategy** — инкапсулирует семейство алгоритмов

### Примеры из реальных проектов

```javascript
// Express.js middleware — адаптеры для HTTP
app.use(express.json());        // Body parser adapter
app.use(express.static('./'));  // Static files adapter

// Mongoose — адаптер MongoDB к ORM интерфейсу
const User = mongoose.model('User', schema);

// Sequelize — адаптер SQL к ORM интерфейсу
const User = sequelize.define('User', { ... });

// Socket.io — адаптер WebSocket к событийной модели
io.on('connection', socket => { ... });
```

## Заключение

Паттерн Adapter — фундаментальный инструмент в арсенале JavaScript/TypeScript разработчика. Он решает одну из самых частых задач — преобразование несовместимых интерфейсов — и делает это элегантно и эффективно.

**Основная ценность адаптера:**
- Развязывает код и повышает гибкость
- Скрывает сложность за простым интерфейсом
- Обеспечивает переносимость между платформами
- Улучшает тестируемость кода
- Позволяет использовать стандартные контракты

Глубокое понимание этого паттерна открывает путь к написанию по-настоящему профессионального, гибкого и поддерживаемого кода.
