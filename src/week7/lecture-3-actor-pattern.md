# Паттерн Actor — безопасное управление состоянием в асинхронном коде

---

## Введение

Паттерн Actor (Актор) решает одну из наиболее коварных проблем асинхронного
программирования: гонки данных (data races). В отличие от многопоточных языков (C++, Java),
в JavaScript нет разделяемой памяти между потоками выполнения (за исключением
`SharedArrayBuffer`). Тем не менее гонки данных в JavaScript реальны и проявляются
в асинхронном коде при параллельных вызовах функций, работающих с общим состоянием.

---

## 3.1 Проблема: гонки данных в асинхронном JavaScript

```javascript
// Пример 18. Демонстрация гонки данных — ОПАСНЫЙ КОД (антипаттерн)
//
// Задача: склад с 5 единицами товара (артикул 1722).
// Три одновременных заказа по 2 единицы.
// Ожидаемый результат: первые два заказа выполнены, третий отклонён.
// Реальный результат: все три проходят, счётчик уходит в -1.
//
// Почему: каждый buy() читает стейт ДО того, как другие buy() успели его изменить.
// Event Loop — не защита от гонок. Он предотвращает параллельное ВЫПОЛНЕНИЕ,
// но не параллельное ЧЕРЕДОВАНИЕ асинхронных операций.

'use strict';

// Состояние склада
const warehouseState = { 1722: 5 };

// Имитация асинхронных бизнес-операций

async function checkAvailability(order, state) {
  // Имитируем запрос к БД
  await new Promise((r) => setTimeout(r, Math.random() * 10));
  if (state[order.lot] < order.qty) {
    throw new Error(`Недостаточно товара: есть ${state[order.lot]}, запрошено ${order.qty}`);
  }
  console.log(`[${order.id}] Проверка наличия ОК: ${state[order.lot]} >= ${order.qty}`);
}

async function processPayment(order) {
  await new Promise((r) => setTimeout(r, Math.random() * 15));
  console.log(`[${order.id}] Платёж проведён на сумму ${order.qty * 1000} руб.`);
}

async function shiftGoods(order, state) {
  await new Promise((r) => setTimeout(r, Math.random() * 5));
  state[order.lot] -= order.qty;
  console.log(`[${order.id}] Товар списан. Остаток: ${state[order.lot]}`);
}

async function sendNotification(order) {
  await new Promise((r) => setTimeout(r, 5));
  console.log(`[${order.id}] Уведомление отправлено`);
}

// Функция бизнес-логики покупки
async function buy(order, state) {
  await checkAvailability(order, state); // <- ТОЧКА ГОНКИ: читаем стейт
  await processPayment(order);           // <- другие buy() тоже дошли сюда
  await shiftGoods(order, state);        // <- все три декрементируют!
  await sendNotification(order);
}

// Три одновременных заказа
const orders = [
  { id: 'ORDER-1', lot: 1722, qty: 2 },
  { id: 'ORDER-2', lot: 1722, qty: 2 },
  { id: 'ORDER-3', lot: 1722, qty: 2 },
];

async function runRace() {
  console.log('=== ДЕМОНСТРАЦИЯ ГОНКИ ДАННЫХ ===');
  console.log(`Начальный остаток: ${warehouseState[1722]}\n`);

  // Запускаем все три одновременно через setTimeout с одинаковой задержкой
  // Это имитирует три HTTP-запроса, поступивших в одно время
  const promises = orders.map(
    (order) => new Promise((resolve) => setTimeout(() => resolve(buy(order, warehouseState)), 0))
  );

  const results = await Promise.allSettled(promises);

  console.log('\n=== РЕЗУЛЬТАТ ===');
  results.forEach((result, i) => {
    if (result.status === 'fulfilled') {
      console.log(`ORDER-${i + 1}: ВЫПОЛНЕН`);
    } else {
      console.log(`ORDER-${i + 1}: ОШИБКА — ${result.reason.message}`);
    }
  });

  // ПРОБЛЕМА: финальный остаток может быть отрицательным!
  console.log(`\nФинальный остаток: ${warehouseState[1722]}`);
  console.log('(Ожидалось: 1, может быть -1 или другое отрицательное число)');
}

runRace();
```

**Разбор механизма гонки:**

Event Loop в JavaScript однопоточный — два колбэка не выполняются буквально одновременно.
Однако `async/await` — это синтаксический сахар над промисами, которые ставятся в очередь
микрозадач. Последовательность выполнения:

1. `ORDER-1.buy()` → `checkAvailability()` — видит остаток 5, ОК
2. `ORDER-2.buy()` → `checkAvailability()` — видит остаток 5, ОК (декремент ещё не произошёл!)
3. `ORDER-3.buy()` → `checkAvailability()` — видит остаток 5, ОК
4. Все три прошли проверку и перешли к `processPayment`
5. Все три прошли к `shiftGoods` и каждый уменьшил остаток на 2
6. Итог: 5 - 2 - 2 - 2 = -1

---

## 3.2 Решение: Паттерн Actor

```javascript
// Пример 19. Реализация паттерна Actor
//
// Actor — объект с единственной публичной точкой входа (send),
// который внутри организует очередь сообщений (mailbox).
// Каждое сообщение обрабатывается строго последовательно с await.
//
// Ключевые элементы:
//   #queue      — массив-очередь сообщений (FIFO)
//   #processing — флаг активного исполнения (предотвращает повторный запуск #process)
//   #behavior   — функция бизнес-логики (передаётся через конструктор)
//   #state      — состояние, принадлежащее только этому актору
//
// Паттерн "Открытый конструктор" (Revealing Constructor):
//   behavior передаётся в конструктор — актор не знает о конкретной логике,
//   он только гарантирует последовательность обработки.
//
// Историческая справка:
//   Когда Алан Кей изобретал ООП, он представлял методы именно как "отправку сообщений".
//   Вызов метода = отправка сообщения на экземпляр. Actor реализует эту идею буквально:
//   есть объект (Actor), есть почтовый ящик (#queue), есть поведение (#behavior).

'use strict';

class Actor {
  #behavior;  // функция: (message, state) => Promise<void>
  #state;     // изолированное состояние актора
  #queue = [];           // очередь входящих сообщений (mailbox)
  #processing = false;   // флаг: идёт ли сейчас обработка

  constructor(behavior, initialState) {
    this.#behavior = behavior;
    this.#state = initialState;
  }

  // Единственный публичный метод: отправить сообщение актору
  // "fire and forget" — мы не ждём результата (нет return Promise)
  send(message) {
    this.#queue.push(message);
    // Запускаем обработку (если ещё не запущена)
    this.#process();
  }

  // Приватный цикл обработки сообщений
  async #process() {
    // Если уже обрабатываем — выходим (другой вызов #process уже крутит цикл)
    if (this.#processing) return;

    this.#processing = true;

    // Обрабатываем сообщения по одному, пока очередь не опустеет
    while (this.#queue.length > 0) {
      const message = this.#queue.shift(); // берём первое сообщение из очереди (FIFO)
      try {
        // await гарантирует: следующее сообщение не начнётся, пока это не завершится
        await this.#behavior(message, this.#state);
      } catch (err) {
        // Ошибка в behavior не останавливает обработку других сообщений
        console.error(`Actor: ошибка при обработке сообщения:`, err.message);
      }
    }

    this.#processing = false;
    // После обнуления флага актор готов принять новые сообщения
  }
}

module.exports = { Actor };
```

---

## 3.3 Применение Actor к задаче со складом

```javascript
// Пример 20. Склад с гонкой данных — ИСПРАВЛЕННАЯ версия с Actor
//
// Единственное изменение по сравнению с опасным кодом:
// вместо прямого вызова buy(order, state) отправляем сообщение актору.
// Бизнес-логика (buy) не изменилась.

'use strict';

// --- Та же бизнес-логика что в примере 18 ---

async function checkAvailability(order, state) {
  await new Promise((r) => setTimeout(r, Math.random() * 10));
  if (state[order.lot] < order.qty) {
    throw new Error(`Недостаточно товара: есть ${state[order.lot]}, запрошено ${order.qty}`);
  }
  console.log(`[${order.id}] Проверка наличия ОК: ${state[order.lot]} >= ${order.qty}`);
}

async function processPayment(order) {
  await new Promise((r) => setTimeout(r, Math.random() * 15));
  console.log(`[${order.id}] Платёж проведён`);
}

async function shiftGoods(order, state) {
  await new Promise((r) => setTimeout(r, Math.random() * 5));
  state[order.lot] -= order.qty;
  console.log(`[${order.id}] Товар списан. Остаток: ${state[order.lot]}`);
}

async function sendNotification(order) {
  await new Promise((r) => setTimeout(r, 5));
  console.log(`[${order.id}] Уведомление отправлено`);
}

// Функция бизнес-логики — НЕ ИЗМЕНИЛАСЬ
async function buy(order, state) {
  await checkAvailability(order, state);
  await processPayment(order);
  await shiftGoods(order, state);
  await sendNotification(order);
}

// --- Реализация Actor ---

class Actor {
  #behavior;
  #state;
  #queue = [];
  #processing = false;

  constructor(behavior, initialState) {
    this.#behavior = behavior;
    this.#state = initialState;
  }

  send(message) {
    this.#queue.push(message);
    this.#process();
  }

  async #process() {
    if (this.#processing) return;
    this.#processing = true;
    while (this.#queue.length > 0) {
      const message = this.#queue.shift();
      try {
        await this.#behavior(message, this.#state);
      } catch (err) {
        console.error(`[${message.id}] ОШИБКА: ${err.message}`);
      }
    }
    this.#processing = false;
  }
}

// --- Применение ---

async function runWithActor() {
  console.log('=== ACTOR: БЕЗОПАСНАЯ ОБРАБОТКА ЗАКАЗОВ ===');

  // Оборачиваем состояние и поведение в актор
  // Состояние изолировано внутри актора — снаружи к нему нет доступа
  const warehouseActor = new Actor(buy, { 1722: 5 });

  console.log('Начальный остаток: 5\n');

  const orders = [
    { id: 'ORDER-1', lot: 1722, qty: 2 },
    { id: 'ORDER-2', lot: 1722, qty: 2 },
    { id: 'ORDER-3', lot: 1722, qty: 2 },
  ];

  // Отправляем все три сообщения "одновременно"
  // Actor поставит их в очередь и обработает последовательно
  for (const order of orders) {
    warehouseActor.send(order);
  }

  // Ждём завершения всех обработок (даём время event loop)
  await new Promise((r) => setTimeout(r, 500));

  console.log('\n=== ИТОГ ===');
  console.log('Все три сообщения обработаны последовательно.');
  console.log('Остаток не ушёл в минус. Гонка данных устранена.');
}

runWithActor();
```

---

## 3.4 Гарантии паттерна Actor

| Гарантия | Описание | Механизм |
|----------|----------|---------|
| Последовательность | Сообщения обрабатываются по одному | `while` + `await` + флаг `#processing` |
| Изоляция стейта | Состояние недоступно снаружи | Приватное поле `#state` |
| Отказоустойчивость | Ошибка одного сообщения не ломает очередь | `try/catch` в `#process` |
| Прозрачность | Бизнес-логика не меняется | Поведение передаётся через конструктор |
| Без блокировок | Нет mutex/lock/semaphore | Только очередь и async/await |

---

## 3.5 Типичная ошибка: утечка ссылки на состояние

```javascript
// Пример 21. Антипаттерн: внешняя ссылка на состояние нарушает гарантии Actor
//
// Если передать ссылку на state наружу, кто угодно сможет изменить его напрямую,
// минуя очередь актора. Изоляция нарушена — гонка возвращается.

'use strict';

class UnsafeActor {
  #state;
  #queue = [];
  #processing = false;

  constructor(initialState) {
    this.#state = initialState;
    // АНТИПАТТЕРН: публичный геттер возвращает ссылку на изменяемый объект
    // Внешний код может изменить this.state[key] напрямую, минуя очередь
    this.state = this.#state; // <-- ОПАСНО
  }

  send(behavior, message) {
    this.#queue.push({ behavior, message });
    this.#process();
  }

  async #process() {
    if (this.#processing) return;
    this.#processing = true;
    while (this.#queue.length > 0) {
      const { behavior, message } = this.#queue.shift();
      await behavior(message, this.#state);
    }
    this.#processing = false;
  }
}

const actor = new UnsafeActor({ counter: 0 });

// Внешний код напрямую модифицирует состояние, минуя очередь:
actor.state.counter = 999; // <-- ГОНКА ДАННЫХ ВОЗМОЖНА

console.log('АНТИПАТТЕРН: внешняя запись в actor.state обходит очередь');
console.log('Всегда используйте приватные поля (#state) и не возвращайте ссылку на state наружу');
```

---

## 3.6 Связи с другими концепциями

### Паттерн Actor и Event Loop

Event Loop в Node.js не защищает от гонок данных сам по себе. Он гарантирует, что
в любой момент времени выполняется только один участок синхронного кода. Но `await`
создаёт точку отказа от управления: между двумя `await` другой асинхронный код может
начать выполнение и изменить общий стейт.

Actor решает это иначе: он не запрещает другим запуститься, но гарантирует, что к
данному стейту будет обращаться только одно сообщение за раз, потому что следующее
сообщение не начнётся до `await` в конце предыдущего.

### Паттерн "Открытый конструктор" во всех трёх темах

Все три темы недели 7 используют один и тот же паттерн — передача поведения через
конструктор (Revealing Constructor Pattern):

```javascript
// Node.js Writable: поведение записи — через конструктор
const writable = new Writable({
  write(chunk, encoding, next) { /* логика */ }
});

// Web Streams WritableStream: то же самое в Web API
const ws = new WritableStream({
  write(chunk) { /* логика */ }
});

// Actor: поведение обработки сообщений — через конструктор
const actor = new Actor(behaviorFunction, initialState);
```

Это не случайное совпадение. Паттерн Revealing Constructor — это способ создать объект
с инкапсулированным состоянием, поведение которого задаётся в момент создания. Он
противоположен субклассированию: вместо `extends Transform` мы передаём функцию.

### for await как унифицированный интерфейс

```javascript
// Node.js Readable Stream
for await (const chunk of nodeStream) { ... }

// Web API ReadableStream (response.body из fetch)
for await (const chunk of response.body) { ... }

// HTTP IncomingMessage (является Readable Stream)
for await (const chunk of req) { ... }

// Любой объект с Symbol.asyncIterator
for await (const item of asyncGenerator()) { ... }
```

Это принципиальное решение стандартного комитета TC39 и Node.js: унифицировать
доступ ко всем источникам асинхронных данных через один синтаксис.

---

## 3.7 Сравнение подходов к защите состояния

| Подход | Механизм | Сложность | Применимость |
|--------|----------|-----------|--------------|
| Последовательные `await` | `await buy1; await buy2` | Низкая | Один исполнитель |
| Mutex / Semaphore | Блокировки (библиотеки) | Высокая | Мультипоточность (Worker) |
| Actor Pattern | Очередь + async/await | Низкая | Любой async JS |
| Immutable state | Неизменяемые структуры | Средняя | Функциональный стиль |
| SharedArrayBuffer + Atomics | Атомарные операции | Очень высокая | SharedArrayBuffer |

Для большинства задач в Node.js паттерн Actor — наиболее практичное решение:
он не требует сторонних библиотек, работает со стандартным async/await и не
усложняет бизнес-логику.

---

## Предупреждения и типичные ошибки

### Стримы

**Забытый обработчик ошибок**
```javascript
// НЕПРАВИЛЬНО: без обработки ошибок Node.js упадёт
const stream = fs.createReadStream('./nonexistent.txt');
stream.on('data', (chunk) => { /* ... */ });
// Если файл не существует — необработанное исключение!

// ПРАВИЛЬНО: всегда добавляем on('error')
stream.on('error', (err) => console.error('Ошибка стрима:', err.message));
```

**Использование pipe вместо pipeline**
```javascript
// НЕПРАВИЛЬНО: pipe не обрабатывает ошибки автоматически
readable.pipe(transform).pipe(writable);
// Если transform выдаст ошибку — readable не будет закрыт

// ПРАВИЛЬНО: pipeline корректно обрабатывает ошибки и закрывает все стримы
const { pipeline } = require('node:stream/promises');
await pipeline(readable, transform, writable);
```

**Смешивание on('data') и on('readable')**
```javascript
// НЕПРАВИЛЬНО: подписка на оба события создаёт непредсказуемое поведение
stream.on('data', handler1);    // переводит в flowing mode
stream.on('readable', handler2); // конфликт режимов

// ПРАВИЛЬНО: используйте один стиль чтения
```

**null в стриме объектов**
```javascript
// НЕПРАВИЛЬНО: null — зарезервированный сигнал конца стрима
const stream = new Readable({ read() {} });
stream.push(null);         // <- КОНЕЦ СТРИМА, не данные!
stream.push({ id: null }); // НИКОГДА не выполнится

// ПРАВИЛЬНО: используйте sentinel-объект или objectMode с явным закрытием
```

### Actor Pattern

**Разделённый стейт между акторами**
```javascript
// НЕПРАВИЛЬНО: оба актора работают с одним объектом
const shared = { count: 0 };
const actor1 = new Actor(behavior1, shared);
const actor2 = new Actor(behavior2, shared); // ГОНКА ДАННЫХ!

// ПРАВИЛЬНО: каждый актор владеет своим стейтом
const actor1 = new Actor(behavior1, { count: 0 });
const actor2 = new Actor(behavior2, { count: 0 });
```

**Ожидание результата через send()**
```javascript
// НЕПРАВИЛЬНО: Actor.send() — fire and forget, нет возврата результата
const result = actor.send(message); // result === undefined

// ПРАВИЛЬНО: если нужен результат — используйте callback внутри сообщения
// или расширяйте Actor паттерн возвратом Promise из send()
actor.send({ ...message, onComplete: (result) => { /* ... */ } });
```

---

## Управление памятью через стримы

Ключевое преимущество стримов — данные не загружаются в память целиком.
При чтении файла через стрим в памяти находится не более одного чанка (по умолчанию 64 KB),
независимо от размера файла:

| Подход | Файл 1 GB | Потребление памяти |
|--------|-----------|-------------------|
| `fs.readFileSync` | Целиком в Buffer | ~1 GB |
| `fs.readFile` (колбэк) | Целиком в Buffer | ~1 GB |
| `fs.createReadStream` | Текущий чанк | ~64 KB |
| `pipeline(readStream, ...)` | Текущий чанк | ~64 KB |

Стримы незаменимы для:
- Обработки файлов, не помещающихся в RAM
- Потокового видеокодирования и шифрования
- Курсоров базы данных (записи читаются чанками)
- Передачи больших HTTP-ответов без буферизации всего тела
