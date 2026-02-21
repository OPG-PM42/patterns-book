# Паттерн Адаптер в Node.js и JavaScript - Q&A сессия

## Обзор

Эта лекция представляет собой глубокое погружение в паттерн Адаптер (Adapter Pattern) в контексте JavaScript и Node.js. Основная идея заключается в том, что реализация паттернов в JavaScript радикально отличается от их реализации в других языках программирования благодаря гибкости языка и его уникальным возможностям.

---

## Основная концепция паттерна Адаптер

### Определение

**Паттерн Адаптер** — это структурный паттерн проектирования, который позволяет объектам с несовместимыми интерфейсами работать вместе.

### Аналогия из реальной жизни

Представьте переходник между вилкой и розеткой:
- Есть вилка (один интерфейс)
- Есть розетка (другой интерфейс)
- Нужен адаптер, чтобы их состыковать

### Ключевое отличие от фасада

**Адаптер:**
- Преобразует один простой интерфейс в другой простой интерфейс
- Работает с относительно простыми абстракциями

**Фасад:**
- Скрывает сложную структуру за простым интерфейсом
- За фасадом стоит много абстракций, большое API

---

## Разнообразие реализаций в JavaScript

### Почему в JavaScript так много вариантов?

В отличие от языков со строгой типизацией (Java, C#), где паттерны реализуются по четко определенным "рецептам", JavaScript предоставляет огромное количество возможностей благодаря:

- Гибкости языка
- Поддержке разных парадигм (ООП, функциональное программирование)
- Динамической типизации
- Замыканиям и функциям высшего порядка
- Прототипному наследованию

### Основные подходы к реализации адаптера

#### 1. Боксинг (Boxing)

Помещение одного экземпляра класса или объекта внутрь другого.

**Пример: Адаптер для файловой системы с интерфейсом Map**

```javascript
// Адаптер Map для работы с файловой системой
class FileSystemMap {
  #api;
  #path;

  constructor(api, path) {
    this.#api = api;
    this.#path = path;
  }

  async set(key, value) {
    const filePath = `${this.#path}/${key}`;
    await this.#api.writeFile(filePath, JSON.stringify(value));
  }

  async get(key) {
    const filePath = `${this.#path}/${key}`;
    try {
      const data = await this.#api.readFile(filePath, 'utf8');
      return JSON.parse(data);
    } catch (err) {
      return undefined;
    }
  }

  async delete(key) {
    const filePath = `${this.#path}/${key}`;
    await this.#api.unlink(filePath);
  }

  get size() {
    // Реализация подсчета файлов
  }

  async keys() {
    const files = await this.#api.readdir(this.#path);
    return files;
  }
}

// Использование
const dictionary = new FileSystemMap(fs.promises, './data');
await dictionary.set('key1', { value: 'data' });
const value = await dictionary.get('key1');
```

**Преимущества:**
- Единый интерфейс Map для работы с разными хранилищами
- Можно легко переключаться между памятью, файловой системой, базой данных
- Прозрачность для клиентского кода

#### 2. Функции-обёртки (Wrapper Functions)

Функции, которые оборачивают другие функции, преобразуя их интерфейс.

**Пример: promisify - адаптер callback-last к промисам**

```javascript
// Адаптер от callback-стиля к Promise-стилю
function promisify(fn) {
  return (...args) => {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}

// Использование
const fs = require('fs');
const readFile = promisify(fs.readFile);

// Теперь можно использовать с async/await
const data = await readFile('file.txt', 'utf8');
```

**Ключевые моменты:**
- `promisify` принимает функцию с callback-last стилем
- Возвращает функцию, которая возвращает Promise
- Внутри создается callback, который resolve/reject промис

#### 3. Композиция vs Агрегация

**Композиция:**
- Абстракция создается внутри конструктора адаптера
- Адаптер владеет созданным объектом

**Агрегация:**
- Абстракция передается в адаптер снаружи (через конструктор или сеттер)
- Адаптер только использует переданный объект

**Пример композиции: Таймер с setInterval**

```javascript
class Timer {
  #interval;
  #counter = 0;
  #resolve;

  constructor(delay) {
    // Композиция - создаем setInterval внутри
    this.#interval = setInterval(() => {
      this.#counter++;
      if (this.#resolve) {
        this.#resolve(this.#counter);
      }
    }, delay);
  }

  // Реализация async iterator
  [Symbol.asyncIterator]() {
    return {
      next: () => {
        return new Promise((resolve) => {
          this.#resolve = (value) => {
            resolve({ value, done: false });
          };
        });
      }
    };
  }

  stop() {
    clearInterval(this.#interval);
  }
}

// Использование
const timer = new Timer(1000);
for await (const tick of timer) {
  console.log(`Tick: ${tick}`);
  if (tick >= 5) {
    timer.stop();
    break;
  }
}
```

**Пример агрегации:**

```javascript
class RequestAdapter {
  #request;

  constructor(request) {
    // Агрегация - получаем объект извне
    this.#request = request;
  }

  async get(url) {
    return this.#request({ method: 'GET', url });
  }

  async post(url, data) {
    return this.#request({ method: 'POST', url, body: data });
  }
}
```

#### 4. Реализация нескольких интерфейсов

Одна абстракция может одновременно реализовывать несколько интерфейсов - "универсальная розетка".

**Пример: NodeEventTarget (EventEmitter + EventTarget)**

```javascript
// В Node.js есть класс, который совмещает оба интерфейса
const { EventTarget } = require('events');

class UniversalEventBus extends EventTarget {
  // Методы EventTarget: addEventListener, removeEventListener, dispatchEvent

  // Добавляем методы EventEmitter
  on(event, listener) {
    this.addEventListener(event, listener);
  }

  emit(event, data) {
    this.dispatchEvent(new Event(event, { data }));
  }

  off(event, listener) {
    this.removeEventListener(event, listener);
  }
}

// Теперь код работает и на фронте, и в Node.js
const bus = new UniversalEventBus();
bus.on('message', (e) => console.log(e.data));
bus.emit('message', { text: 'Hello' });
```

**Ограничения:**
- Не всегда возможно совместить несовместимые интерфейсы
- Аналогия: невозможно сделать универсальный переходник для космического корабля и обычной розетки

#### 5. Revealing Constructor Pattern

Использование конструктора для передачи поведения вместо наследования.

**Пример: Readable Stream**

```javascript
const { Readable } = require('stream');

// Вместо наследования передаем функцию read в конструктор
const readable = new Readable({
  read(size) {
    // Кастомная логика чтения
    this.push(data);
  }
});
```

---

## Практические примеры адаптеров

### 1. Адаптер для асинхронных итераторов с таймером

```javascript
class Timer {
  #interval;
  #counter = 0;
  #resolve;

  constructor(delay) {
    this.#interval = setInterval(() => {
      this.#counter++;
      if (this.#resolve) {
        this.#resolve(this.#counter);
      }
    }, delay);
  }

  [Symbol.asyncIterator]() {
    return {
      next: () => {
        return new Promise((resolve) => {
          this.#resolve = (value) => {
            resolve({ value, done: false });
          };
        });
      }
    };
  }
}

// Использование
for await (const count of new Timer(1000)) {
  console.log(count); // 1, 2, 3, ...
}
```

### 2. Адаптер массива к интерфейсу очереди

```javascript
// Функция-адаптер добавляет методы очереди к массиву
function arrayToQueue(arr) {
  arr.enqueue = function(item) {
    this.push(item);
  };

  arr.dequeue = function() {
    return this.shift();
  };

  arr.peek = function() {
    return this[0];
  };

  return arr;
}

// Использование
const queue = arrayToQueue([]);
queue.enqueue(1);
queue.enqueue(2);
console.log(queue.dequeue()); // 1
```

### 3. Адаптер между парадигмами (ООП ↔ ФП)

```javascript
// Функциональный стиль
const map = (fn) => (arr) => arr.map(fn);
const filter = (predicate) => (arr) => arr.filter(predicate);

// Адаптер к ООП стилю
class Collection {
  #data;

  constructor(data) {
    this.#data = data;
  }

  map(fn) {
    return new Collection(map(fn)(this.#data));
  }

  filter(predicate) {
    return new Collection(filter(predicate)(this.#data));
  }

  toArray() {
    return this.#data;
  }
}

// Использование
const result = new Collection([1, 2, 3, 4])
  .map(x => x * 2)
  .filter(x => x > 4)
  .toArray(); // [6, 8]
```

---

## Адаптеры контрактов асинхронности

### Callback-last → Promise

```javascript
function promisify(fn) {
  return (...args) => {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}
```

### Promise → Callback

```javascript
function callbackify(fn) {
  return (...args) => {
    const callback = args.pop();

    fn(...args)
      .then(result => callback(null, result))
      .catch(err => callback(err));
  };
}
```

### Async Iterator → Stream

```javascript
const { Readable } = require('stream');

async function* generator() {
  for (let i = 0; i < 10; i++) {
    yield i;
  }
}

// Адаптер
const stream = Readable.from(generator());
```

---

## Важные концепции

### Клиент в JavaScript

**Ключевое отличие от Java/C#:**

В строго типизированных языках клиентом всегда является другая абстракция (класс), потому что код может существовать только внутри методов классов.

В JavaScript клиентом может быть:
- Код в модуле (top-level)
- Код внутри функции
- Код в обработчике события
- Код в любом другом месте

**Пример:**

```javascript
// usage.js - это и есть клиент
const timer = new Timer(1000);

for await (const tick of timer) {
  console.log(tick);
}
```

### Прототипный стиль (Legacy)

**Старый стиль:**

```javascript
function Timer(delay) {
  this.counter = 0;
  this.interval = setInterval(() => {
    this.counter++;
  }, delay);
}

Timer.prototype.stop = function() {
  clearInterval(this.interval);
};

Object.defineProperty(Timer.prototype, 'count', {
  get: function() {
    return this.counter;
  }
});
```

**Современный стиль (предпочтительный):**

```javascript
class Timer {
  #counter = 0;
  #interval;

  constructor(delay) {
    this.#interval = setInterval(() => {
      this.#counter++;
    }, delay);
  }

  stop() {
    clearInterval(this.#interval);
  }

  get count() {
    return this.#counter;
  }
}
```

---

## Культурные различия в реализации паттернов

### Проблема прямого переноса из других языков

Многие книги и статьи о паттернах просто переводят код с Java/C# на JavaScript **слово в слово**, не адаптируя его под:
- Особенности языка
- Идиомы платформы (Node.js vs Browser)
- Культуру JavaScript-сообщества

**Неправильный подход:**
```javascript
// Прямой перевод с Java
class AdapterImpl extends Adaptee implements Target {
  // Код выглядит как Java на JavaScript
}
```

**Правильный подход:**
```javascript
// Идиоматический JavaScript
const adapter = (adaptee) => ({
  method: () => adaptee.legacyMethod(),
  // Используем возможности языка
});
```

### Адаптация под культуру платформы

**Пример культурных различий:**

В Node.js принято:
- Использовать callback-last, error-first
- Использовать EventEmitter
- Использовать Streams

В браузере принято:
- Использовать EventTarget
- Использовать Promise/async-await
- Использовать Web APIs

**Универсальный адаптер:**

```javascript
// Работает и в Node.js, и в браузере
class UniversalTimer {
  constructor(delay) {
    if (typeof setInterval === 'function') {
      // Браузер или Node.js
      this.timer = setInterval(this.tick.bind(this), delay);
    }
  }

  tick() {
    // Универсальная логика
  }
}
```

---

## Специальные случаи и нюансы

### Await с неправильной реализацией

**Проблема:** Бесконечная рекурсия при `await` функции, которая возвращает промис

```javascript
// НЕПРАВИЛЬНО - вызовет переполнение стека
async function recurse() {
  return await recurse();
}
```

**Объяснение:**
1. Функция вызывается
2. `await` ждет промис
3. Функция вызывается рекурсивно снова
4. Очередь микротасков не успевает обработаться
5. Стек переполняется

**Правильно:** Использовать `await` со скалярным значением

```javascript
async function safe() {
  return await null; // Работает - заглушка пропускает null через await
}
```

**Как это работает:**

В JavaScript есть встроенная заглушка, которая:
1. Оборачивает скаляры в промис
2. Промис резолвится
3. Очередь микротасков прокручивается
4. Рекурсия прерывается

### Примеси (Mixins) vs Адаптеры

**Примесь - это НЕ адаптер**, если она:
- Просто добавляет поля к объекту
- Не преобразует интерфейс
- Не инкапсулирует другую абстракцию

**Примесь:**
```javascript
function addToken(obj, token) {
  obj.token = token; // Просто добавили поле
  return obj;
}
```

**Адаптер:**
```javascript
function addAuthAdapter(request, token) {
  return {
    get: (url) => request.get(url, { headers: { token } }),
    post: (url, data) => request.post(url, data, { headers: { token } })
  };
}
```

---

## Рекомендации по использованию

### Когда использовать адаптер?

1. **Интеграция сторонних библиотек** с несовместимыми интерфейсами
2. **Миграция между технологиями** (например, из монолита в микросервисы)
3. **Кросс-платформенный код** (Node.js + Browser)
4. **Преобразование контрактов** (callback → promise, sync → async)

### Когда НЕ использовать адаптер?

1. Если можно изменить исходный код напрямую
2. Если интерфейсы уже совместимы
3. Если адаптер добавляет только один метод (используйте простую функцию)

### Выбор типа адаптера

| Ситуация | Рекомендация |
|----------|--------------|
| Нужно обернуть класс | Боксинг (класс-обёртка) |
| Нужно обернуть функцию | Функция-обёртка |
| Нужно поддержать 2 интерфейса | Множественная реализация |
| Нужна гибкость | Revealing Constructor |
| Преобразование контрактов | Специализированные адаптеры (promisify, callbackify) |

---

## Ключевые выводы

1. **Гибкость JavaScript** позволяет реализовывать паттерны множеством способов
2. **Адаптер в JS** — это не строгая схема, а семейство техник для преобразования интерфейсов
3. **Культурная адаптация** важнее прямого переноса кода из других языков
4. **Боксинг и обёртки** — два основных подхода к реализации адаптеров
5. **Композиция vs агрегация** определяет владение адаптируемым объектом
6. **Клиентом в JS** может быть любой код, не только классы
7. **Современный синтаксис** (классы, приватные поля) предпочтительнее прототипного стиля

---

## Дополнительные ресурсы

### Примеры кода

Все примеры из лекции доступны в репозитории с практическими задачами на каждый тип адаптера.

### Задачи для практики

1. Реализовать адаптер Map для работы с Redis
2. Создать адаптер таймера с поддержкой AbortSignal
3. Написать адаптер между EventEmitter и EventTarget
4. Реализовать promisify для функций с нестандартным callback
5. Создать адаптер для работы с файловой системой через интерфейс Set

### Связанные паттерны

- **Декоратор** — добавляет функциональность, сохраняя интерфейс
- **Фасад** — упрощает сложный интерфейс
- **Мост (Bridge)** — разделяет абстракцию и реализацию
- **Стратегия** — может использоваться внутри адаптера для выбора реализации

---

## Заключение

Паттерн Адаптер в JavaScript — это мощный инструмент для работы с несовместимыми интерфейсами. Благодаря гибкости языка, мы можем создавать адаптеры, которые элегантно решают проблемы интеграции, не жертвуя читаемостью и производительностью кода.

Ключ к успешному применению паттерна — понимание контекста платформы и использование идиоматичных для JavaScript решений, а не слепое копирование реализаций из других языков программирования.
