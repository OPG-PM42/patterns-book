# Явное управление ресурсами при помощи Using и Symbol.dispose в JavaScript, TypeScript, Node.js

## Обзор

Данная лекция посвящена новым механизмам явного управления ресурсами в JavaScript и TypeScript. Основные темы включают:
- Новые символы `Symbol.dispose` и `Symbol.asyncDispose`
- Конструкции `using` и `await using`
- Сравнение с традиционными подходами (try/finally, Finalization Registry)
- Синхронное и асинхронное освобождение ресурсов
- Практические примеры и вложенные контексты

---

## Содержание

1. [Введение: зачем нужно управление ресурсами](#введение-зачем-нужно-управление-ресурсами)
2. [Традиционный подход: try/finally](#традиционный-подход-tryfinally)
3. [Finalization Registry](#finalization-registry)
4. [Новый подход: using и Symbol.dispose](#новый-подход-using-и-symboldispose)
5. [Асинхронное освобождение: await using](#асинхронное-освобождение-await-using)
6. [Различия между синхронным и асинхронным dispose](#различия-между-синхронным-и-асинхронным-dispose)
7. [Вложенные контексты](#вложенные-контексты)
8. [Порядок вызова dispose](#порядок-вызова-dispose)
9. [Резюме](#резюме)

---

## Введение: зачем нужно управление ресурсами

### Проблематика

При работе с ресурсами (файлы, сетевые соединения, базы данных) необходимо обеспечить их корректное освобождение:
- Предотвратить утечки памяти
- Закрыть файловые дескрипторы
- Освободить системные ресурсы
- Обработать ошибки при освобождении

**Ключевой момент**: ресурс нужно освободить независимо от того, была ли ошибка во время его использования.

---

## Традиционный подход: try/finally

### Пример: работа с логером через try/finally

```javascript
async function usingSection() {
  let logger = null;

  try {
    // Создание ресурса (может выбросить ошибку)
    logger = await new Logger('./app.log');

    // Использование ресурса (может выбросить ошибку)
    await logger.write('Starting application...');
    await logger.write('Processing data...');

  } catch (error) {
    console.error('Error occurred:', error);

  } finally {
    // Обязательная очистка ресурса
    if (logger) {
      await logger.close();
      console.log('Logger closed in finally block');
    }
  }
}
```

### Недостатки традиционного подхода

1. **Громоздкость**: требуется явная проверка существования ресурса
2. **Дублирование кода**: логика очистки повторяется для каждого ресурса
3. **Сложность**: при множественных ресурсах код становится трудночитаемым
4. **Ошибки**: легко забыть добавить finally или проверку на null

**Пример проблемы с несколькими ресурсами:**

```javascript
async function multipleResources() {
  let connection = null;
  let file = null;
  let cache = null;

  try {
    connection = await openDatabase();
    file = await openFile('./data.txt');
    cache = await createCache();

    // Работа с ресурсами...

  } finally {
    // Нужно закрыть все ресурсы в обратном порядке
    if (cache) await cache.flush();
    if (file) await file.close();
    if (connection) await connection.close();
  }
}
```

---

## Finalization Registry

### Концепция

`FinalizationRegistry` — это механизм, позволяющий выполнить код при сборке мусора, когда на объект больше нет ссылок.

### Пример использования

```javascript
// Создание глобального реестра финализации
const registry = new FinalizationRegistry((resource) => {
  console.log('Finalization callback triggered');
  if (resource && resource.close) {
    resource.close();
    console.log('File closed by FinalizationRegistry');
  }
});

class Logger {
  #handle;

  constructor(path) {
    this.#handle = fs.openSync(path, 'a');

    // Регистрация ресурса для автоматической очистки
    registry.register(
      this,           // Объект, за которым следим
      this.#handle    // Данные для callback
    );
  }

  write(message) {
    fs.writeSync(this.#handle, message + '\n');
  }
}

// Использование
function example() {
  const logger = new Logger('./app.log');
  logger.write('Message');
  // logger выходит из области видимости
}

example();

// Принудительный вызов сборщика мусора (только с флагом --expose-gc)
if (global.gc) {
  global.gc();
}
```

### Запуск с флагом expose-gc

```bash
# Для запуска с возможностью ручного вызова GC
node --expose-gc example1-finalization.js
```

### Проблемы FinalizationRegistry

1. **Непредсказуемость**: нельзя точно знать, когда вызовется callback
2. **Задержки**: GC может не запуститься долгое время
3. **Нет гарантий**: callback может вообще не вызваться
4. **Зависимость от памяти**: пока есть свободная память, GC не торопится
5. **Старые объекты**: объекты в old generation собираются редко

**Важно**: FinalizationRegistry НЕ подходит для критичных ресурсов (файлы, соединения), которые нужно закрыть немедленно.

---

## Новый подход: using и Symbol.dispose

### Основная идея

Конструкция `using` автоматически вызывает метод `Symbol.dispose` при выходе из блока видимости, где объявлена переменная.

### Базовый пример

```javascript
class Logger {
  #handle;

  constructor(path) {
    this.#handle = fs.openSync(path, 'a');
    console.log(`Logger created for ${path}`);
  }

  write(message) {
    fs.writeSync(this.#handle, message + '\n');
  }

  // Метод для синхронного освобождения ресурса
  [Symbol.dispose]() {
    console.log('Disposing logger (sync)');
    if (this.#handle) {
      fs.closeSync(this.#handle);
      console.log('File closed');
    }
  }
}

// Использование с using
function main() {
  using logger = new Logger('./app.log');

  logger.write('Application started');
  logger.write('Processing...');

  // При выходе из функции автоматически вызовется logger[Symbol.dispose]()
}

main();
console.log('After main completed');
```

### Как это работает

1. **Объявление**: `using logger = new Logger(...)` создает ресурс
2. **Использование**: работаем с ресурсом в пределах блока
3. **Автоматическая очистка**: при достижении закрывающей скобки `}` вызывается `logger[Symbol.dispose]()`
4. **Гарантия выполнения**: dispose вызывается даже при ошибках

### Эквивалентный код без using

```javascript
function main() {
  const logger = new Logger('./app.log');

  try {
    logger.write('Application started');
    logger.write('Processing...');
  } finally {
    logger[Symbol.dispose]();
  }
}
```

### Ключевые особенности using

**1. Привязка к области видимости**

```javascript
function example() {
  using resource = createResource();

  // resource доступен здесь
  console.log('Using resource');

} // <-- resource[Symbol.dispose]() вызывается ЗДЕСЬ
```

**2. Работа с блоками**

```javascript
function blockScopes() {
  console.log('Start');

  {
    using temp = createTempResource();
    console.log('Inside block');
  } // temp.dispose() вызывается здесь

  console.log('After block');
}
```

**3. Передача ссылки не влияет на dispose**

```javascript
let externalRef;

function example() {
  using resource = createResource();
  externalRef = resource; // Сохранили ссылку во внешней переменной

} // resource[Symbol.dispose]() ВСЁ РАВНО вызовется здесь!

// externalRef теперь указывает на "disposed" объект
```

**Важное отличие от Finalization Registry**: `using` освобождает ресурс при выходе из блока, а не при отсутствии ссылок!

---

## Асинхронное освобождение: await using

### Symbol.asyncDispose

Для асинхронной очистки ресурсов используется `Symbol.asyncDispose` в комбинации с `await using`.

### Пример с асинхронным dispose

```javascript
class AsyncLogger {
  #handle;
  #buffer = [];

  constructor(path) {
    this.#handle = fs.openSync(path, 'a');
    console.log(`AsyncLogger created`);
  }

  write(message) {
    this.#buffer.push(message);
  }

  // Асинхронный метод очистки
  async [Symbol.asyncDispose]() {
    console.log('Disposing logger (async)');

    // Шаг 1: Сбросить буфер в файл
    for (const msg of this.#buffer) {
      fs.writeSync(this.#handle, msg + '\n');
    }

    // Шаг 2: Закрыть файл
    fs.closeSync(this.#handle);

    // Шаг 3: Отправить уведомление (асинхронная операция)
    await this.sendNotification();

    console.log('Async disposal completed');
  }

  async sendNotification() {
    return new Promise(resolve => {
      setTimeout(() => {
        console.log('Notification sent');
        resolve();
      }, 100);
    });
  }
}

// Использование с await using
async function main() {
  await using logger = new AsyncLogger('./app.log');

  logger.write('Message 1');
  logger.write('Message 2');

  // При выходе ждем завершения logger[Symbol.asyncDispose]()
}

await main();
console.log('Auto dispose completed');
```

### Зачем нужен асинхронный dispose?

Асинхронный dispose позволяет:
1. **Сбросить кэш** в хранилище
2. **Отправить уведомления** о закрытии ресурса
3. **Дождаться завершения** фоновых операций
4. **Корректно закрыть** сетевые соединения
5. **Запустить обработчики** логов или метрик

### Пример со сложной асинхронной логикой

```javascript
class DatabaseConnection {
  #connection;

  async [Symbol.asyncDispose]() {
    console.log('Starting database cleanup...');

    // 1. Завершить активные транзакции
    await this.#connection.commitPendingTransactions();

    // 2. Сбросить кэш
    await this.#connection.flushCache();

    // 3. Закрыть соединение
    await this.#connection.close();

    // 4. Уведомить monitoring систему
    await this.notifyMonitoring('connection_closed');

    console.log('Database cleanup completed');
  }
}
```

---

## Различия между синхронным и асинхронным dispose

### Таблица приоритетов

| Конструкция | Есть Symbol.asyncDispose | Нет Symbol.asyncDispose |
|-------------|--------------------------|-------------------------|
| `await using` | Вызывается `Symbol.asyncDispose` | Вызывается `Symbol.dispose` |
| `using` | Вызывается `Symbol.dispose` | Вызывается `Symbol.dispose` |

### Правила выбора метода dispose

**Правило 1**: `await using` предпочитает асинхронный метод

```javascript
class Resource {
  [Symbol.dispose]() {
    console.log('Sync dispose');
  }

  async [Symbol.asyncDispose]() {
    console.log('Async dispose');
  }
}

async function test() {
  await using r = new Resource();
} // Вызовется: Async dispose
```

**Правило 2**: `using` всегда использует синхронный метод

```javascript
async function test() {
  using r = new Resource(); // Даже если есть asyncDispose
} // Вызовется: Sync dispose
```

**Правило 3**: Если нужного метода нет — ошибка

```javascript
class NoDispose {
  // Нет ни Symbol.dispose, ни Symbol.asyncDispose
}

async function test() {
  await using r = new NoDispose();
  // TypeError: Cannot find Symbol.dispose method
}
```

### Пример: проверка всех комбинаций

```javascript
// Пример 1: Оба метода определены, await using
class BothMethods {
  [Symbol.dispose]() {
    console.log('Sync dispose');
  }

  async [Symbol.asyncDispose]() {
    console.log('Async dispose');
  }
}

async function example1() {
  await using r = new BothMethods();
}
// Вывод: Async dispose

// Пример 2: Оба метода определены, using
function example2() {
  using r = new BothMethods();
}
// Вывод: Sync dispose

// Пример 3: Только sync dispose, await using
class OnlySync {
  [Symbol.dispose]() {
    console.log('Sync dispose');
  }
}

async function example3() {
  await using r = new OnlySync();
}
// Вывод: Sync dispose (откат к синхронному)

// Пример 4: Только async dispose, using
class OnlyAsync {
  async [Symbol.asyncDispose]() {
    console.log('Async dispose');
  }
}

function example4() {
  using r = new OnlyAsync();
  // ОШИБКА: не найден Symbol.dispose
}
```

---

## Порядок вызова dispose

### Важная особенность синхронного dispose с await внутри

Если в синхронном `Symbol.dispose` используется `await`, выполнение разрывается!

#### Пример 4: Разрыв выполнения

```javascript
class Logger {
  [Symbol.dispose]() {
    console.log('Dispose: before await');

    await new Promise(resolve => setTimeout(resolve, 100));

    console.log('Dispose: after await');
  }
}

function main() {
  using logger = new Logger();
  console.log('Inside main');
}

main();
console.log('Auto dispose');
```

**Вывод:**
```
Inside main
Dispose: before await
Auto dispose              <-- Управление передалось сюда!
Dispose: after await      <-- А это выполнится позже
```

**Объяснение**: Синхронный dispose с `await` разрывается на две части. Управление передается дальше после первого `await`, не дожидаясь завершения dispose.

#### Пример 5: Правильный асинхронный dispose

```javascript
class Logger {
  async [Symbol.asyncDispose]() {
    console.log('Async dispose: before await');

    await new Promise(resolve => setTimeout(resolve, 100));

    console.log('Async dispose: after await');
  }
}

async function main() {
  await using logger = new Logger();
  console.log('Inside main');
}

await main();
console.log('Auto dispose');
```

**Вывод:**
```
Inside main
Async dispose: before await
Async dispose: after await    <-- Полностью выполнился
Auto dispose                   <-- Только потом управление передалось
```

**Вывод**: Используйте `Symbol.asyncDispose` с `await using`, если внутри dispose есть асинхронные операции!

---

## Вложенные контексты

### Порядок освобождения: LIFO (Last In, First Out)

Ресурсы освобождаются в порядке стека — последний созданный освобождается первым.

### Пример с вложенными блоками

```javascript
function createDisposable(id) {
  console.log(`Creating ${id}`);
  return {
    [Symbol.dispose]() {
      console.log(`Disposing ${id}`);
    }
  };
}

function nestedExample() {
  console.log('=== Start ===');

  using a = createDisposable('A');
  using b = createDisposable('B');

  {
    using c = createDisposable('C');
    using d = createDisposable('D');
    console.log('Inside inner block');
  } // <-- D и C освобождаются здесь

  console.log('Back to outer block');

  using e = createDisposable('E');

  console.log('=== End ===');
} // <-- E, B и A освобождаются здесь

nestedExample();
```

**Вывод:**
```
=== Start ===
Creating A
Creating B
Creating C
Creating D
Inside inner block
Disposing D       <-- Последний созданный в блоке
Disposing C       <-- Первый созданный в блоке
Back to outer block
Creating E
=== End ===
Disposing E       <-- Последний в функции
Disposing B       <-- Второй в функции
Disposing A       <-- Первый в функции
```

### Асинхронный пример с вложенностью

```javascript
function loggy(id) {
  console.log(`Constructing ${id}`);
  return {
    async [Symbol.asyncDispose]() {
      console.log(`Disposing (async) ${id}`);
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  };
}

async function func() {
  await using a = loggy('a');
  await using b = loggy('b');

  {
    await using c = loggy('c');
    await using d = loggy('d');
  } // c и d утилизируются здесь

  await using e = loggy('e');

  return;

  // Недостижимый код
  await using f = loggy('f'); // НЕ создастся, НЕ утилизируется
}

await func();
```

**Вывод:**
```
Constructing a
Constructing b
Constructing c
Constructing d
Disposing (async) d
Disposing (async) c
Constructing e
Disposing (async) e
Disposing (async) b
Disposing (async) a
```

### Визуализация порядка

```
Функция func():
┌─────────────────────────────────────┐
│ await using a                       │ ← Создан первым
│ await using b                       │ ← Создан вторым
│                                     │
│ ┌─────────────────────────────┐    │
│ │ Вложенный блок:             │    │
│ │ await using c               │    │ ← Создан третьим
│ │ await using d               │    │ ← Создан четвертым
│ │                             │    │
│ └─────────────────────────────┘    │
│   ↑ Dispose d (4)                   │
│   ↑ Dispose c (3)                   │
│                                     │
│ await using e                       │ ← Создан пятым
│                                     │
└─────────────────────────────────────┘
  ↑ Dispose e (5)
  ↑ Dispose b (2)
  ↑ Dispose a (1)
```

### Перекрытие идентификаторов (shadowing)

```javascript
function shadowingExample() {
  using console = createDisposable('outer console');

  {
    using console = createDisposable('inner console');
    // Здесь доступен inner console
  } // inner console освобождается

  // Здесь снова доступен outer console
} // outer console освобождается
```

---

## Практические сценарии использования

### Сценарий 1: Работа с файлами

```javascript
class TempFile {
  #path;
  #handle;

  constructor(path) {
    this.#path = path;
    this.#handle = fs.openSync(path, 'w+');
    console.log(`Opened temp file: ${path}`);
  }

  write(data) {
    fs.writeSync(this.#handle, data);
  }

  read() {
    const stats = fs.fstatSync(this.#handle);
    const buffer = Buffer.alloc(stats.size);
    fs.readSync(this.#handle, buffer, 0, stats.size, 0);
    return buffer.toString();
  }

  [Symbol.dispose]() {
    fs.closeSync(this.#handle);
    fs.unlinkSync(this.#path);
    console.log(`Deleted temp file: ${this.#path}`);
  }
}

function processTempData() {
  using tempFile = new TempFile('./temp_data.tmp');

  tempFile.write('Important data');
  const content = tempFile.read();
  console.log('Content:', content);

  // Файл автоматически удалится при выходе
}
```

### Сценарий 2: Трассировка выполнения

```javascript
class TraceActivity {
  #name;
  #startTime;

  constructor(name) {
    this.#name = name;
    this.#startTime = Date.now();
    console.log(`→ Entering: ${name}`);
  }

  [Symbol.dispose]() {
    const duration = Date.now() - this.#startTime;
    console.log(`← Exiting: ${this.#name} (${duration}ms)`);
  }
}

function businessLogic() {
  using _trace = new TraceActivity('businessLogic');

  // Бизнес-логика
  for (let i = 0; i < 1000000; i++) {
    Math.sqrt(i);
  }
}

businessLogic();
// Вывод:
// → Entering: businessLogic
// ← Exiting: businessLogic (15ms)
```

### Сценарий 3: Управление транзакциями

```javascript
class Transaction {
  #db;
  #committed = false;

  constructor(db) {
    this.#db = db;
    this.#db.begin();
    console.log('Transaction started');
  }

  commit() {
    this.#db.commit();
    this.#committed = true;
    console.log('Transaction committed');
  }

  [Symbol.dispose]() {
    if (!this.#committed) {
      this.#db.rollback();
      console.log('Transaction rolled back');
    }
  }
}

function updateDatabase(db) {
  using transaction = new Transaction(db);

  db.update('users', { name: 'John' });
  db.update('orders', { status: 'completed' });

  // Если ошибка — автоматический rollback
  if (someCondition()) {
    transaction.commit();
  }
  // Иначе — rollback при выходе
}
```

### Сценарий 4: Блокировки и мьютексы

```javascript
class Lock {
  #resource;

  constructor(resource) {
    this.#resource = resource;
    resource.acquire();
    console.log('Lock acquired');
  }

  [Symbol.dispose]() {
    this.#resource.release();
    console.log('Lock released');
  }
}

function criticalSection(resource) {
  using lock = new Lock(resource);

  // Критическая секция — только один поток может быть здесь
  modifySharedData();

  // Блокировка автоматически освободится
}
```

---

## Reference Counting (предварительно)

Лектор упоминает, что `using` **не реализует автоматический подсчет ссылок** (reference counting), как в некоторых других языках.

### Что это значит?

```javascript
let externalRef;

function example() {
  using resource = createResource();
  externalRef = resource; // Создали дополнительную ссылку

} // resource.dispose() вызовется, даже если externalRef еще жива!

// externalRef теперь указывает на "мертвый" объект
```

**Планы**: Лектор обещает отдельную лекцию о том, как реализовать reference counting поверх `using`.

---

## Совместимость и полифиллы

### Проверка поддержки

```javascript
// Проверка наличия Symbol.dispose
if (typeof Symbol.dispose === 'undefined') {
  // Полифилл
  Symbol.dispose = Symbol('Symbol.dispose');
}

if (typeof Symbol.asyncDispose === 'undefined') {
  Symbol.asyncDispose = Symbol('Symbol.asyncDispose');
}
```

### Полифилл от TypeScript

TypeScript предоставляет полифиллы для окружений без нативной поддержки:

```typescript
Symbol.dispose ??= Symbol("Symbol.dispose");
Symbol.asyncDispose ??= Symbol("Symbol.asyncDispose");
```

### Поддержка в рантаймах

На момент записи лекции (упоминается в конце):
- **Node.js 24**: Полная поддержка ✅
- **Chrome/Chromium**: Полная поддержка ✅
- **TypeScript 5.2+**: Полная поддержка ✅

---

## Обработка ошибок

### SuppressedError

Если ошибка происходит и в основном коде, и в `dispose`, создается специальный `SuppressedError`:

```javascript
class ErrorThrower {
  [Symbol.dispose]() {
    throw new Error('Error in dispose');
  }
}

function test() {
  using resource = new ErrorThrower();
  throw new Error('Error in main code');
}

try {
  test();
} catch (e) {
  console.log(e.name);              // SuppressedError
  console.log(e.message);           // An error was suppressed during disposal
  console.log(e.error.message);     // Error in dispose
  console.log(e.suppressed.message); // Error in main code
}
```

### Множественные ошибки dispose

```javascript
function multipleErrors() {
  using a = { [Symbol.dispose]() { throw new Error('A failed'); } };
  using b = { [Symbol.dispose]() { throw new Error('B failed'); } };
}

try {
  multipleErrors();
} catch (e) {
  // Обе ошибки будут собраны в цепочку SuppressedError
}
```

---

## Использование в циклах

### For loop

```javascript
function processFiles(files) {
  for (using file = openFile(files[0]); file.hasNext(); file.next()) {
    console.log(file.read());
  } // file закрывается после цикла
}
```

### For...of loop

```javascript
function* createResources() {
  yield new Resource('A');
  yield new Resource('B');
  yield new Resource('C');
}

function processResources() {
  for (using resource of createResources()) {
    console.log(resource.use());
    // Каждый resource автоматически освобождается после итерации
  }
}
```

---

## Резюме

### Основные концепции

1. **`using` объявление**
   - Автоматически вызывает `Symbol.dispose` при выходе из блока
   - Работает синхронно
   - Гарантирует освобождение ресурса даже при ошибках

2. **`await using` объявление**
   - Автоматически вызывает `Symbol.asyncDispose` при выходе из блока
   - Ожидает завершения асинхронной очистки
   - Откатывается к `Symbol.dispose`, если асинхронного метода нет

3. **Порядок освобождения**
   - LIFO (Last In, First Out) — стековый порядок
   - Вложенные контексты освобождаются корректно

4. **Отличия от других подходов**
   - **vs try/finally**: Меньше кода, автоматическая очистка
   - **vs FinalizationRegistry**: Детерминированное время освобождения
   - **vs GC**: Явный контроль, не зависит от сборщика мусора

### Когда использовать

✅ **Используйте `using`:**
- Файловые дескрипторы
- Сетевые соединения
- Блокировки и мьютексы
- Временные ресурсы
- Трассировка и профилирование

✅ **Используйте `await using`:**
- Когда нужно дождаться асинхронной очистки
- Сброс кэша в БД
- Отправка метрик
- Корректное закрытие соединений

❌ **Не используйте для:**
- Обычных объектов без внешних ресурсов
- Объектов, управляемых GC
- Ситуаций, где нужен reference counting (требует доп. обертки)

### Преимущества

- **Читаемость**: Меньше boilerplate кода
- **Надежность**: Автоматическая очистка даже при ошибках
- **Предсказуемость**: Детерминированное время освобождения
- **Композируемость**: Легко работать с множественными ресурсами

### Ограничения

- Нет автоматического reference counting
- Требует явной реализации `Symbol.dispose` / `Symbol.asyncDispose`
- Относительно новая возможность (Node.js 24+, современные браузеры)

### Дальнейшее изучение

Лектор обещает следующие темы:
1. **Reference Counting** — реализация подсчета ссылок поверх `using`
2. **Расширенные паттерны** — сложные сценарии использования
3. **Интеграция с фреймворками** — примеры из реальных проектов

---

## Полезные ссылки

### Документация

- [TypeScript 5.2 Release Notes - using](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html)
- [ECMAScript Explicit Resource Management Proposal](https://github.com/tc39/proposal-explicit-resource-management)
- [MDN: FinalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry)

### Флаги для запуска примеров

```bash
# Включить ручной вызов Garbage Collection
node --expose-gc script.js

# Проверить версию Node.js
node --version  # Должна быть 24+
```

### Типы TypeScript

```typescript
interface Disposable {
  [Symbol.dispose](): void;
}

interface AsyncDisposable {
  [Symbol.asyncDispose](): void | Promise<void>;
}
```

---

## Практические задания

### Задание 1: Базовое использование
Создайте класс `DatabaseConnection` с методом `Symbol.dispose`, который логирует закрытие соединения.

### Задание 2: Асинхронная очистка
Реализуйте класс `CachedLogger`, который при dispose:
1. Сбрасывает буфер в файл
2. Закрывает файл
3. Отправляет метрики (асинхронно)

### Задание 3: Вложенные контексты
Напишите функцию с вложенными `using` блоками и проверьте порядок освобождения.

### Задание 4: Обработка ошибок
Создайте класс, который выбрасывает ошибку в `dispose`, и обработайте `SuppressedError`.

---

## Заключение

Механизм явного управления ресурсами через `using` и `Symbol.dispose` — это мощное дополнение к JavaScript/TypeScript, которое:

- Делает код чище и безопаснее
- Гарантирует освобождение ресурсов
- Упрощает работу со сложными сценариями
- Приближает JavaScript к языкам с RAII (Resource Acquisition Is Initialization)

Начинайте использовать эту возможность в новых проектах уже сейчас!

---

**Дата создания конспекта**: 2025-12-09
**Версия Node.js**: 24+
**Версия TypeScript**: 5.2+
