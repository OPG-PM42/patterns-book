# Node.js Streams — Readable, Writable, Transform, backpressure, pipeline

---

## Введение

Стримы (потоки данных) — один из наиболее мощных и при этом наименее понятых инструментов
Node.js. Это абстракция, позволяющая обрабатывать данные последовательно, кусками (chunks),
не загружая их целиком в оперативную память. Стримы пронизывают всю экосистему Node.js,
и разработчики нередко работают с ними, даже не осознавая этого.

Примеры стримов в Node.js:

- `request` и `response` в HTTP-сервере — это `Readable` и `Writable` стрим
- `process.stdin`, `process.stdout`, `process.stderr` — стандартные стримы процесса
- TCP-сокеты, WebSocket-соединения — дуплексные стримы
- `fs.createReadStream`, `fs.createWriteStream` — файловые стримы
- `zlib.createGzip()` — стрим-трансформатор (сжатие данных на лету)

Понимание стримов необходимо для написания эффективных серверных приложений: без них
обработка файлов размером, превышающим RAM, или потоковая передача данных по HTTP
была бы невозможна.

---

## 1.1 Архитектура стримов

Стримы в Node.js имеют два ключевых свойства:

1. **Наследуют от `EventEmitter`** — можно эмитировать события (`data`, `end`, `error`,
   `drain`) и подписываться на них методами `on`, `once`, `removeListener`.
2. **Реализуют протокол асинхронной итерации** (`Symbol.asyncIterator`) — поддерживают
   конструкцию `for await...of`.

Типы стримов:

| Тип | Описание | Примеры |
|-----|----------|---------|
| `Readable` | Поток для чтения | `fs.createReadStream`, `process.stdin`, `http.IncomingMessage` |
| `Writable` | Поток для записи | `fs.createWriteStream`, `process.stdout`, `http.ServerResponse` |
| `Transform` | Чтение и запись с трансформацией | `zlib.createGzip`, `crypto.createCipher` |
| `Duplex` | Чтение и запись без связи между ними | TCP-сокеты (`net.Socket`) |

`Transform` является частным случаем `Duplex`: то, что записывается в `Writable`-сторону,
проходит через функцию преобразования и выходит на `Readable`-стороне.

---

## 1.2 Readable Stream — создание и стили чтения

### Способы создания Readable Stream

```javascript
// Пример 1. Три способа создания Readable Stream
'use strict';
const { Readable } = require('node:stream');
const fs = require('node:fs');

// --- Способ 1: из файловой системы ---
// Создаём поток чтения файла. Файл читается кусками по 64 KB (highWaterMark по умолчанию).
const fileStream = fs.createReadStream('/tmp/example.txt', { encoding: 'utf8' });

// --- Способ 2: через конструктор с push ---
// Создаём "ручной" стрим, куда мы сами кладём данные методом push.
// push(null) — служебный сигнал: конец стрима (по аналогии с EOF).
// ВАЖНО: обычные null-значения нельзя стримить — они зарезервированы для сигнала завершения.
const manualStream = new Readable({ read() {} });
manualStream.push('Hello, ');
manualStream.push('World!');
manualStream.push(null); // конец стрима

// --- Способ 3: Readable.from — из итерируемого или асинхронного итератора ---
// Это самый удобный способ, когда данные порождает генераторная функция.
async function* generateData() {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  for (const char of chars) {
    yield char;
  }
}
const generatorStream = Readable.from(generateData());
```

### Стиль 1: событие `data`

```javascript
// Пример 2. Чтение через событие 'data' — flowing mode (поточный режим)
//
// Подписка на событие 'data' переводит стрим в "flowing mode":
// данные начинают автоматически читаться из источника и передаваться в обработчик
// по одному чанку за раз.
//
// Преимущество: простота.
// Недостаток: нет контроля над скоростью потребления данных (backpressure не работает).

'use strict';
const { Readable } = require('node:stream');

async function* tenLines() {
  for (let i = 1; i <= 10; i++) {
    yield `Строка номер ${i}\n`;
  }
}

const readable = Readable.from(tenLines());

let totalBytes = 0;

readable.on('data', (chunk) => {
  // chunk — это Buffer (по умолчанию) или string, если задана кодировка
  totalBytes += Buffer.isBuffer(chunk) ? chunk.length : Buffer.byteLength(chunk);
  process.stdout.write(chunk);
});

readable.on('end', () => {
  console.log(`\nВсего байт прочитано: ${totalBytes}`);
});

// ВАЖНО: всегда подписываться на 'error', иначе EventEmitter
// выбросит ошибку как необработанное исключение и Node.js упадёт.
readable.on('error', (err) => {
  console.error('Ошибка стрима:', err.message);
});
```

### Стиль 2: событие `readable`

```javascript
// Пример 3. Чтение через событие 'readable' — paused mode (паузированный режим)
//
// Событие 'readable' сигнализирует: в буфере накопились данные, можно читать.
// Чтение выполняется явно через readable.read(), который возвращает null,
// когда буфер опустел (пора ждать следующего 'readable').
//
// Отличие от 'data': одно событие 'readable' может охватывать несколько чанков,
// накопившихся в буфере. Это даёт больше контроля над потреблением.

'use strict';
const fs = require('node:fs');
const path = require('node:path');
const os = require('node:os');

// Создадим тестовый файл
const tmpFile = path.join(os.tmpdir(), 'readable-test.txt');
fs.writeFileSync(tmpFile, 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'.repeat(100));

// Читаем маленькими чанками — highWaterMark: 16 байт
const stream = fs.createReadStream(tmpFile, { highWaterMark: 16 });

let chunkCount = 0;

stream.on('readable', () => {
  let chunk;
  // Читаем ВСЕ доступные данные в цикле.
  // read() возвращает null, когда данные в буфере закончились.
  // Выход из цикла по null — обязателен, иначе infinite loop.
  while ((chunk = stream.read()) !== null) {
    chunkCount++;
    console.log(`Чанк #${chunkCount}: ${chunk.length} байт`);
  }
});

stream.on('end', () => {
  console.log(`Итого чанков: ${chunkCount}`);
  // Удаляем тестовый файл
  fs.unlink(tmpFile, () => {});
});

stream.on('error', (err) => {
  console.error('Ошибка:', err.message);
});
```

### Стиль 3: `for await...of` (рекомендуется)

```javascript
// Пример 4. Чтение через for await...of — унифицированный асинхронный интерфейс
//
// Это рекомендуемый способ чтения в современном Node.js.
// Readable реализует Symbol.asyncIterator, что позволяет использовать его
// в конструкции for await напрямую.
//
// Преимущества:
// - Читаемый, линейный код без колбэков
// - Ошибки ловятся через try/catch (стандартный механизм)
// - Работает одинаково для Node.js стримов и Web Streams

'use strict';
const { Readable } = require('node:stream');

// Генератор, имитирующий чтение данных из внешнего источника
async function* dataSource() {
  const items = ['{"id":1,"name":"Alice"}', '{"id":2,"name":"Bob"}', '{"id":3,"name":"Carol"}'];
  for (const item of items) {
    // Имитируем задержку I/O (например, чтение из сети)
    await new Promise((resolve) => setTimeout(resolve, 50));
    yield item + '\n';
  }
}

const stream = Readable.from(dataSource());

async function processStream() {
  const lines = [];

  try {
    for await (const chunk of stream) {
      // chunk — Buffer или string
      const line = chunk.toString().trim();
      if (line) {
        lines.push(JSON.parse(line));
        console.log('Прочитана запись:', line);
      }
    }
  } catch (err) {
    // Ошибки из стрима (и из генератора) ловятся здесь
    console.error('Ошибка при чтении:', err.message);
  }

  console.log('Все записи:', lines);
}

processStream();
```

---

## 1.3 Writable Stream — создание и запись

```javascript
// Пример 5. Создание Writable Stream через открытый конструктор (Revealing Constructor Pattern)
//
// Writable создаётся передачей объекта с методом write в конструктор.
// Метод write вызывается для каждого записанного чанка и принимает три аргумента:
//   chunk    — данные (Buffer или string)
//   encoding — кодировка (актуально только для строк)
//   next     — колбэк для сигнала "обработка завершена, можно принять следующий чанк"
//
// Вызов next() управляет backpressure: пока next не вызван, следующий чанк не придёт.
// Если в процессе обработки произошла ошибка — передаём её: next(new Error(...)).

'use strict';
const { Writable } = require('node:stream');
const { Readable } = require('node:stream');

// Создаём WritableStream, который записывает данные в массив (in-memory sink)
function createCollectorStream() {
  const collected = [];

  const writable = new Writable({
    // defaultEncoding по умолчанию 'utf8' для строк
    defaultEncoding: 'utf8',

    async write(chunk, encoding, next) {
      // Имитируем асинхронную операцию записи (например, запись в БД)
      await new Promise((resolve) => setTimeout(resolve, 10));

      const str = chunk.toString(encoding);
      collected.push(str);
      console.log(`Записан чанк [${collected.length}]: "${str.trim()}"`);

      // Сигнализируем: чанк обработан, можно принять следующий
      next();
    },

    // final вызывается когда все данные записаны и стрим закрывается
    final(callback) {
      console.log(`Стрим закрыт. Всего записано: ${collected.length} чанков`);
      callback();
    }
  });

  return { writable, collected };
}

// Демонстрация
async function demo() {
  const { writable, collected } = createCollectorStream();

  // Запись данных в стрим
  writable.write('Первая строка\n');
  writable.write('Вторая строка\n');
  writable.write('Третья строка\n');

  // end() записывает последний чанк (опционально) и закрывает стрим
  writable.end('Финальная строка\n');

  // Ждём завершения записи
  await new Promise((resolve, reject) => {
    writable.on('finish', resolve);
    writable.on('error', reject);
  });

  console.log('Собранные данные:', collected);
}

demo().catch(console.error);
```

---

## 1.4 Pipe и Pipeline

### Метод `pipe`

`pipe` перенаправляет данные из `Readable` в `Writable`. Внутри он подписывается на
событие `data` и вызывает `writable.write(chunk)`, управляя паузой/возобновлением чтения.
Никакой магии — это просто автоматизированный backpressure-менеджмент.

```javascript
// Пример 6. Использование pipe — перенаправление данных
//
// pipe(destination) возвращает destination, что позволяет строить цепочки.
// Один Readable можно pipe-ить в несколько Writable одновременно (разветвление потока).

'use strict';
const fs = require('node:fs');
const path = require('node:path');
const os = require('node:os');

// Создаём тестовый файл
const src = path.join(os.tmpdir(), 'pipe-src.txt');
const dst1 = path.join(os.tmpdir(), 'pipe-dst1.txt');
const dst2 = path.join(os.tmpdir(), 'pipe-dst2.txt');

fs.writeFileSync(src, 'Hello, Streams World!\n'.repeat(1000));

const readable = fs.createReadStream(src);
const writable1 = fs.createWriteStream(dst1);
const writable2 = fs.createWriteStream(dst2);

// Один Readable -> два Writable (разветвление потока)
readable.pipe(writable1);
readable.pipe(writable2);

// ПРОБЛЕМА с pipe: ошибки не propagate-ятся автоматически.
// Если writable1 выдаст ошибку, readable не будет уничтожен автоматически.
// Для этого существует pipeline.
readable.on('error', (err) => console.error('Ошибка чтения:', err.message));
writable1.on('error', (err) => console.error('Ошибка записи 1:', err.message));
writable2.on('error', (err) => console.error('Ошибка записи 2:', err.message));

writable1.on('finish', () => {
  console.log('Файл 1 записан');
  // Чистим временные файлы
  fs.unlink(src, () => {});
  fs.unlink(dst1, () => {});
  fs.unlink(dst2, () => {});
});
```

### Функция `pipeline`

`pipeline` — предпочтительная альтернатива `pipe`. Она автоматически обрабатывает
ошибки и уничтожает все стримы в цепочке при ошибке в любом из них. Поддерживает
отмену операции через `AbortSignal`.

```javascript
// Пример 7. pipeline с AbortSignal и таймаутом
//
// pipeline гарантирует:
// 1. Все стримы будут корректно закрыты при завершении или ошибке
// 2. Ошибка любого стрима отменяет всю цепочку
// 3. AbortSignal позволяет отменить передачу извне
//
// AbortSignal.timeout(ms) — удобный способ без явного AbortController.
// Этот метод возвращает сигнал, который автоматически сработает через ms миллисекунд.

'use strict';
const { pipeline } = require('node:stream/promises');
const zlib = require('node:zlib');
const fs = require('node:fs');
const path = require('node:path');
const os = require('node:os');

const srcFile = path.join(os.tmpdir(), 'pipeline-src.txt');
const gzFile = path.join(os.tmpdir(), 'pipeline-out.gz');

// Создаём достаточно большой файл для демонстрации
fs.writeFileSync(srcFile, 'Node.js Streams are powerful!\n'.repeat(50000));

async function compressWithTimeout() {
  // Создаём AbortController вручную для ручного управления отменой
  const ac = new AbortController();

  // Устанавливаем таймаут: если сжатие займёт больше 5 секунд — отменяем
  const timeoutId = setTimeout(() => {
    console.log('Таймаут! Отменяем pipeline...');
    ac.abort();
  }, 5000);

  try {
    await pipeline(
      fs.createReadStream(srcFile),
      zlib.createGzip(),          // Transform stream: сжатие данных
      fs.createWriteStream(gzFile),
      { signal: ac.signal }       // Передаём AbortSignal
    );
    console.log('Сжатие завершено успешно');
  } catch (err) {
    if (err.name === 'AbortError') {
      console.log('Pipeline отменён по AbortSignal');
    } else {
      console.error('Ошибка pipeline:', err.message);
    }
  } finally {
    clearTimeout(timeoutId);
    // Чистим файлы
    fs.unlink(srcFile, () => {});
    fs.unlink(gzFile, () => {});
  }
}

// Альтернатива — AbortSignal.timeout без явного AbortController:
async function compressWithAutoTimeout() {
  const srcFile2 = path.join(os.tmpdir(), 'pipeline-src2.txt');
  const gzFile2 = path.join(os.tmpdir(), 'pipeline-out2.gz');
  fs.writeFileSync(srcFile2, 'Data\n'.repeat(10));

  try {
    await pipeline(
      fs.createReadStream(srcFile2),
      zlib.createGzip(),
      fs.createWriteStream(gzFile2),
      { signal: AbortSignal.timeout(10) } // 10 мс — файл не успеет сжаться
    );
    console.log('Сжатие завершено');
  } catch (err) {
    if (err.name === 'AbortError' || err.name === 'TimeoutError') {
      console.log('Операция прервана по таймауту (ожидаемо)');
    }
  } finally {
    fs.unlink(srcFile2, () => {});
    fs.unlink(gzFile2, () => {});
  }
}

compressWithTimeout().then(compressWithAutoTimeout).catch(console.error);
```

---

## 1.5 Transform Stream

Transform — это двунаправленный стрим. Данные, записанные в его `Writable`-сторону,
проходят через метод `transform`, преобразуются и выходят на `Readable`-стороне.

```javascript
// Пример 8. Transform Stream: "медленный поток" с побуквенной выдачей
//
// Этот Transform принимает строки и выдаёт их посимвольно с задержкой.
// Эффект: текст "печатается" в терминале с заданной скоростью.
//
// Ключевые моменты:
// - this.push(data) — положить данные в readable-сторону трансформа
// - next() — сигнал "чанк обработан, можно принять следующий"
// - setTimeoutPromise из node:timers/promises — неблокирующая пауза

'use strict';
const { Transform, Readable } = require('node:stream');
const { setTimeout: setTimeoutPromise } = require('node:timers/promises');
const { pipeline } = require('node:stream/promises');

// Фабрика создаёт SlowStream с настраиваемой задержкой между символами
function createSlowStream(delayMs) {
  return new Transform({
    async transform(chunk, encoding, next) {
      const str = chunk.toString();
      for (const char of str) {
        // Кладём символ в readable-буфер трансформа
        this.push(char);
        // Ждём перед отправкой следующего символа
        await setTimeoutPromise(delayMs);
      }
      // Сигнализируем: чанк обработан
      next();
    }
  });
}

async function demo() {
  const text = 'Hello, Node.js Streams!\n';
  const source = Readable.from([text]);
  const slowStream = createSlowStream(50); // 50 мс между символами

  await pipeline(
    source,
    slowStream,
    process.stdout   // Writable — финальный получатель данных
  );
}

demo().catch(console.error);
```

### Object Mode

По умолчанию стримы работают с `Buffer` и строками. Включив `objectMode: true`,
можно передавать произвольные JavaScript-объекты. Это принципиально меняет семантику:
`highWaterMark` теперь считается не в байтах, а в количестве объектов.

```javascript
// Пример 9. Transform Stream в objectMode — агрегация данных
//
// Задача: есть каталог товаров, разбитый по группам.
// Нужно для каждой группы посчитать общую стоимость (SAP Total).
//
// objectMode позволяет передавать структурированные JS-объекты,
// а не только байтовые данные. Это естественно для трансформации данных в памяти.

'use strict';
const { Transform, Writable, Readable } = require('node:stream');
const { pipeline } = require('node:stream/promises');

// Тестовые данные: каталог товаров по группам
const catalog = new Map([
  ['Электроника', [
    { name: 'Ноутбук', price: 75000 },
    { name: 'Телефон', price: 45000 },
    { name: 'Планшет', price: 30000 },
  ]],
  ['Канцелярия', [
    { name: 'Бумага A4', price: 350 },
    { name: 'Ручка', price: 50 },
    { name: 'Тетрадь', price: 80 },
  ]],
  ['Мебель', [
    { name: 'Стол', price: 12000 },
    { name: 'Стул', price: 5000 },
  ]],
]);

// Transform: принимает { group, items }, выдаёт { group, count, total }
const aggregateStream = new Transform({
  objectMode: true, // оба конца (readable и writable) работают с объектами

  transform({ group, items }, encoding, next) {
    // Валидация: цены не могут быть отрицательными
    for (const item of items) {
      if (item.price < 0) {
        // Передача ошибки в next() аварийно завершает стрим
        return next(new Error(`Отрицательная цена для "${item.name}" в группе "${group}"`));
      }
    }

    const total = items.reduce((sum, item) => sum + item.price, 0);

    // this.push() кладёт объект в readable-буфер
    this.push({ group, count: items.length, total });
    next();
  }
});

// Writable: выводит результат агрегации
const printStream = new Writable({
  objectMode: true,
  write(record, encoding, next) {
    const { group, count, total } = record;
    console.log(`Группа: ${group.padEnd(15)} | Позиций: ${count} | Итого: ${total.toLocaleString('ru')} руб.`);
    next();
  }
});

// Readable из Map-а — создаём через Readable.from с кастомным итератором
function* catalogIterator(catalog) {
  for (const [group, items] of catalog) {
    yield { group, items };
  }
}

async function runAggregation() {
  try {
    await pipeline(
      Readable.from(catalogIterator(catalog)),
      aggregateStream,
      printStream
    );
    console.log('Агрегация завершена');
  } catch (err) {
    console.error('Ошибка агрегации:', err.message);
  }
}

runAggregation();
```

---

## 1.6 Backpressure — управление давлением данных

Backpressure — это механизм обратного давления в потоке данных. Он возникает когда
производитель данных работает быстрее потребителя: `Readable` генерирует чанки быстрее,
чем `Writable` успевает их обрабатывать.

**Почему это важно:** без управления backpressure данные накапливаются в буфере `Writable`
в оперативной памяти. При передаче больших файлов (гигабайты) или при медленном потребителе
(медленная запись на диск, медленная сеть) память сервера может быть исчерпана.

**Механизм:**

1. `writable.write(chunk)` возвращает `true` — буфер не заполнен (ниже `highWaterMark`)
2. `writable.write(chunk)` возвращает `false` — буфер заполнен, запись продолжается,
   но нужно паузировать производителя
3. Событие `drain` на `Writable` — буфер опустел, можно возобновить чтение

```javascript
// Пример 10. Ручное управление backpressure
//
// Демонстрирует механизм pause/resume для предотвращения переполнения памяти.
//
// Без backpressure: медленный writable накапливает ВСЕ данные в буфере.
// С backpressure: readable приостанавливается, пока writable не освободит буфер.
//
// Аналогия: кран (readable) и ведро (writable).
// Когда ведро переполняется — закрываем кран (pause).
// Когда ведро опустело — открываем кран снова (resume).

'use strict';
const { Readable, Writable } = require('node:stream');
const { setTimeout: delay } = require('node:timers/promises');

// Быстрый производитель: генерирует 100 чанков по 1 KB мгновенно
async function* fastProducer() {
  const chunk = Buffer.alloc(1024, 'X'); // 1 KB
  for (let i = 1; i <= 100; i++) {
    yield chunk;
  }
}

// Медленный потребитель: обрабатывает каждый чанк 20 мс
function createSlowWriter() {
  let receivedChunks = 0;
  let pauseCount = 0;

  const writable = new Writable({
    // highWaterMark: 4 KB — буфер заполнится очень быстро
    highWaterMark: 4 * 1024,

    async write(chunk, encoding, next) {
      receivedChunks++;
      // Имитируем медленную запись (например, запись на медленный диск)
      await delay(20);
      next();
    },

    final(callback) {
      console.log(`\nЗавершено. Получено чанков: ${receivedChunks}, пауз: ${pauseCount}`);
      callback();
    }
  });

  return { writable, getPauseCount: () => pauseCount, incPause: () => pauseCount++ };
}

async function runWithBackpressure() {
  const readable = Readable.from(fastProducer(), { highWaterMark: 2 * 1024 });
  const { writable, incPause } = createSlowWriter();

  // Подписываемся на 'data' и вручную управляем потоком
  readable.on('data', (chunk) => {
    const canContinue = writable.write(chunk);

    if (!canContinue) {
      // Буфер writable заполнен — паузируем чтение
      readable.pause();
      incPause();
      process.stdout.write('P'); // визуализируем паузу
    }
  });

  writable.on('drain', () => {
    // Буфер writable освободился — возобновляем чтение
    readable.resume();
    process.stdout.write('R'); // визуализируем возобновление
  });

  readable.on('end', () => {
    writable.end();
  });

  await new Promise((resolve, reject) => {
    writable.on('finish', resolve);
    writable.on('error', reject);
    readable.on('error', reject);
  });
}

console.log('Запуск передачи с backpressure (P=пауза, R=возобновление):');
runWithBackpressure().catch(console.error);
```

### Параметр `highWaterMark`

```javascript
// Пример 11. Настройка highWaterMark
//
// highWaterMark — это мягкий порог буфера:
// - Для byte streams: количество байт (по умолчанию 64 * 1024 = 64 KB)
// - Для objectMode streams: количество объектов (по умолчанию 16)
//
// Это МЯГКИЙ порог: write() продолжит работу даже после его превышения,
// но вернёт false, сигнализируя о необходимости паузы.

'use strict';
const fs = require('node:fs');
const path = require('node:path');
const os = require('node:os');

const srcFile = path.join(os.tmpdir(), 'hwm-test.bin');
// Создаём файл 1 MB
fs.writeFileSync(srcFile, Buffer.alloc(1024 * 1024, 0x41)); // 'A' * 1 MB

const outFile = path.join(os.tmpdir(), 'hwm-out.bin');

// Маленький highWaterMark -> больше чанков, больше переключений контекста
const readable = fs.createReadStream(srcFile, { highWaterMark: 16 * 1024 }); // 16 KB

// Большой highWaterMark у writable -> реже срабатывает backpressure
const writable = fs.createWriteStream(outFile, { highWaterMark: 64 * 1024 }); // 64 KB

let chunkCount = 0;

readable.on('data', (chunk) => {
  chunkCount++;
  writable.write(chunk);
});

readable.on('end', () => {
  writable.end();
  console.log(`Файл скопирован: ${chunkCount} чанков по ~${16} KB`);
  fs.unlink(srcFile, () => {});
  fs.unlink(outFile, () => {});
});

readable.on('error', console.error);
writable.on('error', console.error);
```

---

## 1.7 Cork и Uncork — буферизация записи

`writable.cork()` и `writable.uncork()` управляют внутренним буфером `Writable`:

- `cork()` — остановить немедленную запись, накапливать данные в буфере
- `uncork()` — сбросить накопленный буфер одной операцией записи

```javascript
// Пример 12. Cork / Uncork — батчинг мелких записей
//
// Задача: у нас есть много мелких чанков, и мы хотим объединить их
// в один крупный пакет перед реальной записью.
// Пример из жизни: ввод данных посимвольно из stdin, отправка строкой.

'use strict';
const { Writable } = require('node:stream');

let writeCount = 0;

const batchedWritable = new Writable({
  write(chunk, encoding, next) {
    writeCount++;
    // Имитируем запись (например, системный вызов write())
    process.stdout.write(`[write #${writeCount}: "${chunk.toString()}"]\n`);
    next();
  }
});

// БЕЗ cork: каждый write() вызывает метод write немедленно
console.log('=== Без cork ===');
batchedWritable.write('a');
batchedWritable.write('b');
batchedWritable.write('c');

// Сбрасываем счётчик
writeCount = 0;

// С cork: данные накапливаются, uncork() сбрасывает их одним пакетом
console.log('\n=== С cork ===');
batchedWritable.cork();
batchedWritable.write('x');
batchedWritable.write('y');
batchedWritable.write('z');
// Здесь ни одного write() в обработчик ещё не поступало
console.log('(до uncork — записей нет)');
batchedWritable.uncork();
// Теперь все три записи отправлены одним пакетом

batchedWritable.end();
```

---

## Сравнение стилей чтения Readable Stream

| Стиль | Режим | Контроль backpressure | Обработка ошибок | Рекомендация |
|-------|-------|----------------------|------------------|--------------|
| `on('data')` | Flowing | Ручной (`pause`/`resume`) | `on('error')` | Низкоуровневое управление |
| `on('readable')` | Paused | Встроенный | `on('error')` | Больше контроля над чтением |
| `for await...of` | Async | Встроенный | `try/catch` | Современный стиль, рекомендуется |
| `pipe` / `pipeline` | Flowing | Автоматический | `pipeline` пропагейтит | Для цепочек трансформаций |

---

