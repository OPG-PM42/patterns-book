# Лекция 2. Паттерн Наблюдатель (Observer / Observable)

## Введение

Паттерн Наблюдатель (Observer) — одна из фундаментальных программных абстракций,
лежащая в основе реактивного программирования. Идея проста:

- Есть **источник событий** (Observable / наблюдаемый) — объект, который порождает данные
- Есть **наблюдатель** (Observer) — объект или функция, которые получают эти данные

Обе абстракции могут быть реализованы как через функции и замыкания, так и через классы.
На базе этого паттерна строятся более мощные абстракции, например RxJS.

---

## 2.1 Простейшая функциональная реализация

```javascript
// Пример 1. Observable и Observer на чистых функциях

// Вспомогательная функция: генерирует случайную строчную букву латинского алфавита
// Код символа 'a' = 97, 'z' = 122. Итого 26 символов.
const randomChar = () =>
  String.fromCharCode(97 + Math.floor(Math.random() * 26));

// subscribe — фабричная функция, создающая Observable
// Принимает функцию-наблюдатель (observer) и возвращает объект Observable
const subscribe = (observer) => {
  // Запускаем внутренний таймер — источник событий
  const timer = setInterval(() => {
    // Каждые 200 мс вызываем observer с новой случайной буквой
    observer(randomChar());
  }, 200);

  // Возвращаем объект Observable — он хранит ссылку на подписку
  // и позволяет управлять ею (например, отписаться)
  const observable = {
    observer,    // ссылка на функцию-наблюдатель
    timer,       // ссылка на таймер для возможной отписки
  };

  return observable;
};

// Создаём observer — функцию, которая будет получать данные
let count = 0;
const observer = (char) => {
  process.stdout.write(char); // печатаем букву без переноса строки
  count++;
  if (count % 50 === 0) {
    process.stdout.write('\n');
  }
  if (count >= 150) {
    process.exit(0); // выходим после 150 букв
  }
};

// Выводим структуры для понимания
console.log('Тип observer:', typeof observer);       // function

// Подписываемся — запускаем генерацию букв
const observable = subscribe(observer);
console.log('Тип observable:', typeof observable);   // object
// С этого момента каждые 200 мс будет печататься случайная буква
```

---

## 2.2 Реализация на классах (Observable как класс)

```javascript
// Пример 2. Observable и Observer с использованием классов

const randomDigit = () => Math.floor(Math.random() * 10);

// Класс Observable (наблюдаемый источник событий)
class Observable {
  constructor() {
    // Храним ссылку на одного подписчика (упрощённая реализация)
    // В полноценной реализации здесь был бы массив (collection) наблюдателей
    this.observer = null;
    this.timer = null;
  }

  // Метод подписки — сохраняет наблюдателя и запускает генерацию данных
  subscribe(observer) {
    this.observer = observer;

    // Запускаем таймер — генерируем случайную цифру каждые 200 мс
    this.timer = setInterval(() => {
      if (this.observer) {
        // Если есть подписчик — передаём ему данные
        this.observer(randomDigit());
      } else {
        // Если подписчик отсутствует — останавливаем таймер
        clearInterval(this.timer);
      }
    }, 200);
  }

  // Метод отписки
  unsubscribe() {
    this.observer = null;
    clearInterval(this.timer);
  }
}

// Демонстрация использования
const source = new Observable();

let digitCount = 0;
const digitObserver = (digit) => {
  process.stdout.write(String(digit));
  digitCount++;
  if (digitCount % 50 === 0) process.stdout.write('\n');
  if (digitCount >= 100) {
    source.unsubscribe();
    process.exit(0);
  }
};

source.subscribe(digitObserver);
```

---

## 2.3 Полная реализация на классах с поддержкой нескольких подписчиков

```javascript
// Пример 3. Observable с коллекцией наблюдателей и абстрактными базовыми классами

// Базовый класс Observable (наблюдаемый объект)
// Содержит абстрактные методы — прямое создание экземпляра запрещено
class Observable {
  constructor() {
    // Коллекция подписчиков — поддерживаем несколько наблюдателей
    this.observers = [];
  }

  // Подписываем наблюдателя на поток событий
  subscribe(observer) {
    // Сохраняем обратную ссылку: наблюдатель знает, на что он подписан
    observer.source = this;
    this.observers.push(observer);
    return this; // цепочка вызовов
  }

  // Отписываем наблюдателя
  unsubscribe(observer) {
    this.observers = this.observers.filter((o) => o !== observer);
    return this;
  }

  // Уведомляем всех подписчиков о новых данных
  notify(data) {
    if (this.observers.length === 0) return;
    for (const observer of this.observers) {
      observer.update(data);
    }
  }

  // Абстрактный метод — подкласс должен его переопределить
  // В реальном коде здесь генерируются данные
  start() {
    throw new Error('Метод start() должен быть переопределён в подклассе');
  }
}

// Базовый класс Observer (наблюдатель)
class Observer {
  constructor() {
    this.source = null; // ссылка на Observable, на который подписан
  }

  // Абстрактный метод — вызывается при получении нового события
  update(data) {
    throw new Error('Метод update() должен быть переопределён в подклассе');
  }
}

// Конкретный Observable: генерирует поток случайных букв
class CharStream extends Observable {
  constructor() {
    super();
    this.timer = null;
  }

  start() {
    this.timer = setInterval(() => {
      const char = String.fromCharCode(97 + Math.floor(Math.random() * 26));
      // Уведомляем всех подписчиков о новой букве
      this.notify(char);
    }, 200);
  }

  stop() {
    clearInterval(this.timer);
  }
}

// Конкретный Observer: считает буквы и выводит их на экран
class CharStreamObserver extends Observer {
  constructor(charStream) {
    super();
    this.source = charStream; // ссылка на поток
    this.count = 0;           // счётчик полученных символов
  }

  update(char) {
    process.stdout.write(char);
    this.count++;
    if (this.count % 50 === 0) process.stdout.write('\n');
    if (this.count >= 100) {
      this.source.stop();
      process.exit(0);
    }
  }
}

// Создание и запуск
const charObserver = new CharStreamObserver(null); // source установим ниже
const charStream = new CharStream();

// Подписываем наблюдателя на поток
charStream.subscribe(charObserver);
charObserver.source = charStream; // устанавливаем обратную ссылку

// Запускаем генерацию символов
charStream.start();
```

---

## 2.4 Observable на базе EventEmitter

Вместо реализации собственного механизма подписки можно использовать встроенный
`EventEmitter` из `node:events`.

```javascript
// Пример 4. Observable, использующий EventEmitter как механизм подписки

import { EventEmitter } from 'node:events';

const randomChar = () =>
  String.fromCharCode(97 + Math.floor(Math.random() * 26));

// CharStream — наблюдаемый поток, делегирующий подписку EventEmitter
class CharStream {
  constructor(emitter) {
    // EventEmitter приходит снаружи (Dependency Injection)
    // CharStream не создаёт его сам — это снижает зацепление
    this.emitter = emitter;
    this.timer = null;
  }

  start() {
    this.timer = setInterval(() => {
      const char = randomChar();
      // Генерируем событие 'char' через EventEmitter
      this.emitter.emit('char', char);
    }, 200);
  }

  complete() {
    clearInterval(this.timer);
    this.emitter.emit('complete');
  }
}

// Создаём EventEmitter — механизм подписки
const emitter = new EventEmitter();

// Создаём CharStream, передавая ему emitter
const stream = new CharStream(emitter);

// Observer подписывается через EventEmitter
let count = 0;
emitter.on('char', (char) => {
  process.stdout.write(char);
  count++;
  if (count % 50 === 0) process.stdout.write('\n');
  if (count >= 100) {
    stream.complete();
  }
});

emitter.on('complete', () => {
  console.log('\nПоток завершён');
  process.exit(0);
});

stream.start();
```

---

## 2.5 Операторы: filter и map для потока событий

Реактивное программирование обогащает паттерн Observer **операторами** — функциями,
которые преобразуют поток событий, порождая новый Observable.

```javascript
// Пример 5. Observable с операторами filter и map

const randomChar = () =>
  String.fromCharCode(97 + Math.floor(Math.random() * 26));

// Гласные английского языка
const VOWELS = new Set(['a', 'e', 'i', 'o', 'u']);

// Функции-операторы возвращают объекты-дескрипторы операций
// Это похоже на паттерн Команда — операция представлена как объект
const filter = (predicate) => ({ type: 'filter', fn: predicate });
const map = (transform) => ({ type: 'map', fn: transform });

class Observable {
  constructor() {
    this.observers = [];
    this.operators = []; // список операторов, применяемых к данным
  }

  subscribe(observer) {
    this.observers.push(observer);
    return this;
  }

  // Метод pipe — применяет операторы и возвращает новый Observable
  // Все аргументы — операторы (объекты с type и fn)
  pipe(...operators) {
    const destination = new Observable();
    destination.operators = operators;

    // Подписываем destination на текущий Observable
    this.subscribe({
      update: (data) => destination.notify(data),
    });

    return destination; // возвращаем преобразованный поток
  }

  // Уведомляем подписчиков, пропуская данные через все операторы
  notify(data) {
    if (this.observers.length === 0) return;

    // Применяем каждый оператор последовательно
    let value = data;
    let skip = false;

    for (const operator of this.operators) {
      if (operator.type === 'filter') {
        // Если предикат возвращает false — пропускаем это значение
        if (!operator.fn(value)) {
          skip = true;
          break;
        }
      } else if (operator.type === 'map') {
        // Преобразуем значение
        value = operator.fn(value);
      }
    }

    if (!skip) {
      for (const observer of this.observers) {
        observer.update(value);
      }
    }
  }
}

// Источник: поток случайных букв
const source = new Observable();

// Создаём преобразованный поток:
// 1. Фильтруем — оставляем только согласные (не гласные)
// 2. Преобразуем — делаем буквы заглавными
const consonantsUpperCase = source.pipe(
  filter((char) => !VOWELS.has(char)),  // только согласные
  map((char) => char.toUpperCase()),    // в верхний регистр
);

// Observer: считает и печатает согласные заглавные буквы
let count = 0;
consonantsUpperCase.subscribe({
  update(char) {
    process.stdout.write(char);
    count++;
    if (count % 50 === 0) process.stdout.write('\n');
    if (count >= 100) {
      console.log('\nГотово!');
      process.exit(0);
    }
  },
});

// Запускаем генерацию случайных букв
const timer = setInterval(() => {
  const char = randomChar();
  source.notify(char);
}, 200);
```

---

## 2.6 Паттерн Revealing Constructor в Observable

Паттерн "Открытый конструктор" (Revealing Constructor) — это когда в конструктор
передаётся функция, которой Observable "открывает" свои внутренние возможности.

```javascript
// Пример 6. Observable с паттерном Revealing Constructor
// Логика генерации событий передаётся прямо в конструктор

const randomChar = () =>
  String.fromCharCode(97 + Math.floor(Math.random() * 26));

const VOWELS = new Set(['a', 'e', 'i', 'o', 'u']);

class Observable {
  constructor(executor) {
    this.observers = [];
    this.operators = [];

    // executor — функция, которую мы передаём в конструктор
    // Она получает функцию next — способ отправить данные в поток
    // Это позволяет "инжектировать" логику генерации внутрь Observable
    if (typeof executor === 'function') {
      executor((data) => this.notify(data));
    }
  }

  subscribe(observer) {
    this.observers.push(observer);
    return this;
  }

  pipe(...operators) {
    const destination = new Observable();
    destination.operators = operators;
    this.subscribe({ update: (data) => destination.notify(data) });
    return destination;
  }

  notify(data) {
    if (this.observers.length === 0) return;
    let value = data;
    for (const op of this.operators) {
      if (op.type === 'filter') {
        if (!op.fn(value)) return;
      } else if (op.type === 'map') {
        value = op.fn(value);
      }
    }
    for (const observer of this.observers) {
      observer.update(value);
    }
  }
}

const filter = (predicate) => ({ type: 'filter', fn: predicate });
const map = (transform) => ({ type: 'map', fn: transform });

// Создаём Observable через Revealing Constructor:
// логика генерации букв "инжектируется" через параметр executor
const charStream = new Observable((next) => {
  // next — функция для отправки данных в поток
  // Таймер теперь живёт ВНУТРИ Observable
  setInterval(() => {
    next(randomChar());
  }, 200);
});

// Применяем операторы и подписываемся
let count = 0;
charStream
  .pipe(
    filter((char) => !VOWELS.has(char)),
    map((char) => char.toUpperCase()),
  )
  .subscribe({
    update(char) {
      process.stdout.write(char);
      count++;
      if (count % 50 === 0) process.stdout.write('\n');
      if (count >= 100) process.exit(0);
    },
  });
```

---

## 2.7 Сравнение реализаций паттерна Observer

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Функции и замыкания | Краткость, простота | Меньше возможностей для расширения |
| Классы (Observer + Observable) | Явная структура, наследование | Больше кода, шаблонность |
| EventEmitter | Готовый механизм, не нужно писать самому | Завязан на Node.js API |
| Revealing Constructor | Логика инкапсулирована, гибко | Сложнее читать поначалу |
