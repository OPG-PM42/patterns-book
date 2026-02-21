# Паттерны инстанцирования: Пул объектов и Фабрика

## Обзор лекции

В этой лекции рассматриваются фундаментальные паттерны инстанцирования объектов в JavaScript/TypeScript: паттерн **Фабрика (Factory)** и **Пул объектов (Object Pool)**. Изучаются различные способы создания экземпляров объектов, от простых конструкторов до сложных пулов с асинхронным управлением ресурсами.

---

## 1. Основы инстанцирования объектов

### 1.1 Конструктор класса

Самый базовый способ создания экземпляров - через конструктор класса с оператором `new`.

#### Пример: Простой конструктор

```javascript
class Connection {
  constructor(url) {
    this.url = url;
  }
}

// Создание экземпляра через конструктор
const conn = new Connection('http://example.com');
```

**Ключевая особенность**: Использование оператора `new` создаёт новый экземпляр класса.

---

## 2. Паттерн Фабрика (Factory)

### 2.1 Что такое Фабрика?

**Фабрика** - это любая функция или метод, которая возвращает экземпляр объекта. В JavaScript фабрики могут принимать различные формы:

- Обычные функции, возвращающие литералы объектов
- Статические методы класса
- Функции, использующие метапрограммирование
- Функции с замыканиями

### 2.2 Виды фабрик в JavaScript

#### 2.2.1 Возвращение литералов объектов

```javascript
// Простая фабрика, возвращающая объектный литерал
function createUser(name, age) {
  return {
    name,
    age,
    greet() {
      console.log(`Привет, я ${this.name}`);
    }
  };
}

const user = createUser('Иван', 25);
```

#### 2.2.2 Статический метод класса

```javascript
class Connection {
  constructor(url) {
    this.url = url;
  }

  // Статический метод-фабрика
  static create(url) {
    return new Connection(url);
  }
}

const conn = Connection.create('http://example.com');
```

**Преимущества**:
- Инкапсуляция логики создания
- Чистый и понятный интерфейс
- Возможность добавления дополнительной логики при создании

#### 2.2.3 Фабрика через метапрограммирование

Фабрики могут создавать объекты используя:
- `Object.setPrototypeOf()`
- `Object.defineProperties()`
- Mixins (примеси)
- Прототипное наследование

```javascript
// Пример с метапрограммированием
function createConfigurable(data) {
  const obj = {};
  Object.defineProperties(obj, {
    value: {
      value: data,
      writable: false,
      enumerable: true
    }
  });
  return obj;
}
```

**Важно**: В прикладном коде желательно использовать более простые фабрики. Метапрограммирование допустимо в платформенном коде.

### 2.3 Универсальная фабрика высшего порядка

Можно создать универсальную функцию, которая превращает любой класс в фабрику.

#### Пример: Функция `factory`

```javascript
// Универсальная фабрика для любого класса
const factory = (Class) => (...args) => new Class(...args);

// Использование
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

// Создаём фабрику для класса Point
const createPoint = factory(Point);

// Используем фабрику
const p1 = createPoint(10, 20);
const p2 = createPoint(30, 40);
```

**Особенности**:
- Использует замыкание для захвата класса
- Применима к любому классу
- Возвращает стрелочную функцию, создающую экземпляры
- Очень универсальный подход

---

## 3. Паттерн Пул объектов (Object Pool)

### 3.1 Концепция пула объектов

**Пул объектов** - это паттерн, который:
1. Заранее создаёт определённое количество экземпляров
2. Хранит их в коллекции
3. Выдаёт экземпляры по мере необходимости
4. Принимает обратно использованные экземпляры для переиспользования

**Цель**: Экономия времени на создании новых объектов и переиспользование существующих.

### 3.2 Простейший пул объектов

#### Пример: Базовая реализация пула

```javascript
// Простейший пул объектов
const pool = (factory) => {
  const items = [];

  // Заполняем пул начальными экземплярами
  for (let i = 0; i < 10; i++) {
    items.push(factory());
  }

  // Возвращаем функцию для получения/возврата экземпляров
  return (item) => {
    if (item) {
      // Возврат экземпляра в пул
      items.push(item);
      console.log(`Возвращён в пул. Размер: ${items.length}`);
    } else {
      // Получение экземпляра из пула
      const instance = items.pop();
      console.log(`Получен из пула. Осталось: ${items.length}`);
      return instance;
    }
  };
};

// Фабрика для массивов
const createArray = () => new Array(1000).fill(0);

// Создаём пул массивов
const arrayPool = pool(createArray);

// Получаем массив из пула
const arr1 = arrayPool();
console.log(arr1.reduce((a, b) => a + b, 0)); // 0

// Возвращаем массив обратно
arrayPool(arr1);

// Получаем снова (будет тот же самый переиспользованный)
const arr2 = arrayPool();
```

**Ключевые моменты**:
- Одна функция для получения и возврата
- Если передан аргумент - возврат в пул
- Если без аргумента - получение из пула
- Использует `pop()` и `push()` для работы с массивом

### 3.3 Пул с фабрикой для конкретного типа

#### Пример: Пул подключений (Connection Pool)

```javascript
// Класс Connection
class Connection {
  constructor(index) {
    this.url = `http://example.com/db${index}`;
  }
}

// Фабрика подключений с счётчиком
class Factory {
  constructor() {
    this.index = 0;
  }

  create() {
    this.index++;
    return new Connection(this.index);
  }
}

// Создаём пул
const factory = new Factory();
const connectionPool = pool(() => factory.create());

// Использование
const conn1 = connectionPool();
console.log(conn1.url); // http://example.com/db1

const conn2 = connectionPool();
console.log(conn2.url); // http://example.com/db2

// Возвращаем в пул
connectionPool(conn1);
connectionPool(conn2);
```

### 3.4 Альтернативная фабрика с замыканием

Вместо класса фабрики можно использовать замыкание с IIFE (Immediately Invoked Function Expression).

#### Пример: Фабрика на замыканиях

```javascript
// Альтернативная фабрика через замыкание
const alternativeFactory = (() => {
  let index = 0;

  return () => {
    index++;
    return new Connection(index);
  };
})();

// Использование
const conn = alternativeFactory();
console.log(conn.url); // http://example.com/db1
```

**Сравнение подходов**:
- **Класс**: Более явный и понятный код
- **Замыкание**: Более короткий, но может быть запутанным
- **Функциональный подход**: Хранит мутабельное состояние в замыкании (не чистое ФП)

**Важно**: Функциональные программисты скажут, что хранение мутабельных значений в замыкании - это не настоящее функциональное программирование, и будут правы. Это просто способ создать контекст с мутируемым замыканием.

### 3.5 Пул с интегрированной фабрикой

Фабрику можно интегрировать прямо в класс как статический метод.

#### Пример: Класс с встроенной фабрикой

```javascript
class Connection {
  static index = 0;

  constructor(index) {
    this.url = `http://example.com/db${index}`;
  }

  // Статический метод-фабрика
  static create() {
    Connection.index++;
    return new Connection(Connection.index);
  }
}

// Пул с привязкой к конкретному классу
const poolForConnection = () => {
  const items = [];

  // Заполняем пул используя статический метод
  for (let i = 0; i < 10; i++) {
    items.push(Connection.create());
  }

  return (item) => {
    if (item) {
      items.push(item);
    } else {
      return items.pop();
    }
  };
};

const pool = poolForConnection();
const conn = pool();
console.log(conn.url); // http://example.com/db1
```

**Особенность**: Этот пул менее универсален, так как жёстко привязан к классу `Connection`.

---

## 4. Продвинутые возможности пулов

### 4.1 Пул с динамическим созданием

Если пул исчерпан, можно создавать новые экземпляры через фабрику.

#### Пример: Пул с автоматическим расширением

```javascript
const poolWithFactory = (factory, initialSize = 10) => {
  const items = [];

  // Начальное заполнение
  for (let i = 0; i < initialSize; i++) {
    items.push(factory());
  }

  return (item) => {
    if (item) {
      // Возврат в пул
      items.push(item);
      console.log(`Returned. Size: ${items.length}`);
    } else {
      // Получение или создание нового
      const instance = items.pop() || factory();
      console.log(`Retrieved. Remaining: ${items.length}`);
      return instance;
    }
  };
};

// Использование
const createBuffer = () => new Uint32Array(1024);
const bufferPool = poolWithFactory(createBuffer, 5);

// Забираем больше, чем было изначально
for (let i = 0; i < 10; i++) {
  const buffer = bufferPool();
  console.log(`Buffer ${i + 1}`);
}
```

**Логика**:
- `items.pop()` возвращает экземпляр из пула
- Если пул пуст (`pop()` вернёт `undefined`), создаётся новый через `factory()`
- Оператор `||` обеспечивает fallback на создание нового

### 4.2 Пул с ограничениями размера

Реальные пулы должны иметь три параметра размера:

1. **Минимальный размер (min)** - меньше которого нельзя опускаться
2. **Нормальный размер (normal)** - размер при создании пула
3. **Максимальный размер (max)** - выше которого нельзя подниматься

#### Пример: Пул с тройным ограничением размера

```javascript
const poolWithLimits = (factory, min = 5, normal = 10, max = 15) => {
  const items = [];

  // Заполняем до нормального размера
  for (let i = 0; i < normal; i++) {
    items.push(factory());
  }

  return (item) => {
    if (item) {
      // Возврат в пул, но не превышаем максимум
      if (items.length < max) {
        items.push(item);
        console.log(`Returned. Size: ${items.length}`);
      } else {
        console.log(`Pool full (max: ${max}). Item discarded.`);
      }
    } else {
      // Получение из пула
      if (items.length > 0) {
        return items.pop();
      } else if (items.length === 0) {
        // Создаём новый, если пул пуст, но следим за лимитами
        console.log('Pool empty, creating new instance');
        return factory();
      } else {
        // Пул исчерпан и достиг максимума
        console.log('Pool exhausted, no instances available');
        return undefined;
      }
    }
  };
};

// Использование с типизированными массивами
const createTypedArray = () => new Uint32Array(4096); // 4KB буфер

const pool = poolWithLimits(createTypedArray, 5, 10, 15);

// Тестирование
for (let i = 0; i < 20; i++) {
  const buffer = pool();
  console.log(`Iteration ${i}, buffer:`, buffer ? 'received' : 'null');
}
```

**Применение**:
- **Минимальный**: Гарантирует, что всегда есть готовые экземпляры
- **Нормальный**: Оптимальное количество для большинства сценариев
- **Максимальный**: Предотвращает неограниченный рост и переполнение памяти

### 4.3 Практический пример: Пул буферов

#### Пример: Пул для перекодирования данных

```javascript
// Фабрика для создания буферов фиксированного размера
const createBuffer = () => new Uint32Array(4096); // 16KB буфер (4096 * 4 байта)

// Создаём пул из 10 буферов
const bufferPool = poolWithLimits(createBuffer, 5, 10, 20);

// Симуляция перекодирования видео/крипто данных
function processData(data) {
  const buffer = bufferPool(); // Получаем буфер

  if (!buffer) {
    throw new Error('No buffers available');
  }

  try {
    // Используем буфер для обработки
    buffer.set(data);
    // ... обработка данных ...
    return buffer;
  } finally {
    // Возвращаем буфер в пул
    bufferPool(buffer);
  }
}

// Пример использования
const testData = new Uint32Array(100).fill(42);
const result = processData(testData);
```

**Применение пулов буферов**:
- Перекодирование видео
- Криптографические операции
- Обработка больших файлов
- Работа с сетевыми пакетами

---

## 5. Асинхронные пулы

### 5.1 Зачем нужны асинхронные пулы?

**Проблема синхронных пулов**: Если пул исчерпан, клиент либо получает `undefined`, либо создаётся новый экземпляр, что может нарушить ограничения.

**Решение**: Асинхронный пул позволяет клиенту **ждать**, пока кто-то вернёт экземпляр в пул.

### 5.2 Концепция асинхронного пула

Асинхронный пул:
1. Принимает запросы на получение экземпляра
2. Если пул пуст, запрос помещается в очередь ожидания
3. Когда кто-то возвращает экземпляр, он передаётся первому ожидающему
4. Поддерживает таймауты для предотвращения бесконечного ожидания

#### Концептуальная схема

```javascript
// Псевдокод асинхронного пула
const asyncPool = (factory, size) => {
  const items = [];
  const waitQueue = []; // Очередь ожидающих запросов

  // Инициализация
  for (let i = 0; i < size; i++) {
    items.push(factory());
  }

  return {
    // Получение с ожиданием
    async acquire(timeout = 5000) {
      if (items.length > 0) {
        return items.pop();
      }

      // Создаём промис для ожидания
      return new Promise((resolve, reject) => {
        const timer = setTimeout(() => {
          reject(new Error('Timeout waiting for pool item'));
        }, timeout);

        waitQueue.push({ resolve, reject, timer });
      });
    },

    // Возврат в пул
    release(item) {
      if (waitQueue.length > 0) {
        // Есть ожидающие - отдаём им сразу
        const { resolve, timer } = waitQueue.shift();
        clearTimeout(timer);
        resolve(item);
      } else {
        // Никто не ждёт - возвращаем в пул
        items.push(item);
      }
    }
  };
};
```

### 5.3 Использование асинхронного пула

```javascript
// Пример использования
const connPool = asyncPool(() => new Connection(), 5);

async function doWork() {
  const conn = await connPool.acquire(); // Ждём, если нужно

  try {
    // Работаем с подключением
    await conn.query('SELECT * FROM users');
  } finally {
    // ОБЯЗАТЕЛЬНО возвращаем в пул
    connPool.release(conn);
  }
}

// Запуск множества параллельных операций
for (let i = 0; i < 20; i++) {
  doWork().catch(console.error);
}
```

### 5.4 Преимущества асинхронных пулов

1. **Контроль ресурсов**: Не превышается максимальный размер пула
2. **Эффективное использование**: Все экземпляры постоянно в работе
3. **Управление нагрузкой**: Естественное throttling через ожидание
4. **Предсказуемость**: Гарантированное ограничение потребления памяти

**Важно**: Асинхронные пулы рассматриваются в отдельной лекции более подробно.

---

## 6. Ключевые концепции и best practices

### 6.1 Когда использовать паттерн Фабрика?

✅ **Используйте Фабрику когда**:
- Логика создания объекта сложная
- Нужно скрыть детали конструирования
- Создание зависит от условий или конфигурации
- Требуется централизованное управление созданием

❌ **Не используйте Фабрику когда**:
- Создание объекта тривиально (достаточно `new`)
- Это добавляет ненужную сложность

### 6.2 Когда использовать паттерн Пул объектов?

✅ **Используйте Пул когда**:
- Создание объектов дорогое (подключения к БД, буферы)
- Объекты часто создаются и уничтожаются
- Нужно ограничить количество экземпляров
- Важна производительность и экономия памяти

❌ **Не используйте Пул когда**:
- Объекты лёгкие и быстро создаются
- Объекты имеют состояние, которое сложно сбрасывать
- Добавляет больше сложности, чем пользы

### 6.3 Общие рекомендации

1. **Простота в прикладном коде**: Избегайте метапрограммирования и сложной магии
2. **Платформенный vs прикладной**: Сложные техники допустимы в библиотеках, но не в бизнес-логике
3. **Именование**: Используйте понятные имена (`poolOfConnections`, `createUser`)
4. **Управление ресурсами**: Всегда возвращайте объекты в пул после использования
5. **Таймауты**: Добавляйте таймауты в асинхронных пулах
6. **Тестирование**: Пулы требуют тщательного тестирования edge cases

---

## 7. Сравнение подходов к инстанцированию

| Подход | Простота | Гибкость | Производительность | Применение |
|--------|----------|----------|-------------------|------------|
| **Конструктор** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | Простые объекты |
| **Фабрика-функция** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | Сложная логика создания |
| **Статический метод** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | Альтернативные конструкторы |
| **Пул объектов** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Дорогие ресурсы |
| **Асинхронный пул** | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Управление конкурентностью |

---

## 8. Итоги и выводы

### Основные takeaways:

1. **Инстанцирование начинается с конструктора**, но не заканчивается им
2. **Фабрики** предоставляют гибкость в создании объектов различными способами
3. **Универсальная фабрика** `factory(Class)` может превратить любой класс в фабричную функцию
4. **Пул объектов** экономит время и ресурсы через переиспользование
5. **Параметры пула** (min, normal, max) критичны для правильного управления ресурсами
6. **Асинхронные пулы** решают проблему ожидания и контроля конкурентности
7. **Простота важнее изощрённости** в прикладном коде

### Что дальше?

- Изучите **асинхронные пулы** в специальной лекции
- Рассмотрите примеры пулов подключений к БД
- Изучите паттерны управления жизненным циклом объектов
- Познакомьтесь с disposable-паттернами для автоматической очистки ресурсов

---

## 9. Дополнительные примеры

### Пример: Полная реализация продвинутого пула

```javascript
class AdvancedPool {
  constructor(factory, options = {}) {
    this.factory = factory;
    this.min = options.min || 5;
    this.normal = options.normal || 10;
    this.max = options.max || 15;
    this.items = [];
    this.allocated = 0;

    // Инициализация
    this._initialize();
  }

  _initialize() {
    for (let i = 0; i < this.normal; i++) {
      this.items.push(this.factory());
    }
  }

  acquire() {
    if (this.items.length > 0) {
      this.allocated++;
      return this.items.pop();
    }

    if (this.allocated < this.max) {
      this.allocated++;
      console.log(`Creating new instance. Total allocated: ${this.allocated}`);
      return this.factory();
    }

    throw new Error(`Pool exhausted. Max limit (${this.max}) reached.`);
  }

  release(item) {
    this.allocated--;

    if (this.items.length < this.max) {
      this.items.push(item);
    } else {
      console.log('Pool at max capacity, discarding item');
    }
  }

  get size() {
    return this.items.length;
  }

  get totalAllocated() {
    return this.allocated;
  }
}

// Использование
const pool = new AdvancedPool(
  () => new Uint32Array(1024),
  { min: 5, normal: 10, max: 20 }
);

const items = [];

// Забираем больше максимума
try {
  for (let i = 0; i < 25; i++) {
    items.push(pool.acquire());
  }
} catch (e) {
  console.error(e.message); // Pool exhausted
}

// Возвращаем
items.forEach(item => pool.release(item));
console.log(`Pool size after release: ${pool.size}`);
```

### Пример: Фабрика с валидацией

```javascript
class UserFactory {
  static create(userData) {
    // Валидация входных данных
    if (!userData.email || !userData.name) {
      throw new Error('Email and name are required');
    }

    // Нормализация
    const normalizedData = {
      email: userData.email.toLowerCase().trim(),
      name: userData.name.trim(),
      createdAt: new Date()
    };

    // Создание с дополнительной логикой
    return new User(normalizedData);
  }
}

class User {
  constructor(data) {
    Object.assign(this, data);
  }
}

// Использование
const user = UserFactory.create({
  email: '  JOHN@EXAMPLE.COM  ',
  name: '  John Doe  '
});

console.log(user.email); // john@example.com
console.log(user.name);  // John Doe
```

---

## Глоссарий

- **Инстанцирование (Instantiation)** - процесс создания экземпляра объекта
- **Фабрика (Factory)** - функция или метод для создания объектов
- **Пул объектов (Object Pool)** - коллекция готовых экземпляров для переиспользования
- **Замыкание (Closure)** - функция, захватывающая переменные из внешней области видимости
- **IIFE** - Immediately Invoked Function Expression (немедленно вызываемая функция)
- **Метапрограммирование** - техники создания объектов через манипуляции с прототипами и дескрипторами
- **Mixins (примеси)** - способ добавления функциональности через композицию

---

**Следующая лекция**: Асинхронные пулы объектов с Promise и async/await
