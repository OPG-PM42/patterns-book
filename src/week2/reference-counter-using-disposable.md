# Подсчет ссылок на ресурс при помощи Using и Disposable в JavaScript

## Обзор

Данная лекция посвящена реализации паттерна Reference Counter (счетчик ссылок) с использованием новых возможностей JavaScript — `using` и `disposable`. Эти механизмы появились недавно (Node.js v24, современные версии Chrome) и позволяют управлять жизненным циклом ресурсов более явно и безопасно, чем традиционные подходы.

**Основные темы:**
- Explicit Resource Management (явное управление ресурсами)
- Реализация Reference Counter на базе `using` и `disposable`
- Автоматическое освобождение ресурсов при выходе из блока
- Сравнение с подходом на базе Finalization Registry
- Практические примеры с файловыми дескрипторами и логированием

---

## Введение в Explicit Resource Management

### Что такое Explicit Resource Management?

**Explicit Resource Management** — это паттерн явного управления ресурсами, который позволяет гарантировать освобождение ресурсов (файлы, соединения, память) при выходе из области видимости.

### Ключевые концепции

**`using` declaration** — специальное объявление переменной, которое автоматически вызывает метод `Symbol.dispose` при выходе из блока.

**`Symbol.dispose`** — специальный символ, определяющий метод очистки ресурса.

**Disposable объект** — объект, имеющий метод `[Symbol.dispose]()`, который вызывается автоматически.

#### Базовый пример использования

```javascript
// Объект с методом dispose
const resource = {
  value: 'some data',
  [Symbol.dispose]() {
    console.log('Resource disposed');
    // Здесь происходит очистка ресурса
  }
};

// Использование using
{
  using myResource = resource;
  console.log(myResource.value); // 'some data'
  // При выходе из блока автоматически вызовется resource[Symbol.dispose]()
}
// Вывод: 'Resource disposed'
```

**Преимущества:**
- Автоматическое освобождение ресурсов
- Защита от утечек памяти
- Работает даже при выбросе исключений
- Более короткий и понятный синтаксис по сравнению с `try...finally`

---

## Reference Counter: Базовая реализация

### Концепция Reference Counter

**Reference Counter (счетчик ссылок)** — паттерн, который отслеживает количество активных ссылок на ресурс и автоматически освобождает ресурс, когда счетчик достигает нуля.

### Структура Reference Counter

```javascript
class ReferenceCounter {
  #resource = null;      // Сам ресурс (например, консоль)
  #context = null;       // Контекст с данными для финализации
  #counter = 0;          // Счетчик активных ссылок
  #create = null;        // Функция создания ресурса
  #dispose = null;       // Функция освобождения ресурса

  constructor(create, dispose) {
    this.#create = create;
    this.#dispose = dispose;
  }

  // Методы будут рассмотрены ниже
}
```

**Поля класса:**
- `#resource` — хранит созданный экземпляр ресурса
- `#context` — контекст с дополнительными данными (например, файловый дескриптор)
- `#counter` — текущее количество активных ссылок
- `#create` — функция-фабрика для создания ресурса
- `#dispose` — функция для освобождения ресурса

---

## Пример 7: Простой Reference Counter для логера

### Постановка задачи

Создать логер, который:
1. Записывает данные в файл
2. Автоматически открывает файл при первом использовании
3. Автоматически закрывает файл, когда все блоки завершены
4. Использует Reference Counter для отслеживания активных использований

### Реализация создания логера

```javascript
import { open } from 'node:fs';
import { Console } from 'node:console';

// Функция создания экземпляра логера
const createLogger = () => {
  // Открываем файл для записи
  const fd = open('output.log', 'w');

  // Создаем поток для записи
  const stream = fs.createWriteStream(null, { fd });

  // Создаем консоль, которая пишет в поток
  const console = new Console(stream);

  // Возвращаем ресурс и контекст
  return {
    resource: console,    // Сама консоль для использования
    context: { fd }       // Файловый дескриптор для закрытия
  };
};

// Функция финализации (освобождения) логера
const disposeLogger = (context) => {
  // Закрываем файловый дескриптор из контекста
  context.fd.close();
};

// Создаем Reference Counter для логера
const logger = new ReferenceCounter(createLogger, disposeLogger);
```

**Важные моменты:**
- Функция `createLogger` возвращает объект с двумя полями: `resource` и `context`
- `resource` — это то, что будет использоваться в коде (консоль)
- `context` — это данные для финализации (файловый дескриптор)
- Reference Counter универсален — он не привязан к конкретному типу ресурса

### Метод use() — создание Disposable

```javascript
class ReferenceCounter {
  // ... предыдущий код ...

  use() {
    // Увеличиваем счетчик ссылок
    this.#counter++;

    // Берем ресурс (консоль)
    const resource = this.#resource;

    // Создаем новый объект, наследующий от ресурса
    const disposable = Object.create(resource);

    // Добавляем метод dispose
    disposable[Symbol.dispose] = () => {
      // Уменьшаем счетчик при выходе из блока
      this.#counter--;

      // Если счетчик достиг нуля
      if (this.#counter === 0) {
        // Вызываем функцию освобождения ресурса
        this.#dispose(this.#context);

        // Очищаем ссылки
        this.#resource = null;
        this.#context = null;
      }
    };

    // Возвращаем disposable объект
    return disposable;
  }
}
```

**Механизм работы `use()`:**

1. **Инкремент счетчика:** при каждом вызове `use()` увеличивается `#counter`
2. **Создание disposable:** создается новый объект с прототипной ссылкой на ресурс
3. **Добавление `Symbol.dispose`:** в объект добавляется метод для автоматической очистки
4. **Декремент при dispose:** когда блок `using` завершается, счетчик уменьшается
5. **Финализация при нуле:** когда счетчик достигает 0, вызывается функция освобождения

### Использование в коде

```javascript
async function main() {
  // Блок 0: первое использование логера
  {
    using console0 = logger.use();
    console0.log('log 0');

    // Блок 1: вложенное использование
    {
      using console1 = logger.use();
      console1.log('log 1');

      // Блок 2: еще более вложенное использование
      {
        using console2 = logger.use();
        console2.log('log 2');

        // Сохраняем ссылку для использования вне блока
        rf3 = console2;
      } // dispose для console2 - счетчик: 3 -> 2
    } // dispose для console1 - счетчик: 2 -> 1
  } // dispose для console0 - счетчик: 1 -> 0, вызов disposeLogger

  // Блок 3: попытка использовать утекшую ссылку
  {
    using rf4 = rf3; // Это та же ссылка, но ресурс уже освобожден
    rf4.log('log 3'); // Запись НЕ произойдет - файл закрыт!
  }
}

const rf4 = await main();
console.log('After main');
rf4.log('log 4'); // Тоже НЕ запишется - ресурс освобожден
```

**Пошаговая трассировка:**

```
1. Открытие файла:
   - createLogger вызван
   - fd открыт, stream создан, console создан
   - counter = 0

2. Вход в блок 0:
   - use() вызван
   - counter = 1
   - console0.log('log 0') → запись в файл ✓

3. Вход в блок 1:
   - use() вызван
   - counter = 2
   - console1.log('log 1') → запись в файл ✓

4. Вход в блок 2:
   - use() вызван
   - counter = 3
   - console2.log('log 2') → запись в файл ✓
   - rf3 = console2 (сохранение ссылки)

5. Выход из блока 2:
   - Symbol.dispose вызван для console2
   - counter = 2

6. Выход из блока 1:
   - Symbol.dispose вызван для console1
   - counter = 1

7. Выход из блока 0:
   - Symbol.dispose вызван для console0
   - counter = 0
   - disposeLogger вызван
   - fd.close() - ФАЙЛ ЗАКРЫТ
   - resource = null, context = null

8. Использование rf3/rf4:
   - Ссылка существует, но ресурс уже освобожден
   - Попытки записи не дадут результата
```

### Результат выполнения

**Вывод в консоль:**
```
open
use (counter: 1)
dispose (counter: 0)
use (counter: 1)
use (counter: 2)
dispose (counter: 1)
dispose (counter: 0)
close
After main
```

**Содержимое output.log:**
```
log 0
log 1
log 2
```

**Что НЕ записалось:**
- `log 3` — ресурс уже освобожден
- `log 4` — ресурс уже освобожден

---

## Пример 8: Reference Counter с коллекцией файлов

### Постановка задачи

Создать логер, который может работать с **несколькими файлами одновременно**, где:
- Для каждого файла ведется свой счетчик ссылок
- Файл открывается только один раз при первом обращении
- Файл закрывается, когда его счетчик достигает нуля
- Разные блоки могут писать в разные файлы

### Структура данных для коллекции файлов

```javascript
class MultiFileLogger {
  // Коллекция открытых файлов
  // Ключ: имя файла (строка)
  // Значение: объект с полями { counter, fd, console }
  #files = new Map();

  constructor() {
    // Инициализация пустой коллекции
  }
}
```

**Структура записи в коллекции:**
```javascript
{
  'output.log': {
    counter: 2,           // Количество активных использований
    fd: <FileDescriptor>, // Файловый дескриптор
    console: <Console>    // Экземпляр консоли для записи
  },
  'output3.log': {
    counter: 1,
    fd: <FileDescriptor>,
    console: <Console>
  }
}
```

### Метод open() — открытие файла

```javascript
class MultiFileLogger {
  #files = new Map();

  open(fileName) {
    // Открываем файл для записи
    const fd = fs.openSync(fileName, 'w');

    // Создаем поток записи
    const stream = fs.createWriteStream(null, { fd });

    // Создаем консоль для этого потока
    const console = new Console(stream);

    // Возвращаем структуру для сохранения в коллекцию
    return {
      counter: 0,  // Начальное значение счетчика
      fd,          // Файловый дескриптор
      console      // Консоль для записи
    };
  }
}
```

### Метод use() — получение disposable для файла

```javascript
class MultiFileLogger {
  #files = new Map();

  use(fileName) {
    // Пытаемся найти файл в коллекции
    let instance = this.#files.get(fileName);

    // Если файл еще не открыт
    if (!instance) {
      // Открываем файл и получаем структуру
      instance = this.open(fileName);

      // Сохраняем в коллекцию
      this.#files.set(fileName, instance);
    }

    // Увеличиваем счетчик для этого файла
    instance.counter++;

    // Создаем disposable объект на базе консоли
    const disposable = Object.create(instance.console);

    // Добавляем метод dispose
    disposable[Symbol.dispose] = () => {
      // Уменьшаем счетчик
      instance.counter--;

      // Если счетчик достиг нуля
      if (instance.counter === 0) {
        // Закрываем файл
        instance.fd.close();

        // Удаляем из коллекции
        this.#files.delete(fileName);
      }
    };

    // Возвращаем disposable
    return disposable;
  }
}

// Создаем экземпляр логера
const logger = new MultiFileLogger();
```

**Логика работы `use(fileName)`:**

1. **Поиск в коллекции:** проверяем, открыт ли уже этот файл
2. **Ленивое открытие:** если файла нет, открываем его и добавляем в коллекцию
3. **Инкремент счетчика:** увеличиваем счетчик для данного файла
4. **Создание disposable:** создаем объект с методом `Symbol.dispose`
5. **Закрытие при нуле:** когда счетчик файла становится 0, файл закрывается и удаляется из коллекции

### Использование с несколькими файлами

```javascript
async function main() {
  // Блок 0: работа с output.log
  {
    using c0 = logger.use('output.log');
    c0.log('log 0');  // Файл открывается, counter = 1

    // Блок 1: вложенное использование того же файла
    {
      using c1 = logger.use('output.log');
      c1.log('log 1');  // Файл уже открыт, counter = 2

      // Блок 2: еще более вложенное использование
      {
        using c2 = logger.use('output.log');
        c2.log('log 2');  // counter = 3

        // Блок 3: работа с ДРУГИМ файлом
        {
          using c3 = logger.use('output3.log');
          c3.log('log 3');  // Новый файл открывается, его counter = 1
        } // dispose c3: output3.log counter = 0, файл закрывается

        // Сохраняем ссылку для демонстрации утечки
        rf = c2;
      } // dispose c2: output.log counter = 2
    } // dispose c1: output.log counter = 1
  } // dispose c0: output.log counter = 0, файл закрывается

  // Возвращаем утекшую ссылку
  return rf;
}

// Блок 4: использование утекшей ссылки
{
  const f1 = await main();
  using c4 = f1;
  c4.log('log 4');  // НЕ запишется - файл уже закрыт!
}

// Блок 5: попытка использовать после завершения блока 4
{
  using c5 = f1;
  c5.log('log 5');  // НЕ запишется - файл закрыт!
}
```

### Пошаговая трассировка выполнения

```
=== Начало выполнения ===

1. Вход в блок 0:
   logger.use('output.log')
   - Файл не найден в коллекции
   - open('output.log') вызван
   - files.set('output.log', { counter: 0, fd, console })
   - counter для output.log = 1
   - c0.log('log 0') → запись в output.log ✓

2. Вход в блок 1:
   logger.use('output.log')
   - Файл найден в коллекции
   - counter для output.log = 2
   - c1.log('log 1') → запись в output.log ✓

3. Вход в блок 2:
   logger.use('output.log')
   - Файл найден в коллекции
   - counter для output.log = 3
   - c2.log('log 2') → запись в output.log ✓
   - rf = c2 (сохранение ссылки)

4. Вход в блок 3:
   logger.use('output3.log')
   - Файл не найден в коллекции
   - open('output3.log') вызван
   - files.set('output3.log', { counter: 0, fd, console })
   - counter для output3.log = 1
   - c3.log('log 3') → запись в output3.log ✓

5. Выход из блока 3:
   - Symbol.dispose для c3
   - counter для output3.log = 0
   - fd.close() для output3.log
   - files.delete('output3.log')
   - output3.log ЗАКРЫТ ✓

6. Выход из блока 2:
   - Symbol.dispose для c2
   - counter для output.log = 2

7. Выход из блока 1:
   - Symbol.dispose для c1
   - counter для output.log = 1

8. Выход из блока 0:
   - Symbol.dispose для c0
   - counter для output.log = 0
   - fd.close() для output.log
   - files.delete('output.log')
   - output.log ЗАКРЫТ ✓

9. Использование в блоке 4:
   - f1 = rf (утекшая ссылка)
   - c4.log('log 4') → НЕ запишется (файл закрыт) ✗

10. Использование в блоке 5:
    - c5.log('log 5') → НЕ запишется (файл закрыт) ✗
```

### Результаты выполнения

**Вывод в консоль:**
```
open output.log
use output.log (counter: 1)
use output.log (counter: 2)
use output.log (counter: 3)
open output3.log
use output3.log (counter: 1)
dispose output3.log (counter: 0)
close output3.log
dispose output.log (counter: 2)
dispose output.log (counter: 1)
dispose output.log (counter: 0)
close output.log
After main
```

**Содержимое output.log:**
```
log 0
log 1
log 2
```

**Содержимое output3.log:**
```
log 3
```

**Что НЕ записалось:**
- `log 4` — output.log уже закрыт
- `log 5` — output.log уже закрыт

---

## Сравнение подходов: Finalization Registry vs Using/Disposable

### Finalization Registry

**Принцип работы:**
- Регистрирует callback для вызова при сборке мусора
- Зависит от работы Garbage Collector
- Недетерминированное время выполнения

#### Пример с Finalization Registry

```javascript
// Создаем registry для отслеживания освобождения ресурсов
const registry = new FinalizationRegistry((context) => {
  // Этот callback вызовется когда-то в будущем
  console.log('Finalization:', context.name);
  context.fd.close();
});

class LoggerWithRegistry {
  constructor(fileName) {
    const fd = fs.openSync(fileName, 'w');
    const stream = fs.createWriteStream(null, { fd });
    this.console = new Console(stream);

    // Регистрируем для финализации
    registry.register(this, { name: fileName, fd });
  }
}

// Использование
{
  const logger = new LoggerWithRegistry('output.log');
  logger.console.log('Hello');
  // logger выходит из области видимости
}

// Файл может закрыться через несколько секунд (или минут, или никогда)
// В зависимости от того, когда запустится GC
```

**Проблемы Finalization Registry:**

1. **Недетерминированность:**
   ```javascript
   {
     const logger = new LoggerWithRegistry('output.log');
     logger.console.log('Message');
   }

   // Файл может остаться открытым неопределенно долго
   fs.readFileSync('output.log'); // Может быть не готов!
   ```

2. **Может не вызваться вообще:**
   ```javascript
   {
     const logger = new LoggerWithRegistry('output.log');
     logger.console.log('Message');
   }

   // Если программа завершится до GC
   process.exit(0); // Файл может остаться незакрытым!
   ```

3. **Невозможно контролировать момент очистки:**
   ```javascript
   for (let i = 0; i < 1000; i++) {
     const logger = new LoggerWithRegistry(`file${i}.log`);
     logger.console.log(`Message ${i}`);
   }

   // Все 1000 файлов могут оставаться открытыми!
   // Достижение лимита файловых дескрипторов
   ```

### Using/Disposable

**Принцип работы:**
- Детерминированное освобождение ресурсов
- Вызывается сразу при выходе из блока
- Работает даже при исключениях

#### Пример с Using/Disposable

```javascript
class LoggerWithDisposable {
  #fd = null;

  constructor(fileName) {
    this.#fd = fs.openSync(fileName, 'w');
    const stream = fs.createWriteStream(null, { fd: this.#fd });
    this.console = new Console(stream);
  }

  [Symbol.dispose]() {
    // Вызовется ГАРАНТИРОВАННО при выходе из блока
    console.log('Disposing logger');
    this.#fd.close();
  }
}

// Использование
{
  using logger = new LoggerWithDisposable('output.log');
  logger.console.log('Hello');
} // Файл закрывается СРАЗУ И ГАРАНТИРОВАННО
```

**Преимущества Using/Disposable:**

1. **Детерминированность:**
   ```javascript
   {
     using logger = new LoggerWithDisposable('output.log');
     logger.console.log('Message');
   } // Файл закрыт ПРЯМО ЗДЕСЬ

   // Файл точно закрыт
   fs.readFileSync('output.log'); // Всегда готов для чтения
   ```

2. **Работает с исключениями:**
   ```javascript
   try {
     using logger = new LoggerWithDisposable('output.log');
     logger.console.log('Message');
     throw new Error('Something went wrong');
   } catch (err) {
     // Symbol.dispose ВСЁ РАВНО вызовется
     // Файл точно закрыт
   }
   ```

3. **Контролируемое управление ресурсами:**
   ```javascript
   for (let i = 0; i < 1000; i++) {
     using logger = new LoggerWithDisposable(`file${i}.log`);
     logger.console.log(`Message ${i}`);
     // Файл закрывается на каждой итерации
   } // Максимум 1 открытый файл одновременно
   ```

### Сравнительная таблица

| Характеристика | Finalization Registry | Using/Disposable |
|----------------|----------------------|------------------|
| **Время вызова** | Недетерминированное (зависит от GC) | Детерминированное (сразу при выходе из блока) |
| **Гарантия вызова** | Может не вызваться | Гарантированно вызывается |
| **Работа с исключениями** | Может не сработать при ранних выходах | Работает всегда (как `finally`) |
| **Предсказуемость** | Низкая | Высокая |
| **Синтаксис** | Многословный | Краткий (`using`) |
| **Использование в продакшене** | Только для некритичных случаев | Рекомендуется для управления ресурсами |
| **Задержка освобождения** | От секунд до бесконечности | Мгновенно |
| **Контроль памяти** | Слабый | Полный |

### Когда использовать каждый подход

**Finalization Registry:**
- Кеширование, где важна производительность
- Очистка некритичных данных
- Дополнительная подстраховка (но не основной механизм)

```javascript
// Пример: кеш с автоочисткой
const cache = new Map();
const registry = new FinalizationRegistry((key) => {
  cache.delete(key);
});

function cacheValue(key, value) {
  cache.set(key, value);
  registry.register(value, key);
}
```

**Using/Disposable:**
- Файловые дескрипторы
- Сетевые соединения
- Блокировки и мьютексы
- Транзакции базы данных
- Любые критичные ресурсы, требующие своевременного освобождения

```javascript
// Пример: транзакция базы данных
async function updateUser(userId, data) {
  using transaction = await db.beginTransaction();
  try {
    await transaction.update('users', { id: userId }, data);
    await transaction.commit();
  } catch (err) {
    // transaction.dispose() вызовется автоматически
    // и сделает rollback
  }
}
```

---

## Преимущества и ограничения Reference Counter

### Преимущества

#### 1. Автоматическое управление жизненным циклом

```javascript
// БЕЗ Reference Counter - ручное управление
function withoutRefCounter() {
  const fd = fs.openSync('file.log', 'w');
  const stream = fs.createWriteStream(null, { fd });
  const logger = new Console(stream);

  try {
    logger.log('Message 1');

    try {
      logger.log('Message 2');
      // ... вложенная логика
    } finally {
      // Нужно отслеживать вручную
    }

    logger.log('Message 3');
  } finally {
    // Закрываем только здесь
    fd.close();
  }
}

// С Reference Counter - автоматически
function withRefCounter() {
  {
    using logger1 = logger.use('file.log');
    logger1.log('Message 1');

    {
      using logger2 = logger.use('file.log');
      logger2.log('Message 2');
    } // Счетчик уменьшается автоматически

    logger1.log('Message 3');
  } // Файл закрывается автоматически
}
```

#### 2. Безопасность при исключениях

```javascript
// Даже при ошибках ресурсы освобождаются
async function processLogs() {
  {
    using logger = logger.use('output.log');
    logger.log('Start processing');

    // Может выбросить ошибку
    await riskyOperation();

    logger.log('End processing');
  } // dispose вызовется ДАЖЕ если riskyOperation бросит ошибку
}
```

#### 3. Защита от утечек ресурсов

```javascript
// Невозможно забыть закрыть ресурс
function safeResourceUsage() {
  {
    using logger = logger.use('file.log');
    logger.log('Message');

    // Даже если забыли вручную закрыть - не проблема
    // return; // Ранний выход - dispose всё равно вызовется
  }
}
```

### Ограничения и проблемы

#### 1. Утечшие ссылки не работают

```javascript
let leakedReference;

{
  using logger = logger.use('file.log');
  logger.log('Message');
  leakedReference = logger; // Сохраняем ссылку
} // Файл закрывается, счетчик = 0

// Ссылка существует, но бесполезна
leakedReference.log('This will not work'); // Файл уже закрыт!
```

**Решение:** не сохранять ссылки на disposable объекты вне блока `using`.

#### 2. Не рекомендуется для продакшена (пока)

```javascript
// Это прототип, не используйте в продакшене
// Нужно больше тестирования и доработки
const logger = new ReferenceCounter(/* ... */);

// TODO:
// - Обработка race conditions
// - Поддержка async dispose
// - Более надежная обработка ошибок
// - Тестирование edge cases
```

#### 3. Необходима поддержка браузером/Node.js

```javascript
// Проверка поддержки
if (typeof Symbol.dispose === 'undefined') {
  console.error('Using/Disposable not supported!');
  // Нужен fallback или polyfill
}
```

---

## Практические рекомендации

### 1. Структура проекта с Reference Counter

```javascript
// logger.js - модуль логирования
import { ReferenceCounter } from './reference-counter.js';

export const logger = new ReferenceCounter(
  // Создание ресурса
  () => {
    const fd = fs.openSync('app.log', 'a');
    const stream = fs.createWriteStream(null, { fd });
    const console = new Console(stream);
    return { resource: console, context: { fd } };
  },
  // Освобождение ресурса
  (context) => {
    context.fd.close();
  }
);

// app.js - использование
import { logger } from './logger.js';

export async function processRequest(request) {
  using log = logger.use();
  log.log('Request started:', request.id);

  try {
    await handleRequest(request);
    log.log('Request completed:', request.id);
  } catch (err) {
    log.error('Request failed:', err);
    throw err;
  }
  // dispose вызовется автоматически
}
```

### 2. Множественные ресурсы

```javascript
// Управление несколькими ресурсами одновременно
async function complexOperation() {
  using dbConnection = connectionPool.use();
  using fileLogger = logger.use('operation.log');
  using lockHandle = lockManager.use('operation-lock');

  // Все три ресурса будут автоматически освобождены
  await performOperation(dbConnection, fileLogger);

} // dispose вызовется для всех трех ресурсов
```

### 3. Правильная обработка ошибок

```javascript
class ReferenceCounter {
  use() {
    this.#counter++;
    const resource = this.#resource;
    const disposable = Object.create(resource);

    disposable[Symbol.dispose] = () => {
      this.#counter--;

      if (this.#counter === 0) {
        try {
          // Обрабатываем ошибки при dispose
          this.#dispose(this.#context);
        } catch (err) {
          // Логируем, но не пробрасываем
          console.error('Error during dispose:', err);
        } finally {
          // Всегда очищаем ссылки
          this.#resource = null;
          this.#context = null;
        }
      }
    };

    return disposable;
  }
}
```

### 4. Паттерн для ленивой инициализации

```javascript
class ReferenceCounter {
  #initialized = false;

  use() {
    // Инициализация при первом использовании
    if (!this.#initialized) {
      const { resource, context } = this.#create();
      this.#resource = resource;
      this.#context = context;
      this.#initialized = true;
    }

    this.#counter++;
    // ... остальной код
  }
}
```

---

## Диаграмма жизненного цикла Reference Counter

```
Состояние счетчика для output.log:

Блок 0 начало:  counter = 1  [FILE OPENED]
│
├── Блок 1 начало:  counter = 2
│   │
│   ├── Блок 2 начало:  counter = 3
│   │   │
│   │   └── Блок 2 конец:  counter = 2  [dispose вызван]
│   │
│   └── Блок 1 конец:  counter = 1  [dispose вызван]
│
└── Блок 0 конец:  counter = 0  [FILE CLOSED, dispose вызван]


Состояние счетчика для output3.log:

Блок 3 начало:  counter = 1  [FILE OPENED]
│
└── Блок 3 конец:  counter = 0  [FILE CLOSED, dispose вызван]
```

---

## Полный код реализации Reference Counter

### reference-counter.js

```javascript
/**
 * Reference Counter для управления жизненным циклом ресурсов
 * на основе механизма using/disposable
 */
export class ReferenceCounter {
  // Приватные поля
  #resource = null;   // Сам ресурс (объект для использования)
  #context = null;    // Контекст с данными для финализации
  #counter = 0;       // Счетчик активных ссылок
  #create = null;     // Функция создания ресурса
  #dispose = null;    // Функция освобождения ресурса

  /**
   * @param {Function} create - Функция создания ресурса
   *   Должна возвращать { resource, context }
   * @param {Function} dispose - Функция освобождения ресурса
   *   Принимает context в качестве параметра
   */
  constructor(create, dispose) {
    this.#create = create;
    this.#dispose = dispose;
  }

  /**
   * Создает disposable объект для использования с using
   * @returns {Object} Disposable объект с методом Symbol.dispose
   */
  use() {
    // Ленивая инициализация при первом использовании
    if (this.#resource === null) {
      const { resource, context } = this.#create();
      this.#resource = resource;
      this.#context = context;
    }

    // Увеличиваем счетчик ссылок
    this.#counter++;
    console.log(`use (counter: ${this.#counter})`);

    // Создаем новый объект, наследующий от ресурса
    const disposable = Object.create(this.#resource);

    // Добавляем метод dispose
    disposable[Symbol.dispose] = () => {
      // Уменьшаем счетчик
      this.#counter--;
      console.log(`dispose (counter: ${this.#counter})`);

      // Если счетчик достиг нуля - освобождаем ресурс
      if (this.#counter === 0) {
        try {
          this.#dispose(this.#context);
          console.log('close');
        } catch (err) {
          console.error('Error during dispose:', err);
        } finally {
          this.#resource = null;
          this.#context = null;
        }
      }
    };

    return disposable;
  }
}
```

### multi-file-logger.js

```javascript
import fs from 'node:fs';
import { Console } from 'node:console';

/**
 * Логер с поддержкой нескольких файлов
 * и reference counting для каждого файла
 */
export class MultiFileLogger {
  // Коллекция открытых файлов
  // Map<fileName, { counter, fd, console }>
  #files = new Map();

  /**
   * Открывает файл и создает консоль для записи
   * @param {string} fileName - Имя файла
   * @returns {Object} Структура { counter, fd, console }
   */
  open(fileName) {
    console.log(`open ${fileName}`);

    // Открываем файл для записи
    const fd = fs.openSync(fileName, 'w');

    // Создаем поток записи
    const stream = fs.createWriteStream(null, { fd });

    // Создаем консоль для этого потока
    const console = new Console(stream);

    // Возвращаем структуру
    return {
      counter: 0,
      fd,
      console
    };
  }

  /**
   * Получает disposable для записи в файл
   * @param {string} fileName - Имя файла
   * @returns {Object} Disposable объект консоли
   */
  use(fileName) {
    // Ищем файл в коллекции
    let instance = this.#files.get(fileName);

    // Если файл не открыт - открываем
    if (!instance) {
      instance = this.open(fileName);
      this.#files.set(fileName, instance);
    }

    // Увеличиваем счетчик
    instance.counter++;
    console.log(`use ${fileName} (counter: ${instance.counter})`);

    // Создаем disposable
    const disposable = Object.create(instance.console);

    // Добавляем метод dispose
    disposable[Symbol.dispose] = () => {
      instance.counter--;
      console.log(`dispose ${fileName} (counter: ${instance.counter})`);

      // Если счетчик достиг нуля - закрываем файл
      if (instance.counter === 0) {
        instance.fd.close();
        this.#files.delete(fileName);
        console.log(`close ${fileName}`);
      }
    };

    return disposable;
  }
}
```

### example-7.js

```javascript
import fs from 'node:fs';
import { Console } from 'node:console';
import { ReferenceCounter } from './reference-counter.js';

// Создание логера
const logger = new ReferenceCounter(
  // Функция создания
  () => {
    console.log('open');
    const fd = fs.openSync('output.log', 'w');
    const stream = fs.createWriteStream(null, { fd });
    const console = new Console(stream);
    return { resource: console, context: { fd } };
  },
  // Функция освобождения
  (context) => {
    context.fd.close();
  }
);

// Использование
async function main() {
  let rf3;

  // Блок 0
  {
    using console0 = logger.use();
    console0.log('log 0');

    // Блок 1
    {
      using console1 = logger.use();
      console1.log('log 1');

      // Блок 2
      {
        using console2 = logger.use();
        console2.log('log 2');
        rf3 = console2; // Сохраняем ссылку
      }
    }
  }

  // Блок 3 - утекшая ссылка
  {
    using rf4 = rf3;
    rf4.log('log 3'); // Не запишется
  }

  return rf3;
}

const rf4 = await main();
console.log('After main');
rf4.log('log 4'); // Не запишется
```

### example-8.js

```javascript
import { MultiFileLogger } from './multi-file-logger.js';

const logger = new MultiFileLogger();

async function main() {
  let rf;

  // Блок 0
  {
    using c0 = logger.use('output.log');
    c0.log('log 0');

    // Блок 1
    {
      using c1 = logger.use('output.log');
      c1.log('log 1');

      // Блок 2
      {
        using c2 = logger.use('output.log');
        c2.log('log 2');

        // Блок 3 - другой файл
        {
          using c3 = logger.use('output3.log');
          c3.log('log 3');
        }

        rf = c2;
      }
    }
  }

  return rf;
}

// Блок 4
{
  const f1 = await main();
  using c4 = f1;
  c4.log('log 4'); // Не запишется
}

console.log('After main');

// Блок 5
{
  using c5 = f1;
  c5.log('log 5'); // Не запишется
}
```

---

## Ключевые выводы

### 1. Reference Counter решает важные проблемы

✓ **Автоматическое управление ресурсами:** не нужно вручную отслеживать открытие/закрытие
✓ **Безопасность при исключениях:** ресурсы освобождаются даже при ошибках
✓ **Защита от утечек:** невозможно забыть закрыть ресурс
✓ **Детерминированность:** ресурсы освобождаются сразу, а не когда-нибудь

### 2. Using/Disposable vs Finalization Registry

| Using/Disposable | Finalization Registry |
|------------------|----------------------|
| Детерминированное освобождение | Недетерминированное |
| Гарантия вызова | Может не вызваться |
| Мгновенное освобождение | Задержка до GC |
| Рекомендуется для критичных ресурсов | Только для некритичных случаев |

### 3. Практические советы

**Делайте:**
- Используйте `using` для файлов, соединений, транзакций
- Обрабатывайте ошибки в методе `dispose`
- Проверяйте поддержку `Symbol.dispose` в целевой среде

**Не делайте:**
- Не сохраняйте ссылки на disposable объекты вне блока `using`
- Не используйте текущую реализацию Reference Counter в продакшене (пока это прототип)
- Не полагайтесь на Finalization Registry для критичных ресурсов

### 4. Статус технологии

⚠️ **Текущий статус:** Прототип, экспериментальная реализация

**Доступность:**
- Node.js v24+
- Современные версии Chrome
- Требует проверки поддержки

**Будущее:**
- Доработка и тестирование
- Возможное включение в Metarhia technology stack
- Создание библиотеки с готовыми классами

---

## Дополнительные материалы

### Полезные ссылки

- [TC39 Proposal: Explicit Resource Management](https://github.com/tc39/proposal-explicit-resource-management)
- [MDN: Symbol.dispose](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/dispose)
- Исходники примеров под видео лекции

### Связанные темы для изучения

1. **Finalization Registry** - альтернативный подход к управлению ресурсами
2. **WeakRef и WeakMap** - слабые ссылки в JavaScript
3. **Try...finally** - классический подход к освобождению ресурсов
4. **RAII (Resource Acquisition Is Initialization)** - паттерн из C++, вдохновивший using/disposable
5. **Async Disposable** - асинхронная версия disposable (Symbol.asyncDispose)

### Практические задания

1. Реализуйте Reference Counter для пула соединений к базе данных
2. Создайте систему блокировок (locks) с автоматическим освобождением
3. Реализуйте файловый кеш с автоматической очисткой
4. Добавьте поддержку async dispose для асинхронных ресурсов

---

## Заключение

Механизм **using/disposable** представляет собой мощный инструмент для управления жизненным циклом ресурсов в JavaScript. В сочетании с паттерном **Reference Counter**, он позволяет создавать надежные и безопасные абстракции для работы с файлами, соединениями и другими критичными ресурсами.

**Основные преимущества подхода:**
- Детерминированное освобождение ресурсов
- Автоматическая защита от утечек
- Краткий и понятный синтаксис
- Безопасность при исключениях

**Текущие ограничения:**
- Относительно новая технология
- Требует поддержки современных версий платформ
- Прототипная реализация Reference Counter не готова для продакшена

Следите за развитием технологии и экспериментируйте с примерами, чтобы быть готовыми к ее широкому применению в будущем!
