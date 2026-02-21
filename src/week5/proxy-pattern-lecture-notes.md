# Паттерн Proxy (Прокси) — GoF и встроенный класс Proxy в JavaScript

## Обзор

Паттерн Proxy (Прокси) — это структурный паттерн проектирования из каталога Gang of Four, который позволяет перехватывать управление над абстракцией и оборачивать её. В JavaScript существует встроенный класс `Proxy`, который решает схожую задачу, но другим способом. Этот паттерн относится к семейству паттернов, включающему Adapter и Facade, которые имеют похожие цели и задачи.

## Основные концепции

### Суть паттерна Proxy

**Основная задача** паттерна Proxy — **перехват управления над абстракцией и её обёртка**.

Можно оборачивать любую абстракцию:
- Функциональную (функторы, функции)
- Объектную (экземпляры классов, объекты)
- Любые другие сущности

### Различия между GoF Proxy и встроенным Proxy в JavaScript

| Аспект | GoF Proxy | JavaScript Proxy |
|--------|-----------|------------------|
| **Реализация** | Через наследование и делегирование | Через встроенный механизм перехватчиков |
| **Производительность** | Более легковесный, меньше накладных расходов | Больший overhead на производительность |
| **Гибкость** | Требует явной реализации интерфейса | Может перехватывать любые операции |
| **Сложность кода** | Больше кода, сложнее структура | Компактнее, но с ограничениями |

## Преимущества паттерна Proxy

### 1. Добавление функциональности без наследования

Вместо создания наследника класса и увеличения дерева наследования, можно:
- Упростить задачу через композицию
- Избежать создания множества классов
- Применить одну и ту же функциональность к десяткам классов

**Пример проблемы с наследованием:**
```javascript
// При наследовании: одна функциональность = один новый класс
class BaseClass {
  doSomething() { }
}

class DerivedWithLogging extends BaseClass {
  doSomething() {
    console.log('Logging...');
    super.doSomething();
  }
}

// Нужна та же логика для другого класса? Ещё один наследник!
class AnotherDerivedWithLogging extends AnotherClass {
  doSomething() {
    console.log('Logging...');
    super.doSomething();
  }
}
```

**Решение через Proxy:**
```javascript
// Один прокси для многих классов с подходящим интерфейсом
class LoggingProxy {
  constructor(target) {
    this.target = target;
  }

  doSomething() {
    console.log('Logging...');
    return this.target.doSomething();
  }
}

// Применяем к любому объекту
const proxy1 = new LoggingProxy(new BaseClass());
const proxy2 = new LoggingProxy(new AnotherClass());
```

### 2. Добавление кросс-функциональных возможностей

Можно добавлять различные эффекты:
- **Логирование** (logging)
- **Телеметрия** (telemetry)
- **Кэширование**
- **Валидация**
- **Контроль доступа**

Это работает почти как примесь (mixin), но без фактического примешивания кода.

### 3. Сокращение количества классов

При использовании прокси можно обернуть множество объектов, подходящих под определённый интерфейс, без создания отдельного класса для каждого варианта.

## Недостатки паттерна Proxy

### 1. Накладные расходы на производительность

**Для встроенного JavaScript Proxy:**
- Очень большой penalty по производительности
- Дополнительные ресурсы памяти
- Дополнительные процессорные ресурсы
- Все вызовы оборачиваются

**Для GoF Proxy:**
- Меньше накладных расходов
- Более легковесный
- Но всё равно есть overhead

### 2. Усложнение кода

- Код становится больше
- Структура становится сложнее
- Появляются дополнительные уровни абстракции

## Встроенный класс Proxy в JavaScript

### Основы использования

JavaScript предоставляет встроенный класс `Proxy`, который позволяет перехватывать операции над объектами.

#### Пример: базовое использование Proxy

```javascript
const target = {
  name: 'John',
  age: 30
};

const handler = {
  get(target, property) {
    console.log(`Получение свойства: ${property}`);
    return target[property];
  },

  set(target, property, value) {
    console.log(`Установка свойства: ${property} = ${value}`);
    target[property] = value;
    return true;
  }
};

const proxy = new Proxy(target, handler);

console.log(proxy.name); // Логирует: "Получение свойства: name"
                         // Выводит: "John"
proxy.age = 31;          // Логирует: "Установка свойства: age = 31"
```

### Доступные перехватчики (traps)

Класс `Proxy` поддерживает множество перехватчиков:

- `get(target, property, receiver)` — чтение свойства
- `set(target, property, value, receiver)` — запись свойства
- `has(target, property)` — оператор `in`
- `deleteProperty(target, property)` — оператор `delete`
- `apply(target, thisArg, argumentsList)` — вызов функции
- `construct(target, argumentsList, newTarget)` — оператор `new`
- `getPrototypeOf(target)` — получение прототипа
- `setPrototypeOf(target, prototype)` — установка прототипа
- `isExtensible(target)` — проверка расширяемости
- `preventExtensions(target)` — запрет расширения
- `getOwnPropertyDescriptor(target, property)` — получение дескриптора
- `defineProperty(target, property, descriptor)` — определение свойства
- `ownKeys(target)` — получение списка ключей

#### Пример: перехват вызова функции

```javascript
function sum(a, b) {
  return a + b;
}

const handler = {
  apply(target, thisArg, argumentsList) {
    console.log(`Вызов функции с аргументами: ${argumentsList}`);
    const result = target.apply(thisArg, argumentsList);
    console.log(`Результат: ${result}`);
    return result;
  }
};

const proxySum = new Proxy(sum, handler);

proxySum(5, 3);
// Логирует: "Вызов функции с аргументами: 5,3"
// Логирует: "Результат: 8"
// Возвращает: 8
```

## Паттерн Proxy из Gang of Four

### Структура паттерна

GoF Proxy реализуется через **делегирование** — класс-прокси реализует тот же интерфейс, что и оборачиваемый класс, и делегирует ему вызовы.

#### Базовый пример структуры

```javascript
// Интерфейс (в TypeScript)
interface IService {
  operation(a: number, b: number): { result: number; message: string };
}

// Реальный класс
class RealService implements IService {
  operation(a: number, b: number) {
    return {
      result: a + b,
      message: 'Operation completed'
    };
  }
}

// Прокси класс
class ServiceProxy implements IService {
  private realService: RealService;

  constructor(realService: RealService) {
    this.realService = realService;
  }

  operation(a: number, b: number) {
    // Здесь можно добавить дополнительную логику
    console.log(`Вызов operation с аргументами: ${a}, ${b}`);

    // Делегирование реальному объекту
    const result = this.realService.operation(a, b);

    console.log(`Результат: ${result.result}`);
    return result;
  }
}

// Использование
const service = new RealService();
const proxy = new ServiceProxy(service);

proxy.operation(5, 3);
// Логирует: "Вызов operation с аргументами: 5, 3"
// Логирует: "Результат: 8"
// Возвращает: { result: 8, message: 'Operation completed' }
```

### Ключевые элементы реализации

1. **Общий интерфейс** — и прокси, и реальный объект реализуют один интерфейс
2. **Инкапсуляция объекта** — прокси хранит ссылку на реальный объект (в приватном свойстве)
3. **Делегирование** — прокси вызывает методы реального объекта
4. **Дополнительная логика** — прокси может добавлять функциональность до и после делегирования

## Практические примеры

### Пример 1: Config с ленивой загрузкой

#### Реальный класс Config

```javascript
// config.js
const fs = require('fs');

class Config {
  constructor(filePath) {
    // Загрузка файла в конструкторе
    const fileContent = require(filePath);
    this.data = this.parseConfig(fileContent);
  }

  parseConfig(content) {
    // Парсинг конфигурации
    return content;
  }

  read(key) {
    return this.data[key];
  }
}

module.exports = Config;
```

#### Прокси с ленивой загрузкой

```javascript
// config-proxy.js
const Config = require('./config');

class ConfigProxy {
  constructor(filePath) {
    this.filePath = filePath;
    this.config = null; // Ленивая инициализация
  }

  read(key) {
    // Создаём экземпляр только при первом обращении
    if (!this.config) {
      console.log('Загрузка конфигурации...');
      this.config = new Config(this.filePath);
    }

    // Логирование обращения к конфигурации
    console.log(`Чтение ключа: ${key}`);

    return this.config.read(key);
  }
}

module.exports = ConfigProxy;
```

#### Использование

```javascript
const ConfigProxy = require('./config-proxy');

const config = new ConfigProxy('./config.json');

// Конфигурация ещё не загружена
console.log('Config proxy создан');

// При первом обращении загружается файл
const dbHost = config.read('database.host');
// Логирует: "Загрузка конфигурации..."
// Логирует: "Чтение ключа: database.host"

// При повторном обращении файл не перезагружается
const dbPort = config.read('database.port');
// Логирует: "Чтение ключа: database.port"
```

**Преимущества этого подхода:**
- Файл загружается только когда действительно нужен
- Можно добавить логирование без изменения класса Config
- Можно добавить дополнительный парсинг данных
- Можно кэшировать результаты

### Пример 2: Прокси для файловых потоков с статистикой

Это более сложный и реалистичный пример, показывающий мощь паттерна Proxy.

#### Задача

Необходимо собирать статистику по работе с файловыми потоками:
- Количество байт, прочитанных из файлов
- Количество чанков данных
- Количество различных событий

#### Решение через наследование и замыкание модуля

```javascript
// fs-proxy.js
const fs = require('fs');

// Замыкание модуля — statistics будет локальным синглтоном
const statistics = {
  bytes: 0,
  chunks: 0,
  events: new Map() // Храним количество каждого типа события
};

// Наследуем от fs.ReadStream
class StatReadStream extends fs.ReadStream {
  emit(eventName, ...args) {
    // Собираем статистику перед вызовом родительского метода

    if (eventName === 'data') {
      const chunk = args[0];
      statistics.bytes += chunk.length;
      statistics.chunks++;
    }

    // Увеличиваем счётчик для этого типа события
    const count = statistics.events.get(eventName) || 0;
    statistics.events.set(eventName, count + 1);

    // Делегируем вызов родительскому классу (EventEmitter)
    return super.emit(eventName, ...args);
  }
}

// Фабрика для создания потоков
function createReadStream(path, options) {
  return new StatReadStream(path, options);
}

// Функция для получения статистики
function getStatistics() {
  // Возвращаем клон, чтобы никто не мог изменить статистику снаружи
  return {
    bytes: statistics.bytes,
    chunks: statistics.chunks,
    events: new Map(statistics.events)
  };
}

// Экспортируем всё из fs, кроме createReadStream
module.exports = {
  ...fs,
  createReadStream,
  getStatistics
};
```

#### Использование fs-proxy

```javascript
// app.js
// Вместо const fs = require('fs');
const fs = require('./fs-proxy');

// Создаём поток для чтения файла
const stream = fs.createReadStream('./large-file.txt');

stream.on('data', (chunk) => {
  console.log(`Получен чанк размером ${chunk.length} байт`);
});

stream.on('end', () => {
  console.log('Чтение завершено');

  // Получаем статистику
  const stats = fs.getStatistics();
  console.log('Статистика:');
  console.log(`  Байт прочитано: ${stats.bytes}`);
  console.log(`  Чанков получено: ${stats.chunks}`);
  console.log(`  События:`);

  stats.events.forEach((count, eventName) => {
    console.log(`    ${eventName}: ${count}`);
  });
});

// Другие методы fs работают как обычно
const files = fs.readdirSync('./');
console.log(files);

// Можно даже наследоваться от fs.ReadStream, если нужно
class CustomStream extends fs.ReadStream {
  // Ваша кастомная логика
}
```

#### Что делает этот прокси?

1. **Переопределяет только нужное** — только `createReadStream`, остальной API fs остаётся неизменным
2. **Использует наследование** — `StatReadStream` наследуется от `fs.ReadStream`
3. **Применяет делегирование** — вызывает `super.emit()` для делегирования родительскому классу
4. **Использует замыкание модуля** — `statistics` является локальным синглтоном модуля
5. **Защищает данные** — `getStatistics()` возвращает клон через структурное клонирование
6. **Прозрачен для пользователей** — код, использующий `fs`, почти не меняется

### Объяснение замыкания модуля

```javascript
// Каждый модуль в Node.js обернут в функцию:
(function(exports, require, module, __filename, __dirname) {

  // Эта переменная попадает в замыкание всех функций модуля
  const statistics = {
    bytes: 0,
    chunks: 0,
    events: new Map()
  };

  class StatReadStream extends fs.ReadStream {
    emit(eventName, ...args) {
      // Имеет доступ к statistics через замыкание
      statistics.bytes += chunk.length;
      return super.emit(eventName, ...args);
    }
  }

  function getStatistics() {
    // Также имеет доступ к statistics
    return { ...statistics };
  }

  // Если не экспортировать statistics, он остаётся приватным
  module.exports = {
    createReadStream,
    getStatistics
    // statistics НЕ экспортируется!
  };

})(/* параметры от Node.js */);
```

**Ключевые моменты:**
- Переменные на уровне модуля становятся локальными синглтонами
- Они доступны всем функциям и классам модуля через замыкание
- Если не экспортировать, они остаются приватными
- Это альтернатива статическим полям класса

## Сравнение подходов

### Фасад vs Прокси

В примере с `fs-proxy` мы создали что-то похожее на **Facade** (Фасад):
- Обернули модуль `fs` в упрощённую обёртку
- Переопределили только часть функциональности
- Остальной API остался доступным

**Отличия от чистого Proxy:**
- Proxy обычно один-к-одному повторяет интерфейс
- Фасад может упрощать или объединять интерфейсы
- В данном случае мы добавили новый метод `getStatistics()`

### Производительность: GoF Proxy vs JavaScript Proxy

```javascript
// GOF Proxy - более эффективен
class ServiceProxy {
  constructor(service) {
    this.service = service;
  }

  operation() {
    // Минимальный overhead
    return this.service.operation();
  }
}

// JavaScript Proxy - менее эффективен
const serviceProxy = new Proxy(service, {
  get(target, property) {
    // Больший overhead на каждое обращение
    return target[property];
  }
});
```

**Результаты бенчмарков (примерно):**
- GoF Proxy: ~5-10% overhead
- JavaScript Proxy: ~50-200% overhead

**Когда использовать каждый подход:**

| Критерий | GoF Proxy | JavaScript Proxy |
|----------|-----------|------------------|
| Производительность важна | ✅ Да | ❌ Нет |
| Нужна максимальная гибкость | ❌ Нет | ✅ Да |
| Известен конкретный интерфейс | ✅ Да | — Не важно |
| Динамическое поведение | ❌ Сложнее | ✅ Легко |
| Production код | ✅ Предпочтительно | ⚠️ С осторожностью |
| Прототипирование/отладка | — Можно | ✅ Удобно |

## Дополнительные примеры использования Proxy

### Пример 3: Валидация данных

```javascript
function createValidatedUser(userData) {
  return new Proxy(userData, {
    set(target, property, value) {
      if (property === 'age') {
        if (typeof value !== 'number' || value < 0 || value > 150) {
          throw new TypeError('Возраст должен быть числом от 0 до 150');
        }
      }

      if (property === 'email') {
        if (!value.includes('@')) {
          throw new TypeError('Email должен содержать @');
        }
      }

      target[property] = value;
      return true;
    }
  });
}

const user = createValidatedUser({
  name: 'John',
  age: 30,
  email: 'john@example.com'
});

user.age = 25; // OK
user.age = -5; // Ошибка: Возраст должен быть числом от 0 до 150
user.email = 'invalid'; // Ошибка: Email должен содержать @
```

### Пример 4: Кэширование результатов

```javascript
class ExpensiveCalculator {
  calculate(n) {
    console.log(`Вычисление для ${n}...`);
    // Долгая операция
    let result = 0;
    for (let i = 0; i <= n; i++) {
      result += i;
    }
    return result;
  }
}

class CachingProxy {
  constructor(calculator) {
    this.calculator = calculator;
    this.cache = new Map();
  }

  calculate(n) {
    if (this.cache.has(n)) {
      console.log(`Возврат из кэша для ${n}`);
      return this.cache.get(n);
    }

    const result = this.calculator.calculate(n);
    this.cache.set(n, result);
    return result;
  }
}

const calculator = new ExpensiveCalculator();
const proxy = new CachingProxy(calculator);

console.log(proxy.calculate(1000)); // Вычисление для 1000...
console.log(proxy.calculate(1000)); // Возврат из кэша для 1000
console.log(proxy.calculate(2000)); // Вычисление для 2000...
```

### Пример 5: Контроль доступа

```javascript
class SecureDatabase {
  constructor() {
    this.data = {
      users: ['Alice', 'Bob'],
      admin: ['SuperUser']
    };
  }

  getData(table) {
    return this.data[table];
  }
}

class AccessControlProxy {
  constructor(database, userRole) {
    this.database = database;
    this.userRole = userRole;
  }

  getData(table) {
    // Проверка прав доступа
    if (table === 'admin' && this.userRole !== 'admin') {
      throw new Error('Доступ запрещён: требуются права администратора');
    }

    console.log(`Доступ разрешён для роли ${this.userRole} к таблице ${table}`);
    return this.database.getData(table);
  }
}

const db = new SecureDatabase();

const userProxy = new AccessControlProxy(db, 'user');
console.log(userProxy.getData('users')); // OK: ['Alice', 'Bob']
console.log(userProxy.getData('admin')); // Ошибка: Доступ запрещён

const adminProxy = new AccessControlProxy(db, 'admin');
console.log(adminProxy.getData('admin')); // OK: ['SuperUser']
```

## Паттерны, связанные с Proxy

### Proxy vs Adapter

**Adapter** (Адаптер):
- Преобразует интерфейс одного класса в интерфейс другого
- Делает совместимыми несовместимые интерфейсы
- Изменяет сигнатуры методов

**Proxy**:
- Сохраняет тот же интерфейс
- Добавляет дополнительное поведение
- Не изменяет сигнатуры

```javascript
// Adapter - изменяет интерфейс
class OldAPI {
  getUser(id) {
    return { userId: id, userName: 'John' };
  }
}

class NewAPIAdapter {
  constructor(oldAPI) {
    this.oldAPI = oldAPI;
  }

  // Новый интерфейс: fetchUserById вместо getUser
  fetchUserById(id) {
    const oldFormat = this.oldAPI.getUser(id);
    // Преобразование формата
    return {
      id: oldFormat.userId,
      name: oldFormat.userName
    };
  }
}

// Proxy - сохраняет интерфейс
class UserAPIProxy {
  constructor(api) {
    this.api = api;
  }

  // Тот же метод getUser
  getUser(id) {
    console.log(`Запрос пользователя ${id}`);
    return this.api.getUser(id);
  }
}
```

### Proxy vs Decorator

**Decorator** (Декоратор):
- Динамически добавляет новые обязанности объекту
- Можно комбинировать несколько декораторов
- Обычно добавляет новую функциональность

**Proxy**:
- Контролирует доступ к объекту
- Обычно один прокси на объект
- Фокус на управлении доступом

Эти паттерны очень похожи, и граница между ними размыта.

### Proxy vs Facade

**Facade** (Фасад):
- Предоставляет упрощённый интерфейс к сложной подсистеме
- Может объединять несколько классов
- Упрощает использование

**Proxy**:
- Работает с одним объектом
- Сохраняет тот же интерфейс
- Не упрощает, а контролирует

## Резюме

### Ключевые концепции

1. **Паттерн Proxy** позволяет перехватывать управление над объектом и добавлять дополнительную функциональность
2. **JavaScript Proxy** и **GoF Proxy** решают одну задачу разными способами
3. **Делегирование** — ключевая техника реализации GoF Proxy
4. **Замыкание модуля** — полезная техника для хранения приватного состояния

### Когда использовать Proxy

**Используйте паттерн Proxy, когда:**
- ✅ Нужно добавить функциональность без наследования
- ✅ Требуется ленивая инициализация
- ✅ Необходимо кэширование результатов
- ✅ Нужен контроль доступа
- ✅ Требуется логирование или мониторинг
- ✅ Важна производительность (используйте GoF Proxy)

**Избегайте паттерна Proxy, когда:**
- ❌ Overhead на производительность неприемлем
- ❌ Код становится слишком сложным
- ❌ Простое наследование решает задачу лучше

### Выбор между подходами

```javascript
// JavaScript Proxy - для максимальной гибкости
const proxy = new Proxy(obj, {
  get(target, property) { /* ... */ },
  set(target, property, value) { /* ... */ },
  // ... много других перехватчиков
});

// GoF Proxy - для производительности и ясности
class OptimizedProxy {
  constructor(obj) {
    this.obj = obj;
  }

  method() {
    // Контролируемое делегирование
    return this.obj.method();
  }
}
```

### Практические рекомендации

1. **Для production кода** отдавайте предпочтение GoF Proxy
2. **Для отладки и прототипирования** используйте JavaScript Proxy
3. **Комбинируйте** с замыканием модуля для приватного состояния
4. **Используйте TypeScript** для явного определения интерфейсов
5. **Документируйте** добавленную прокси функциональность
6. **Измеряйте** влияние на производительность

## Дополнительные материалы

### Связанные паттерны
- **Adapter** — преобразование интерфейсов
- **Decorator** — динамическое добавление функциональности
- **Facade** — упрощение сложных подсистем

### Связанные концепции
- **Делегирование** — передача вызовов другому объекту
- **Замыкание** — сохранение приватного состояния
- **Композиция** — альтернатива наследованию

### Применение в реальных проектах
- Фреймворки для реактивности (Vue.js, MobX)
- ORM библиотеки (Sequelize, TypeORM)
- Системы логирования и мониторинга
- Библиотеки для работы с API
- Инструменты для тестирования (моки и стабы)
