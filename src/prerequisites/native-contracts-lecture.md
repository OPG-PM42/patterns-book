# JavaScript Native Primitives

## Встроенные контракты JavaScript

### Результаты опроса понимания контрактов

- **Callable** - 43% понимают
- **Thenable** - 60% понимают
- **Promise** - 91% понимают (это понятно)
- **Date-able** - 70% понимают
- **Async Iterator** - 52% понимают
- **Array-like** - примерно 45%
- **Observable** - примерно 45%
- **Stream** - примерно 45%
- **Transferable** - 8% понимают (наиболее неизвестная штука)
- **Callback (error-first)** - 52% понимают

Практически всё, кроме callable и transferable, на слуху. Но нужны примеры и определения. Как для всех паттернов теперь есть индекс с определениями, такие же индексы появятся для контрактов асинхронного программирования и встроенных контрактов JavaScript.

---

## Что такое Callable (вызываемое)

### Определение

**Callable** - это всё, что может быть вызвано как функция. Это контракт, который немножечко сложнее, чем может показаться. Слово "callable" очень редко используется в мире JavaScript, хотя это фундаментальное понятие.

### Что может быть callable?

Вызвано может быть не только функция! Все, что можно вызвать как функцию - это callable:

1. **Обычная функция**
2. **Объект через Proxy** с ловушкой `apply`

### Особенность callable

Это единственная штука, которую невозможно объявить у объекта напрямую. Мы не можем переопределить метод `call` у объекта в JavaScript. Но если мы унаследуемся от класса `Function`, экземпляр этого класса будет callable.

### Примеры кода

#### Пример 1: Обычная функция

```javascript
// Just function
const callable = (...pars) => {
  const { length } = pars;
  return length;
};
```

#### Пример 2: Built-in JavaScript Proxy

```javascript
const proxy = new Proxy(callable, {
  apply: (target, context, args) => {
    console.log('call', target.name, context, args); // "call" "callable" undefined // [object Array] (3) [1,2,3]
    const res = target(...args);
    console.log('return', res); // "return" 3
    return res;
  },
});

const argCount = proxy(1, 2, 3);
console.log({ argCount }); // { "argCount": 3 }
```

### Важные моменты про Proxy.apply

- **target** - оригинальная функция
- **context** - контекст вызова
- **args** - массив аргументов (это именно массив, а не array-like)

Перехват `apply` - это единственный способ сделать обычный объект callable без наследования от `Function`.

---

## Thenable (обещаемое)

### Определение

**Thenable** - это объект, у которого есть метод `then`. На базе этого контракта разработан Promise и совместимые с Promise конструкции, которые умеют взаимодействовать с async/await.

### Структура thenable

```javascript
const thenable = {
  then(onFulfilled, onRejected) {
    // Логика выполнения
    // onFulfilled вызывается при успехе
    // onRejected вызывается при ошибке
  }
};
```

### Примеры кода

#### Пример: Thenable

```javascript
const numbers = {
  then(onFulfilled, onRejected) {
    console.log({ onFulfilled, onRejected }); // { "onFulfilled": function () { [native code] }, "onRejected": undefined  }
    const array = [1, 2, 3];
    onFulfilled(array); // [1,2,3]
    return numbers;
  },
};

const main = async () => {
  numbers.then(console.log); 
  const res = await numbers;
  console.log({ res });  // { "res": [ 1, 2, 3 ]  }
};

main();
```

### Связь с Promise

Promise - это стандартная реализация thenable контракта с дополнительными методами (`catch`, `finally`) и гарантиями выполнения.

---

## Iterable (перебираемое)

### Определение

**Iterable** - это объект, который можно перебрать в цикле `for...of`. Объект становится iterable, когда реализует метод `[Symbol.iterator]()`, возвращающий iterator.

### Структура iterable

```javascript
const iterable = {
  [Symbol.iterator]() {
    // Возвращает iterator с методом next()
    return {
      next() {
        return {
          value: /* значение */,
          done: /* true/false */
        };
      }
    };
  }
};
```

### Примеры кода

#### Пример 1: Простой iterable

```javascript
// Простой iterable объект
const range = (start, end) => ({
  [Symbol.iterator]() {
    let current = start;
    return {
      next() {
        if (current <= end) {
          return { value: current++, done: false };
        }
        return { done: true };
      }
    };
  }
});

// Использование
for (const num of range(1, 5)) {
  console.log(num); // 1, 2, 3, 4, 5
}
```

#### Пример 2: Iterable коллекция

```javascript
// Iterable коллекция
class Collection {
  constructor() {
    this.items = [];
  }

  add(item) {
    this.items.push(item);
  }

  [Symbol.iterator]() {
    let index = 0;
    const items = this.items;
    
    return {
      next() {
        if (index < items.length) {
          return { value: items[index++], done: false };
        }
        return { done: true };
      }
    };
  }
}

// Использование
const collection = new Collection();
collection.add('apple');
collection.add('banana');
collection.add('orange');

for (const item of collection) {
  console.log(item);
}
// apple
// banana
// orange
```

#### Пример 3: Iterable

```javascript
const DONE = { done: true, value: undefined };

const iterable = {
  [Symbol.iterator]() {
    let i = 0;
    const iterator = {
      next() {
        return {
          value: i++,
          done: i > 3,
        };
      },

      throw(error) {
        console.log('throws:', error.message);
        return DONE;
      },

      return(value) {
        console.log('returns', value);
        return DONE;
      },
    };
    return iterator;
  },
};

const fn = () => {
  for (const value of iterable) {
    console.log({ value }); // { "value": 0 }
    //throw new Error('Hello');
    //return;
    break;
  }
};

fn();
```

---

## Async Iterator (асинхронный итератор)

### Определение

**Async Iterator** - это итератор, который работает с асинхронными данными. Реализует метод `[Symbol.asyncIterator]()` и возвращает promises из метода `next()`.

### Структура async iterator

```javascript
const asyncIterable = {
  [Symbol.asyncIterator]() {
    return {
      async next() {
        return {
          value: /* значение */,
          done: /* true/false */
        };
      }
    };
  }
};
```

### Примеры кода

#### Пример:

```javascript
const DONE = { done: true, value: undefined };

const iterable = {
  [Symbol.asyncIterator]() {
    let i = 0;
    const iterator = {
      async next() {
        return {
          value: i++,
          done: i > 3,
        };
      },

      async throw(error) {
        console.log('throws:', error.message);
        return DONE;
      },

      async return(value) {
        console.log('returns', value); // "returns" undefined
        return DONE;
      },
    };
    return iterator;
  },
};

const fn = async () => {
  for await (const value of iterable) {
    console.log({ value }); // { "value": 0 }
    throw new Error('Hello');
    //return;
    //break;
  }
};

fn();
```

---

## Array-like (псевдомассив)

### Определение

**Array-like** - это объект, который выглядит как массив: имеет числовые индексы и свойство `length`, но не является настоящим массивом и не имеет методов Array.prototype.

### Структура array-like

```javascript
const arrayLike = {
  0: 'first',
  1: 'second',
  2: 'third',
  length: 3
};
```

### Примеры кода

#### Пример 1: Создание array-like объекта

```javascript
// Простой array-like объект
const createArrayLike = (...items) => {
  const obj = { length: items.length };
  items.forEach((item, index) => {
    obj[index] = item;
  });
  return obj;
};

const arrayLike = createArrayLike('a', 'b', 'c');
console.log(arrayLike); // { 0: 'a', 1: 'b', 2: 'c', length: 3 }
console.log(arrayLike[1]); // 'b'
console.log(arrayLike.length); // 3
```

#### Пример 2: Преобразование в настоящий массив

```javascript
// Преобразование array-like в массив
const arrayLike = {
  0: 10,
  1: 20,
  2: 30,
  length: 3
};

// Способ 1: Array.from()
const arr1 = Array.from(arrayLike);
console.log(arr1); // [10, 20, 30]

// Способ 2: Spread operator
const arr2 = [...arrayLike]; // Не работает для array-like без итератора

// Способ 3: Array.prototype.slice
const arr3 = Array.prototype.slice.call(arrayLike);
console.log(arr3); // [10, 20, 30]
```

#### Пример 3: Классический array-like на классах

```javascript
// Class

class ArrayLike {
  constructor(...items) {
    for (let i = 0; i < items.length; i++) {
      this[i] = items[i];
    }
    this.length = items.length;
  }
}

const instance = new ArrayLike('a', 'b'); // {"0": "a","1": "b","length": 2}
console.log(Array.from(instance)); // ["a","b"]

// Arguments

const f = function () {
  console.log(arguments);
  console.log(arguments.length);
};

f('a', 'b');
```

---

## Observable (наблюдаемое)

### Определение

**Observable** - это объект, который представляет поток данных или событий, на которые можно подписаться. Это паттерн для реактивного программирования.

### Структура observable

```javascript
const observable = {
  subscribe(observer) {
    // observer - объект с методами: next, error, complete
    // Возвращает объект subscription с методом unsubscribe
  }
};
```

### Примеры кода

#### Пример 1: Простой Observable

```javascript
// Базовая реализация Observable
class Observable {
  constructor(subscribe) {
    this._subscribe = subscribe;
  }

  subscribe(observer) {
    return this._subscribe(observer);
  }
}

// Создание observable
const observable = new Observable((observer) => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  
  setTimeout(() => {
    observer.next(4);
    observer.complete();
  }, 1000);

  // Функция отписки
  return {
    unsubscribe() {
      console.log('Unsubscribed');
    }
  };
});

// Подписка
const subscription = observable.subscribe({
  next(value) {
    console.log('Next:', value);
  },
  error(err) {
    console.error('Error:', err);
  },
  complete() {
    console.log('Complete');
  }
});

// Отписка через 2 секунды
setTimeout(() => {
  subscription.unsubscribe();
}, 2000);
```

#### Пример 2: Observable для событий

```javascript
const observable = new EventTarget();

observable.addEventListener('buy', (event) => {
  const bought = event.detail;
  console.log({ bought }); // { "bought": { "name": "Laptop", "price":1500 } etc.
}
});

const electronics = [
  { name: 'Laptop', price: 1500 },
  { name: 'Keyboard', price: 100 },
  { name: 'HDMI cable', price: 10 },
];

for (const item of electronics) {
  const data = { detail: item };
  const event = new CustomEvent('buy', data);
  observable.dispatchEvent(event);
}
```

---

## Stream (поток)

### Определение

**Stream** - это абстракция для работы с потоковыми данными. В Node.js это события, которые позволяют читать или записывать данные частями, не загружая всё в память.

### Типы стримов в Node.js

1. **Readable** - поток чтения
2. **Writable** - поток записи
3. **Duplex** - двунаправленный поток (чтение и запись)
4. **Transform** - трансформирующий поток (модификация данных)

### Примеры кода

#### Пример 1: Stream

```javascript
const URL = 'https://developer.mozilla.org/';

const main = async () => {
  const response = await fetch(URL);
  const { body } = response;
  let total = 0;

  for await (const chunk of body) {
    const name = chunk.constructor.name;
    const { length } = chunk;
    total += length;
    console.log(`Chunk: ${name}, Length: ${length}`);
  }

  console.log(`Total length: ${total} bytes`);
};

main();
```
---

## Transferable (передаваемое)

### Определение

**Transferable** - это объекты, которые могут быть переданы между Web Workers или Worker threads с передачей владения, а не копированием. После передачи оригинальный объект становится недоступен.

### Transferable объекты

- `ArrayBuffer`
- `MessagePort`
- `ImageBitmap`
- `OffscreenCanvas`

### Примеры кода

#### Пример 1: Transferable 

```javascript
const { Worker, workerData, isMainThread } = require('worker_threads');

if (isMainThread) {
  const workerData = new ArrayBuffer(8);
  const uint32 = new Uint32Array(workerData);
  uint32[0] = 42;
  console.log('Main: before transfer', uint32);

  const transferList = [workerData]; // Mark buffer as transferable
  const worker = new Worker(__filename, { workerData, transferList });

  worker.on('exit', () => {
    console.log('Main: after transfer', uint32);
  });
} else {
  const uint32 = new Uint32Array(workerData);
  console.log('Worker: received buffer', uint32);
  uint32[0] += 1;
}
```

---

## Callback и Error-First Callback

### Определение

**Callback** - это функция, которая передаётся в другую функцию для последующего вызова.

**Error-First Callback** - это соглашение Node.js, где первый аргумент callback всегда ошибка (или null), а последующие - данные.

### Примеры кода

#### Пример 1: Callback-last-error-first

```javascript
const fs = require('node:fs');

const callback = (error, data) => {
  if (error) {
    console.error(error.message);
  } else {
    console.log(data);
  }
};

fs.readFile('1-callback.js', 'utf8', callback);
fs.readFile('unknown-file', 'utf8', callback);
```

---

## Пререквизиты для курсов

### Базовые концепции кода

Что нужно понимать для всех курсов:

1. **Чистая функция** - функция без побочных эффектов
2. **Замыкание** - доступ к переменным внешней области видимости
3. **Колбэк** - функция, переданная как аргумент
4. **Композиция функций** - создание новых функций из существующих
5. **Композиция классов** - комбинирование возможностей классов
6. **Лексический контекст** - область видимости переменных
7. **Форма объектов** - структура объектов для оптимизации
8. **Мономорфный код** - код с предсказуемыми типами
9. **Инлайнинг** - встраивание кода для оптимизации

## Заключение

Встроенные контракты JavaScript - это фундамент, необходимый для понимания:
- Паттернов проектирования
- Асинхронного программирования
- Node.js разработки

Каждый контракт имеет чёткое определение, примеры и область применения. Понимание этих контрактов позволяет писать более качественный и поддерживаемый код.

---

## Полезные ссылки

- **Репозиторий с примерами:** https://github.com/HowProgrammingWorks/NativeContracts
- **Thenable:** https://github.com/HowProgrammingWorks/Thenable
- **YouTube канал:** How Programming Works
- **Telegram канал:** @metarhia

---