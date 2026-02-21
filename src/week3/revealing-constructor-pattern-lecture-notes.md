# Паттерн Revealing Constructor (Открытый конструктор)

## Обзор

Паттерн **Revealing Constructor** (Открытый конструктор) — это паттерн проектирования, который позволяет инициализировать объект через функцию, передаваемую в конструктор. Этот паттерн необходим для понимания работы Promise и построения более сложных абстракций для работы с асинхронностью, таких как паттерн Future (без состояния и многоразовый, в отличие от Promise).

**Основная идея:** Функция попадает в конструктор и инициализирует экземпляр объекта, изменяя его поведение или внедряя зависимости.

---

## Три способа передачи значений в функцию

Прежде чем перейти к паттерну, рассмотрим три различных способа передачи значений внутрь функции:

### Пример: Различные способы передачи данных

```javascript
// Три способа передачи значений
const example = (x, y, z) => {
  // 1. Константа - прямое значение
  console.log(x); // Обращаемся напрямую к переменной

  // 2. Функция, возвращающая константу
  console.log(y()); // Вызываем функцию для получения значения

  // 3. Callback-функция
  z((value) => {
    console.log(value); // Значение приходит через callback
  });
};

// Вызов с различными типами аргументов
example(
  5,                    // 1. Просто константа
  () => 10,            // 2. Функция, возвращающая значение
  (callback) => {      // 3. Функция, принимающая callback
    callback(15);
  }
);
```

**Вывод:**
- **Первый способ (константа):** Самый простой, значение передаётся напрямую
- **Второй способ (функция):** Позволяет отложить вычисление значения
- **Третий способ (callback):** Позволяет асинхронно вернуть значение в будущем

---

## Применение паттерна в Promise

Именно **третий способ** (callback) используется в конструкторе Promise и является примером паттерна Revealing Constructor.

### Пример: Promise с Revealing Constructor

```javascript
// Promise использует паттерн Revealing Constructor
const promise1 = new Promise((resolve, reject) => {
  // Сразу разрешаем промис
  resolve(5);
});

const promise2 = new Promise((resolve, reject) => {
  // Ждём секунду, затем разрешаем
  setTimeout(() => {
    resolve(5);
  }, 1000);
});

// Первый слушатель - навешивается ДО разрешения промиса
promise2.then((x) => {
  console.log('First listener:', x);
});

// Ждём 1.5 секунды
setTimeout(() => {
  // Второй слушатель - навешивается ПОСЛЕ разрешения промиса
  promise2.then((y) => {
    console.log('Second listener:', y);
  });
}, 1500);

console.log(promise2); // Promise { <pending> }
```

**Результат выполнения:**
```
Promise { <pending> }
First listener: 5
Second listener: 5
```

### Почему Promise нужен именно callback?

**Ключевой момент:** Если бы в Promise передавали просто константу или функцию, возвращающую константу, мы не смогли бы:

1. **Отложить разрешение промиса** (подождать секунду в примере выше)
2. **Переводить промис из состояния pending** в resolved или rejected
3. **Подписываться на значение до и после его разрешения**

```javascript
// ❌ Это НЕ сработало бы:
const badPromise = new Promise(() => {
  return 5; // return сразу вернул бы значение,
            // не дав возможности асинхронно разрешить промис
});

// ✅ Правильный способ - через callback (resolve/reject):
const goodPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve(5); // Асинхронное разрешение через callback
  }, 1000);
});
```

---

## Применение паттерна в Node.js Streams

Паттерн Revealing Constructor широко используется в стримах Node.js, позволяя избежать наследования классов.

### Традиционный подход: Наследование через extends

```javascript
const { Readable } = require('node:stream');

// Создаём класс RandomStream, наследуясь от Readable
class RandomStream extends Readable {
  _read(size) {
    // Генерируем массив случайных байтов нужной длины
    const chunk = Buffer.allocUnsafe(size)
      .map(() => Math.floor(Math.random() * 256));

    // Преобразуем в шестнадцатеричную строку
    const hexString = chunk.toString('hex');

    // Отправляем данные в буфер стрима
    this.push(hexString);
  }
}

// Использование
const randomStream = new RandomStream();
randomStream.on('data', (chunk) => {
  console.log(chunk);
});
```

### Современный подход: Revealing Constructor

```javascript
const { Readable } = require('node:stream');

// Создаём Readable stream БЕЗ наследования
const randomStream = new Readable({
  read(size) {
    // Генерируем массив случайных байтов
    const chunk = Buffer.allocUnsafe(size)
      .map(() => Math.floor(Math.random() * 256));

    // Преобразуем в шестнадцатеричную строку
    const hexString = chunk.toString('hex');

    // Отправляем данные в буфер
    this.push(hexString);
  }
});

// Использование точно такое же
randomStream.on('data', (chunk) => {
  console.log(chunk);
});
```

**Преимущества Revealing Constructor в стримах:**

1. **Нет необходимости создавать отдельный класс**
2. **Более компактный код**
3. **Внедрение зависимостей** - метод `read` инжектится внутрь экземпляра Readable
4. **Именно так реализованы стримы в стандартной библиотеке Node.js**

### Практический пример: Pipe в stdout

```javascript
const { Readable } = require('node:stream');

const randomStream = new Readable({
  read(size) {
    const chunk = Buffer.allocUnsafe(size)
      .map(() => Math.floor(Math.random() * 256));
    this.push(chunk.toString('hex') + '\n');
  }
});

// Pipe в stdout для бесконечного вывода случайных чисел
randomStream.pipe(process.stdout);
// Генерирует и выводит бесконечный поток случайных hex-строк
```

**Почему это работает:**
- Как только происходит `pipe`, Readable начинает вызывать метод `read`
- Метод `read` генерирует данные и пушит их в буфер через `this.push()`
- Данные автоматически передаются в Writable stream (stdout)

---

## Другие типы стримов с Revealing Constructor

### Transform Stream

#### Традиционный подход с наследованием:

```javascript
const { Transform } = require('node:stream');

// Стрим, который делает все буквы заглавными
class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // Преобразуем chunk в заглавные буквы
    const upperChunk = chunk.toString().toUpperCase();
    // Передаём результат дальше
    callback(null, upperChunk);
  }
}

// Использование
const upperTransform = new UpperCaseTransform();
process.stdin
  .pipe(upperTransform)
  .pipe(process.stdout);
```

#### Revealing Constructor подход:

```javascript
const { Transform } = require('node:stream');

// Создаём Transform stream без наследования
const upperTransform = new Transform({
  transform(chunk, encoding, callback) {
    // Преобразуем chunk в заглавные буквы
    const upperChunk = chunk.toString().toUpperCase();
    // Передаём результат дальше
    callback(null, upperChunk);
  }
});

// Использование - абсолютно идентично
process.stdin
  .pipe(upperTransform)
  .pipe(process.stdout);
```

### Writable Stream

```javascript
const { Writable } = require('node:stream');

// Создаём Writable stream через Revealing Constructor
const myWritable = new Writable({
  write(chunk, encoding, callback) {
    console.log('Received:', chunk.toString());
    callback(); // Сигнализируем о завершении записи
  }
});
```

### Duplex Stream

```javascript
const { Duplex } = require('node:stream');

// Для Duplex нужно определить и read, и write
const myDuplex = new Duplex({
  read(size) {
    // Логика чтения
    this.push('data');
  },
  write(chunk, encoding, callback) {
    // Логика записи
    console.log(chunk.toString());
    callback();
  }
});
```

---

## Создание собственных абстракций с Revealing Constructor

Паттерн можно использовать не только для Promise и стримов, но и для создания собственных абстракций.

### Пример: Класс Timer

```javascript
class Timer {
  constructor(interval, listener) {
    this.interval = interval;  // Интервал в миллисекундах
    this.listener = listener;  // Функция-обработчик
    this.timerId = null;       // ID таймера
  }

  start() {
    // Запускаем setInterval с переданной функцией
    this.timerId = setInterval(this.listener, this.interval);
  }

  stop() {
    // Останавливаем таймер
    if (this.timerId) {
      clearInterval(this.timerId);
      this.timerId = null;
    }
  }
}

// Использование
const timer = new Timer(1000, () => {
  console.log('Timer event');
});

timer.start(); // Начинаем вызов каждую секунду

setTimeout(() => {
  timer.stop(); // Останавливаем через 5 секунд
}, 5000);
```

**Результат выполнения:**
```
Timer event
Timer event
Timer event
Timer event
```
(4 события за ~5 секунд)

**Особенности паттерна в Timer:**
- Функция `listener` передаётся в конструктор
- Хранится внутри экземпляра объекта
- Используется вместо метода, который можно было бы переопределить
- Изменяет поведение базового функционала (setInterval)

---

## Различные формы передачи функций в Revealing Constructor

### Форма 1: Прямая передача функции (Promise, Timer)

```javascript
// Promise
new Promise((resolve, reject) => {
  // resolve и reject - это callbacks
});

// Timer
new Timer(1000, () => {
  // Функция передаётся напрямую
});
```

### Форма 2: Передача через объект (Streams)

```javascript
// Readable Stream
new Readable({
  read(size) {
    // Функция передаётся как метод объекта
  }
});

// Transform Stream
new Transform({
  transform(chunk, encoding, callback) {
    // Именованная функция в объекте
  }
});

// Duplex Stream
new Duplex({
  read(size) { /* ... */ },
  write(chunk, encoding, callback) { /* ... */ }
});
```

---

## Ключевые концепции паттерна

### 1. Внедрение зависимостей (Dependency Injection)

Revealing Constructor очень похож на внедрение зависимостей, только:
- **Внедряется одна или несколько функций** внутрь экземпляра
- **Функции изменяют поведение** базового класса
- **Альтернатива наследованию** классов

```javascript
// Вместо наследования:
class MyStream extends Readable {
  _read(size) { /* ... */ }
}

// Используем внедрение через конструктор:
const myStream = new Readable({
  read(size) { /* ... */ }
});
```

### 2. Инициализация состояния объекта

Переданная функция может:
- Изменять внутреннее состояние объекта
- Переводить объект из одного состояния в другое (pending → fulfilled/rejected в Promise)
- Управлять жизненным циклом объекта

### 3. Асинхронное управление

Паттерн позволяет:
- Откладывать выполнение операций
- Работать с асинхронными операциями
- Подписываться на события до и после их наступления

---

## Преимущества паттерна

### 1. Инкапсуляция

```javascript
// Внутренние методы (resolve/reject) доступны
// только внутри функции-конструктора
new Promise((resolve, reject) => {
  // resolve и reject доступны ТОЛЬКО здесь
  setTimeout(() => resolve(42), 1000);
});

// Снаружи нельзя напрямую вызвать resolve или reject
```

### 2. Гибкость

```javascript
// Можно передать любую логику инициализации
const stream1 = new Readable({
  read() { this.push('simple data'); }
});

const stream2 = new Readable({
  read(size) {
    // Сложная логика с асинхронными операциями
    fetchData().then(data => this.push(data));
  }
});
```

### 3. Избежание наследования

```javascript
// Не нужно создавать класс для каждого случая
// Плохо:
class SimpleReadable extends Readable { _read() { /* ... */ } }
class ComplexReadable extends Readable { _read() { /* ... */ } }
class AnotherReadable extends Readable { _read() { /* ... */ } }

// Хорошо:
const simple = new Readable({ read() { /* ... */ } });
const complex = new Readable({ read() { /* ... */ } });
const another = new Readable({ read() { /* ... */ } });
```

### 4. Контроль доступа

```javascript
// Promise открывает только resolve/reject внутри конструктора
// но не даёт доступ к ним извне
const promise = new Promise((resolve, reject) => {
  // Только здесь можно управлять состоянием
  setTimeout(() => resolve('done'), 1000);
});

// promise.resolve() - ❌ Такого метода нет!
// promise.reject()  - ❌ Такого метода нет!
```

---

## Сравнение подходов

| Аспект | Наследование (extends) | Revealing Constructor |
|--------|----------------------|---------------------|
| **Создание класса** | Требуется | Не требуется |
| **Многословность** | Больше кода | Компактнее |
| **Переиспользование** | Через иерархию классов | Через композицию функций |
| **Гибкость** | Ограничена иерархией | Высокая |
| **Инкапсуляция** | Через приватные поля | Через замыкания |

### Пример сравнения для стримов:

```javascript
// Подход 1: Наследование (многословно)
class RandomStream extends Readable {
  constructor(options) {
    super(options);
  }
  _read(size) {
    const chunk = Buffer.allocUnsafe(size)
      .map(() => Math.floor(Math.random() * 256));
    this.push(chunk.toString('hex'));
  }
}
const stream1 = new RandomStream();

// Подход 2: Revealing Constructor (компактно)
const stream2 = new Readable({
  read(size) {
    const chunk = Buffer.allocUnsafe(size)
      .map(() => Math.floor(Math.random() * 256));
    this.push(chunk.toString('hex'));
  }
});
```

---

## Применение в реальных проектах

### 1. Promise API

Все промисы используют этот паттерн:

```javascript
// Работа с файлами
const filePromise = new Promise((resolve, reject) => {
  fs.readFile('file.txt', (err, data) => {
    if (err) reject(err);
    else resolve(data);
  });
});

// HTTP запросы
const httpPromise = new Promise((resolve, reject) => {
  http.get('http://api.example.com', (res) => {
    let data = '';
    res.on('data', chunk => data += chunk);
    res.on('end', () => resolve(data));
    res.on('error', reject);
  });
});
```

### 2. Node.js Streams

Стандартная библиотека Node.js реализует все типы стримов с поддержкой Revealing Constructor:

```javascript
const { Readable, Writable, Transform, Duplex } = require('node:stream');

// Все эти конструкторы принимают объекты с функциями
const r = new Readable({ read() { /* ... */ } });
const w = new Writable({ write() { /* ... */ } });
const t = new Transform({ transform() { /* ... */ } });
const d = new Duplex({ read() { /* ... */ }, write() { /* ... */ } });
```

### 3. Custom Event Emitters

```javascript
class Emitter {
  constructor(listener) {
    this.listener = listener;
    this.events = {};
  }

  emit(event, data) {
    if (this.listener) {
      this.listener(event, data);
    }
  }
}

const emitter = new Emitter((event, data) => {
  console.log(`Event ${event}:`, data);
});

emitter.emit('message', 'Hello!');
```

---

## Связь с паттерном Future

Паттерн Revealing Constructor будет использован для создания паттерна **Future** в следующей лекции:

**Отличия Future от Promise:**
- **Без состояния** (stateless)
- **Многоразовый** (можно использовать повторно)
- **Из функционального программирования**
- **Компонуемый** (легко комбинируется)

```javascript
// Пример концепции (упрощённо)
class Future {
  constructor(computation) {
    this.computation = computation; // Сохраняем вычисление
  }

  fork(reject, resolve) {
    // Каждый раз запускаем вычисление заново
    this.computation(reject, resolve);
  }
}

// Future можно запустить многократно
const future = new Future((reject, resolve) => {
  setTimeout(() => resolve(42), 1000);
});

future.fork(console.error, console.log); // Первый запуск
future.fork(console.error, console.log); // Второй запуск
```

---

## Практические рекомендации

### Когда использовать Revealing Constructor

✅ **Используйте когда:**
- Нужно внедрить поведение в конструктор объекта
- Требуется инкапсулировать внутренние методы
- Хотите избежать создания множества классов через наследование
- Работаете с асинхронными операциями
- Нужно управлять состоянием объекта извне конструктора

❌ **Не используйте когда:**
- Логика слишком сложная для одной функции
- Нужна полноценная иерархия классов
- Требуется множественное наследование поведения
- Важна типизация и IDE поддержка (без TypeScript)

### Best Practices

```javascript
// ✅ Хорошо: Явные имена параметров
new Promise((resolve, reject) => {
  // Понятно, что делают эти функции
});

// ✅ Хорошо: Документированные методы объекта
new Readable({
  read(size) {
    // size - понятный параметр
  }
});

// ❌ Плохо: Неясные имена
new SomeClass((a, b) => {
  // Что такое a и b?
});

// ✅ Хорошо: Обработка ошибок
new Promise((resolve, reject) => {
  try {
    // ... операция
    resolve(result);
  } catch (error) {
    reject(error);
  }
});
```

---

## Резюме

### Основные концепции

1. **Revealing Constructor** - паттерн, при котором функция передаётся в конструктор для инициализации объекта
2. **Три способа передачи данных**: константа, функция возвращающая значение, callback-функция
3. **Promise использует Revealing Constructor** для управления состоянием (resolve/reject)
4. **Node.js Streams** используют этот паттерн для внедрения методов read/write/transform
5. **Альтернатива наследованию** - можно избежать создания классов

### Ключевые преимущества

- ✅ Инкапсуляция внутренних методов
- ✅ Гибкость в настройке поведения
- ✅ Компактный код без наследования
- ✅ Внедрение зависимостей через конструктор
- ✅ Поддержка асинхронных операций

### Области применения

- Promise API
- Node.js Streams (Readable, Writable, Transform, Duplex)
- Custom таймеры и планировщики
- Event Emitters
- Паттерн Future (следующая тема)

### Следующие шаги

В следующей лекции мы построим паттерн **Future** на основе Revealing Constructor:
- Без состояния (в отличие от Promise)
- Многоразовый
- Из функционального программирования
- Для работы с асинхронностью

---

## Дополнительные примеры для практики

### Пример 1: Observable-подобный класс

```javascript
class Observable {
  constructor(subscriber) {
    this.subscriber = subscriber;
  }

  subscribe(observer) {
    return this.subscriber(observer);
  }
}

// Использование
const observable = new Observable((observer) => {
  let count = 0;
  const interval = setInterval(() => {
    observer.next(count++);
    if (count > 5) {
      observer.complete();
      clearInterval(interval);
    }
  }, 1000);

  // Функция отписки
  return () => clearInterval(interval);
});

const unsubscribe = observable.subscribe({
  next: (value) => console.log('Value:', value),
  complete: () => console.log('Complete!')
});
```

### Пример 2: Retry механизм

```javascript
class Retryable {
  constructor(operation, maxRetries = 3) {
    this.operation = operation;
    this.maxRetries = maxRetries;
  }

  async execute() {
    let lastError;
    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        return await new Promise(this.operation);
      } catch (error) {
        lastError = error;
        console.log(`Attempt ${attempt + 1} failed:`, error.message);
      }
    }
    throw lastError;
  }
}

// Использование
const unreliableOperation = new Retryable((resolve, reject) => {
  if (Math.random() < 0.7) {
    reject(new Error('Random failure'));
  } else {
    resolve('Success!');
  }
}, 5);

unreliableOperation.execute()
  .then(result => console.log(result))
  .catch(error => console.error('Failed after retries:', error));
```

### Пример 3: Debounced Stream

```javascript
const { Transform } = require('node:stream');

function createDebouncedStream(delay) {
  let timeoutId = null;
  let bufferedChunk = null;

  return new Transform({
    transform(chunk, encoding, callback) {
      // Сбрасываем предыдущий таймер
      if (timeoutId) {
        clearTimeout(timeoutId);
      }

      // Буферизуем chunk
      bufferedChunk = chunk;

      // Устанавливаем новый таймер
      timeoutId = setTimeout(() => {
        this.push(bufferedChunk);
        callback();
      }, delay);
    },

    flush(callback) {
      // Отправляем последний chunk при завершении
      if (timeoutId) {
        clearTimeout(timeoutId);
      }
      if (bufferedChunk) {
        this.push(bufferedChunk);
      }
      callback();
    }
  });
}

// Использование
const debouncedStream = createDebouncedStream(500);
process.stdin
  .pipe(debouncedStream)
  .pipe(process.stdout);
```

---

## Контрольные вопросы

1. В чём основное отличие между передачей константы и callback-функции в конструктор?
2. Почему Promise не может использовать простую функцию вместо callback для инициализации?
3. Какие преимущества даёт Revealing Constructor по сравнению с наследованием классов?
4. Приведите три примера использования этого паттерна в стандартной библиотеке Node.js
5. Как Revealing Constructor связан с паттерном Dependency Injection?

---

**Автор конспекта:** Создано на основе лекции о паттерне Revealing Constructor
**Тема курса:** Паттерны проектирования в Node.js и JavaScript
**Следующая тема:** Паттерн Future для работы с асинхронностью
