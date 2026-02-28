# Web Streams — потоки данных в браузере и современном Node.js

---

## Введение

Web Streams API (WHATWG Streams) — это стандарт W3C/WHATWG для работы с потоками данных
в браузере. Начиная с Node.js 18, этот API доступен глобально без `require`.

Принципиальное отличие от Node.js Streams: Web Streams не основаны на `EventEmitter`.
Вместо событий используется Promise-based API — каждая операция возвращает Promise,
и backpressure реализуется через разрешение или отклонение этих Promise.

Стримы в браузере встречаются при:
- Загрузке файлов с сервера (Fetch API — `response.body` это `ReadableStream`)
- Работе с файлами через File System Access API
- Обработке медиаданных (MediaRecorder)
- Любой потоковой передаче данных через Service Workers

---

## 2.1 ReadableStream — чтение из Fetch API

```javascript
// Пример 13. Чтение потокового ответа через getReader()
//
// fetch() возвращает Response, у которого body — это ReadableStream.
// Мы можем читать его побайтово через reader, не ожидая загрузки всего файла.
//
// Это критически важно для больших файлов: браузер не держит весь файл в памяти.
//
// Чанки приходят как Uint8Array — типизированный байтовый массив.
// В браузере нет Buffer.concat, поэтому склейку реализуем вручную.

// Этот код запускается в Node.js 18+ (fetch доступен глобально)
// или в браузере без изменений.

async function fetchWithProgress(url) {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`HTTP ошибка: ${response.status} ${response.statusText}`);
  }

  // Получаем длину файла из заголовка (может отсутствовать)
  const contentLength = response.headers.get('content-length');
  const total = contentLength ? parseInt(contentLength, 10) : null;

  // Получаем reader — единственный ридер на стрим одновременно.
  // После вызова getReader() стрим становится "locked" (заблокированным).
  const reader = response.body.getReader();

  const chunks = [];
  let received = 0;

  try {
    while (true) {
      // read() возвращает Promise<{ done: boolean, value: Uint8Array | undefined }>
      const { done, value } = await reader.read();

      if (done) {
        // value === undefined когда done === true
        break;
      }

      chunks.push(value);
      received += value.length;

      // Прогресс загрузки
      if (total) {
        const percent = ((received / total) * 100).toFixed(1);
        process.stdout.write(`\rЗагружено: ${percent}% (${received}/${total} байт)`);
      } else {
        process.stdout.write(`\rПолучено: ${received} байт`);
      }
    }
  } finally {
    // Освобождаем reader — это разблокирует стрим
    reader.releaseLock();
  }

  console.log('\nЗагрузка завершена');
  return concatUint8Arrays(chunks);
}

// Аналог Buffer.concat для Uint8Array (в браузере Buffer недоступен)
function concatUint8Arrays(arrays) {
  const totalLength = arrays.reduce((sum, arr) => sum + arr.length, 0);
  const result = new Uint8Array(totalLength);
  let offset = 0;
  for (const arr of arrays) {
    result.set(arr, offset);
    offset += arr.length;
  }
  return result;
}

// Запуск (только в Node.js 18+ или браузере)
async function main() {
  try {
    const data = await fetchWithProgress('https://httpbin.org/bytes/1024');
    const decoder = new TextDecoder();
    console.log(`Получено ${data.length} байт`);
  } catch (err) {
    console.error('Ошибка:', err.message);
  }
}

main();
```

### Чтение через `for await...of` — единый интерфейс

```javascript
// Пример 14. for await...of с Web ReadableStream
//
// ReadableStream (Web Streams API) также реализует Symbol.asyncIterator.
// Это означает, что и Node.js Streams, и Web Streams читаются ОДИНАКОВО
// через for await — унифицированный интерфейс для любых потоков данных.
//
// При использовании for await:
// - Не нужно вручную создавать reader через getReader()
// - Ошибки перехватываются через try/catch
// - Стрим автоматически освобождается при выходе из цикла

async function fetchWithForAwait(url) {
  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`HTTP ошибка: ${response.status}`);
  }

  const chunks = [];
  let byteCount = 0;

  // for await работает напрямую с response.body (ReadableStream)
  // Под капотом создаётся ридер, который освобождается автоматически
  for await (const chunk of response.body) {
    // chunk — Uint8Array
    chunks.push(chunk);
    byteCount += chunk.length;
    console.log(`Чанк: ${chunk.length} байт (тип: ${chunk.constructor.name})`);
  }

  console.log(`Всего получено: ${byteCount} байт в ${chunks.length} чанках`);

  // Декодируем байты в строку
  const decoder = new TextDecoder('utf-8');
  const total = concatUint8Arrays(chunks);
  return decoder.decode(total);
}

function concatUint8Arrays(arrays) {
  const totalLength = arrays.reduce((sum, arr) => sum + arr.length, 0);
  const result = new Uint8Array(totalLength);
  let offset = 0;
  for (const arr of arrays) {
    result.set(arr, offset);
    offset += arr.length;
  }
  return result;
}

// Запуск (Node.js 18+ или браузер)
fetchWithForAwait('https://httpbin.org/json')
  .then((text) => console.log('Ответ:', text.slice(0, 100)))
  .catch(console.error);
```

---

## 2.2 WritableStream в браузере

```javascript
// Пример 15. WritableStream через открытый конструктор
//
// WritableStream создаётся передачей объекта с методами в конструктор
// (паттерн Revealing Constructor — тот же, что в Node.js Writable).
//
// Методы:
//   start(controller)      — вызывается при создании стрима
//   write(chunk, controller) — вызывается для каждого чанка (может быть async)
//   close()                — вызывается когда все данные записаны
//   abort(reason)          — вызывается при отмене стрима

// WritableStream — Web API, доступен в браузере и Node.js 18+
function createCollectorWritable() {
  const collectedChunks = [];

  const stream = new WritableStream({
    // write возвращает Promise — это основа Promise-based backpressure.
    // Следующий чанк не придёт, пока Promise не разрешится.
    async write(chunk) {
      // Имитируем асинхронную обработку
      await new Promise((resolve) => setTimeout(resolve, 5));
      collectedChunks.push(chunk);
      console.log(`WritableStream получил чанк: ${chunk.constructor.name}, ${chunk.length} байт`);
    },

    close() {
      console.log(`WritableStream закрыт. Собрано чанков: ${collectedChunks.length}`);
    },

    abort(reason) {
      console.error('WritableStream отменён:', reason);
    }
  });

  return { stream, getChunks: () => collectedChunks };
}

// Демонстрация записи через writer
async function demoWritableStream() {
  const { stream, getChunks } = createCollectorWritable();

  // getWriter() блокирует стрим — только один writer за раз
  const writer = stream.getWriter();

  try {
    // Записываем данные
    await writer.write(new Uint8Array([72, 101, 108, 108, 111])); // "Hello"
    await writer.write(new Uint8Array([87, 111, 114, 108, 100])); // "World"

    // Закрываем стрим — вызовет close()
    await writer.close();
  } finally {
    // Освобождаем writer
    writer.releaseLock();
  }

  const chunks = getChunks();
  const decoder = new TextDecoder();
  const text = chunks.map((c) => decoder.decode(c)).join('');
  console.log('Собранный текст:', text);
}

demoWritableStream().catch(console.error);
```

---

## 2.3 TransformStream и pipeThrough

```javascript
// Пример 16. TransformStream — трансформация в цепочке Web Streams
//
// TransformStream содержит внутри:
//   .readable — ReadableStream (выход трансформации)
//   .writable — WritableStream (вход трансформации)
//
// pipeThrough(transform) — аналог .pipe(transform) в Node.js.
// Возвращает новый ReadableStream (readable-конец трансформа).
// Это позволяет строить декларативные цепочки.
//
// Метод transform(chunk, controller):
//   controller.enqueue(value) — передать данные дальше (аналог this.push в Node.js)
//   controller.terminate()    — закончить стрим (аналог push(null) в Node.js)
//   controller.error(reason)  — завершить с ошибкой

async function demonstrateTransformStream() {
  // Создаём тестовый ReadableStream из массива строк
  const sourceData = [
    'Hello World',
    'Node.js Streams',
    'Web Streams API',
    'Transform Pipeline',
  ];

  // ReadableStream из массива
  const readable = new ReadableStream({
    start(controller) {
      for (const item of sourceData) {
        controller.enqueue(item); // кладём строку в поток
      }
      controller.close(); // закрываем поток
    }
  });

  // TransformStream 1: преобразует строки в верхний регистр
  const upperCaseTransform = new TransformStream({
    transform(chunk, controller) {
      // chunk — строка
      controller.enqueue(chunk.toUpperCase());
    }
  });

  // TransformStream 2: подсчитывает длину и создаёт объект-статистику
  // objectMode в Web Streams не требуется явно — типы не ограничены
  const statsTransform = new TransformStream({
    transform(chunk, controller) {
      controller.enqueue({
        original: chunk,
        length: chunk.length,
        wordCount: chunk.split(' ').length,
      });
    }
  });

  // Строим цепочку через pipeThrough
  // readable -> upperCase -> stats -> результирующий ReadableStream
  const resultStream = readable
    .pipeThrough(upperCaseTransform)
    .pipeThrough(statsTransform);

  // Читаем результат через for await
  console.log('Результат цепочки трансформаций:');
  for await (const record of resultStream) {
    console.log(`  "${record.original}" | длина: ${record.length} | слов: ${record.wordCount}`);
  }
}

demonstrateTransformStream().catch(console.error);
```

---

## 2.4 Pipe в Web Streams

```javascript
// Пример 17. pipeTo — передача данных из ReadableStream в WritableStream
//
// pipeTo() — аналог pipe() из Node.js для Web Streams.
// Возвращает Promise, который разрешается когда вся передача завершена.
// Поддерживает AbortSignal для отмены.

async function pipeToDemo() {
  // Создаём ReadableStream, генерирующий числа
  let count = 0;
  const numberStream = new ReadableStream({
    pull(controller) {
      if (count >= 5) {
        controller.close();
        return;
      }
      controller.enqueue(count++);
    }
  });

  // TransformStream: число -> строка
  const toStringTransform = new TransformStream({
    transform(chunk, controller) {
      controller.enqueue(`Number: ${chunk}\n`);
    }
  });

  // WritableStream: выводит в консоль
  const consoleWritable = new WritableStream({
    write(chunk) {
      process.stdout.write(chunk);
    },
    close() {
      console.log('WritableStream закрыт');
    }
  });

  // Цепочка: numbers -> toString -> console
  await numberStream
    .pipeThrough(toStringTransform)
    .pipeTo(consoleWritable);
}

pipeToDemo().catch(console.error);
```

---

## 2.5 Отличия Web Streams от Node.js Streams

| Аспект | Node.js Streams | Web Streams (WHATWG) |
|--------|-----------------|----------------------|
| Основан на | `EventEmitter` | Promise API |
| Чтение | `on('data')`, `for await`, `pipe` | `getReader()`, `for await`, `pipeTo` |
| Запись | `write(chunk, enc, next)` | `getWriter()`, `writer.write(chunk)` |
| Трансформация | `Transform` класс | `TransformStream` класс |
| Цепочки | `pipe()` / `pipeline()` | `pipeThrough()` / `pipeTo()` |
| Завершение стрима | `push(null)` | `controller.close()` |
| Завершение трансформа | `push(null)` | `controller.terminate()` |
| Блокировка | нет | `locked: true` при наличии reader/writer |
| Backpressure | Флаг из `write()` + событие `drain` | Promise из `write()` |
| Достпуность | Node.js (`require('node:stream')`) | Глобально (браузер, Node.js 18+) |
| Интероперабельность | `toWeb()` / `fromWeb()` | `Readable.fromWeb()` / `toWeb()` |

**Унифицированный интерфейс через `for await`:**

И Node.js Streams, и Web Streams реализуют `Symbol.asyncIterator`. Это означает, что
один и тот же код `for await (const chunk of stream)` работает для обоих типов.
Это намеренное проектное решение — унифицировать доступ к потокам данных независимо
от их происхождения.

---

