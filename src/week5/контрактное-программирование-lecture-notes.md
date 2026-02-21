# Контрактное программирование с примерами на JavaScript и Node.js

## Обзор

Контрактное программирование (Contract Programming) — это подход к разработке программного обеспечения, который позволяет формализовать требования к функциям, методам и компонентам системы через декларативное описание входных параметров, выходных значений и поведения. Контракты проверяются в runtime и обеспечивают дополнительный уровень надёжности и безопасности приложений.

Эта лекция рассматривает:
- Отличие контрактов от протоколов и интерфейсов
- Практическое применение контрактов в JavaScript/Node.js
- Декларативный подход к описанию контрактов
- Примеры из технологического стека MetarhiaJS
- Преимущества контрактного программирования

---

## Протоколы в JavaScript

### Что такое протокол?

Протокол в JavaScript — это нечто похожее на интерфейс, но более широкое понятие. Протокол включает в себя:
- **Сигнатуры методов** (названия, параметры, возвращаемые значения)
- **Предсказуемую последовательность действий**
- **Конкретное поведение** (не только типы, но и логику выполнения)

В отличие от классических интерфейсов, протоколы в JavaScript проверяются тестами, а не компилятором.

### Примеры встроенных протоколов

#### 1. Протокол Iterator (Итератор)

Итератор — это объект с методом `next()`, который возвращает структуру `{ value, done }`.

**Пример базового итератора:**

```javascript
// Простейший итератор, возвращающий бесконечную последовательность
const iterator = {
  current: 0,
  next() {
    return {
      value: this.current++,
      done: false  // Бесконечный итератор
    };
  }
};

// Использование
const it1 = iterator.next(); // { value: 0, done: false }
const it2 = iterator.next(); // { value: 1, done: false }
const it3 = iterator.next(); // { value: 2, done: false }
```

**Ключевые моменты:**
- `done: false` означает, что итерация не закончена
- В примере выше итератор никогда не завершается (бесконечный)
- Протокол определяет структуру возвращаемого объекта

#### 2. Протокол Iterable

Iterable — это более высокоуровневый протокол. Объект является iterable, если у него определён символ `Symbol.iterator`, который возвращает объект, соответствующий протоколу Iterator.

**Пример создания iterable объекта:**

```javascript
// Создаём кастомный iterable объект
const myIterable = {
  data: [10, 20, 30, 40],

  // Определяем метод Symbol.iterator
  [Symbol.iterator]() {
    let index = 0;
    const data = this.data;

    // Возвращаем объект итератора
    return {
      next() {
        if (index < data.length) {
          return { value: data[index++], done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

// Теперь можно использовать в for...of
for (const value of myIterable) {
  console.log(value); // 10, 20, 30, 40
}

// Или использовать spread оператор
const arr = [...myIterable]; // [10, 20, 30, 40]
```

**Важно:** Это не массив и не коллекция, а просто объект с ключом `Symbol.iterator`, соответствующий контракту iterable.

#### 3. Протокол Thenable

Thenable — это протокол, на базе которого построены Promise. Объект является thenable, если у него есть метод `then()`.

**Пример:**

```javascript
// Простейший thenable объект
const thenable = {
  then(onFulfilled, onRejected) {
    setTimeout(() => {
      onFulfilled('Success!');
    }, 1000);
  }
};

// Можно использовать с Promise
Promise.resolve(thenable).then(result => {
  console.log(result); // 'Success!' (через 1 секунду)
});
```

**Важное замечание:**
- Promise всегда соответствует протоколу Thenable
- Но не каждый Thenable является Promise
- Протокол проверяется через тесты, а не статически

#### 4. Асинхронный итератор (AsyncIterable)

Для асинхронных операций существует протокол AsyncIterable, где:
- Метод `next()` возвращает Promise
- Результаты упаковываются в Promise
- Используется с `for await...of`

**Пример асинхронного итератора:**

```javascript
const asyncIterable = {
  data: [1, 2, 3, 4, 5],

  [Symbol.asyncIterator]() {
    let index = 0;
    const data = this.data;

    return {
      async next() {
        // Имитация асинхронной операции
        await new Promise(resolve => setTimeout(resolve, 100));

        if (index < data.length) {
          return { value: data[index++], done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
};

// Использование с for await...of
async function processData() {
  for await (const value of asyncIterable) {
    console.log(value); // 1, 2, 3, 4, 5 (с задержками)
  }
}

processData();
```

---

## Что такое контракт?

### Определение

**Контракт** — это протокол, который проверяется в runtime (во время выполнения). В отличие от статической типизации (TypeScript), контракты:
- Проверяются динамически во время выполнения
- Могут быть включены/выключены по необходимости
- Работают на стыке систем (межпроцессное взаимодействие, API, базы данных)
- Описываются декларативно, а не императивно

### Контракт vs Протокол vs Интерфейс

| Аспект | Интерфейс | Протокол | Контракт |
|--------|-----------|----------|----------|
| Проверка типов | Compile-time | Runtime (тесты) | Runtime (автоматически) |
| Поведение | Не описывает | Описывает | Описывает |
| Последовательность | Не контролирует | Контролирует | Контролирует |
| Декларативность | Средняя | Низкая | Высокая |
| Применение | Статическая типизация | Спецификации JS | API, межсистемное взаимодействие |

---

## Императивная проверка контракта

### Проблема императивного подхода

Рассмотрим простую функцию сравнения двух чисел:

```javascript
const assert = require('assert');

function compare(a, b) {
  // Императивные проверки контракта
  assert(typeof a === 'number', 'Parameter "a" must be a number');
  assert(typeof b === 'number', 'Parameter "b" must be a number');

  // Бизнес-логика
  const result = a > b;

  // Проверка результата
  assert(typeof result === 'boolean', 'Result must be a boolean');

  return result;
}

// Тестирование
console.log(compare(7, 5)); // true
console.log(compare(5, 7)); // false

// Это вызовет ошибку
try {
  console.log(compare('7', 5));
} catch (err) {
  console.error('Contract violation:', err.message);
  // Contract violation: Parameter "a" must be a number
}
```

**Недостатки императивного подхода:**
1. ❌ Много повторяющегося кода
2. ❌ Легко допустить ошибку в самих проверках
3. ❌ Код проверок смешивается с бизнес-логикой
4. ❌ Снижается читаемость
5. ❌ Трудно поддерживать

---

## Декларативное описание контрактов

### Основная идея

Вместо императивных проверок мы описываем контракт декларативно — в виде метаданных, отделённых от реализации метода.

### Пример из технологического стека MetarhiaJS

```javascript
// Декларативное описание контракта для метода compare
const methodContract = {
  // Сам метод — только бизнес-логика!
  method: async (a, b) => a > b,

  // Описание входных параметров
  parameters: {
    a: 'number',
    b: 'number'
  },

  // Описание возвращаемого значения
  returns: 'boolean'
};
```

**Преимущества:**
- ✅ Чистая бизнес-логика без проверок
- ✅ Явное определение контракта
- ✅ Проверки выполняются автоматически
- ✅ Легко читать и поддерживать

### Сложные структуры данных

Контракты могут описывать вложенные объекты:

```javascript
const createUserContract = {
  method: async (name, age, address) => {
    // Бизнес-логика создания пользователя
    return { id: generateId(), name, age, address };
  },

  parameters: {
    name: 'string',
    age: 'number',
    address: {
      // Вложенная структура
      city: 'string',
      street: 'string',
      building: 'number'
    }
  },

  returns: {
    id: 'string',
    name: 'string',
    age: 'number',
    address: 'Address'  // Ссылка на внешнюю схему
  }
};
```

### Ссылки на внешние схемы

Можно ссылаться на предопределённые типы:

```javascript
// Определение переиспользуемой схемы
const schemas = {
  Account: {
    login: 'string',
    passwordHash: 'string',
    email: 'string',
    address: 'Address'  // Ссылка на другую схему
  },

  Address: {
    city: 'string',
    street: 'string',
    building: 'number',
    apartment: { type: 'number', optional: true }
  }
};

// Использование в контракте
const updateAccountContract = {
  method: async (accountId, data) => {
    // Обновление аккаунта
  },

  parameters: {
    accountId: 'string',
    data: 'Account'  // Ссылка на схему Account
  },

  returns: 'Account'
};
```

---

## Проверка контрактов в Runtime

### Механизм работы

Когда метод с контрактом вызывается, специальный **runner** (исполнитель):
1. Читает поля `parameters` и `returns`
2. Проверяет входящие данные перед вызовом метода
3. Выполняет метод
4. Проверяет возвращаемое значение
5. Выбрасывает исключение при нарушении контракта

**Псевдокод runner'а:**

```javascript
function executeWithContract(contractDefinition, ...args) {
  const { method, parameters, returns } = contractDefinition;

  // 1. Проверка входных параметров
  validateParameters(args, parameters);

  // 2. Выполнение метода
  const result = await method(...args);

  // 3. Проверка результата
  validateResult(result, returns);

  return result;
}

function validateParameters(args, parametersSchema) {
  // Проверка типов и структуры аргументов
  Object.keys(parametersSchema).forEach((key, index) => {
    const expectedType = parametersSchema[key];
    const actualValue = args[index];

    if (!checkType(actualValue, expectedType)) {
      throw new Error(`Contract violation: parameter "${key}" expected ${expectedType}`);
    }
  });
}

function validateResult(result, returnsSchema) {
  // Проверка типа и структуры возвращаемого значения
  if (!checkType(result, returnsSchema)) {
    throw new Error(`Contract violation: return value expected ${returnsSchema}`);
  }
}
```

---

## Применение контрактов в распределённых системах

### Проблема межсистемного взаимодействия

TypeScript и статическая типизация работают только на этапе компиляции и **не помогают** в следующих случаях:
- API вызовы между клиентом и сервером
- Межпроцессное взаимодействие
- Чтение данных из базы данных
- Интеграция с внешними системами
- Чтение данных из файлов или очередей

В этих сценариях контракты **незаменимы**.

### Архитектура клиент-сервер с контрактами

```javascript
// ========== СЕРВЕР ==========
// Сервер описывает и публикует контракт
const serverAPI = {
  compare: {
    method: async (a, b) => a > b,
    parameters: { a: 'number', b: 'number' },
    returns: 'boolean'
  }
};

// ========== КЛИЕНТ ==========
// Клиент вызывает метод, как будто это локальная функция
const result = await api.example.compare(7, 5);
console.log(result); // true

// Но на самом деле:
// 1. Проверка контракта на клиенте (опционально)
// 2. Сериализация и отправка запроса на сервер
// 3. Проверка контракта на сервере
// 4. Выполнение метода
// 5. Проверка результата на сервере
// 6. Отправка результата клиенту
// 7. Проверка результата на клиенте (опционально)
```

### Двусторонняя проверка

**На клиенте:**
- Контракт проверяется до отправки запроса
- Экономит трафик и время при некорректных данных
- Обеспечивает быструю валидацию

**На сервере:**
- Контракт проверяется обязательно (защита от недоверенных клиентов)
- Гарантирует корректность входных данных
- Проверяет результат перед отправкой

```javascript
// Пример обработки ошибок контракта
try {
  // Клиент пытается вызвать с неправильными типами
  await api.example.compare('7', 5);
} catch (err) {
  // Ошибка возникнет ещё на клиенте (экономия запроса!)
  console.error('Contract error:', err.message);
  // Contract error: parameter "a" expected number, got string
}
```

---

## Практические примеры контрактов

### Пример 1: Регистрация пользователя

```javascript
const registerUserContract = {
  method: async (name, email, address) => {
    // Бизнес-логика: создание пользователя, запись в БД
    const user = {
      id: generateUUID(),
      name,
      email,
      address,
      createdAt: new Date()
    };

    await db.users.insert(user);
    return user;
  },

  // Входные параметры
  parameters: {
    name: 'string',
    email: 'string',
    address: {
      city: 'string',
      street: 'string',
      building: 'number'
    }
  },

  // Возвращаемая структура
  returns: {
    id: 'string',
    name: 'string',
    email: 'string',
    address: 'Address',
    createdAt: 'Date'
  },

  // Возможные ошибки (опционально)
  errors: {
    'EMAIL_EXISTS': 'User with this email already exists',
    'INVALID_ADDRESS': 'Address validation failed'
  }
};
```

### Пример 2: Работа с внешним API

```javascript
// Описание контракта внешнего API
const externalAPIContract = {
  getUserData: {
    method: async (userId) => {
      const response = await fetch(`https://api.example.com/users/${userId}`);
      return response.json();
    },

    parameters: {
      userId: 'string'
    },

    returns: {
      id: 'string',
      name: 'string',
      email: 'string',
      profile: {
        age: 'number',
        country: 'string'
      }
    },

    errors: {
      'USER_NOT_FOUND': 'User does not exist',
      'API_ERROR': 'External API returned an error'
    }
  }
};

// Использование
try {
  const userData = await externalAPI.getUserData('user-123');
  console.log(userData.name);
} catch (err) {
  if (err.code === 'USER_NOT_FOUND') {
    console.log('User not found');
  }
}
```

---

## Генерация на основе контрактов

Контракты — это не только валидация. Они служат **единым источником истины** для множества задач.

### 1. Генерация структуры базы данных

Из контракта можно автоматически создать SQL-схему:

```javascript
// Контракт
const UserSchema = {
  id: 'string',
  name: 'string',
  email: 'string',
  age: 'number',
  createdAt: 'Date'
};

// Автоматическая генерация SQL
function generateCreateTable(schemaName, schema) {
  const columns = Object.entries(schema).map(([field, type]) => {
    const sqlType = typeToSQL(type);
    return `${field} ${sqlType}`;
  });

  return `CREATE TABLE ${schemaName} (${columns.join(', ')})`;
}

// Результат:
// CREATE TABLE User (
//   id VARCHAR(255),
//   name VARCHAR(255),
//   email VARCHAR(255),
//   age INTEGER,
//   createdAt TIMESTAMP
// )
```

### 2. Генерация документации

Контракты — это готовая документация API:

```javascript
// Автоматическая генерация Markdown документации
function generateDocs(contracts) {
  return Object.entries(contracts).map(([name, contract]) => `
## ${name}

**Parameters:**
${formatParameters(contract.parameters)}

**Returns:**
${formatReturns(contract.returns)}

**Possible Errors:**
${formatErrors(contract.errors)}
  `).join('\n\n');
}
```

### 3. Генерация тестовых данных

Контракты позволяют автоматически генерировать данные для тестов:

```javascript
function generateTestData(schema) {
  const generators = {
    string: () => Math.random().toString(36).substring(7),
    number: () => Math.floor(Math.random() * 1000),
    boolean: () => Math.random() > 0.5,
    Date: () => new Date()
  };

  const data = {};

  for (const [key, type] of Object.entries(schema)) {
    if (typeof type === 'object') {
      // Рекурсивная генерация для вложенных объектов
      data[key] = generateTestData(type);
    } else {
      data[key] = generators[type]();
    }
  }

  return data;
}

// Использование
const testUser = generateTestData({
  name: 'string',
  age: 'number',
  address: {
    city: 'string',
    building: 'number'
  }
});

console.log(testUser);
// {
//   name: 'x7k9m2',
//   age: 742,
//   address: { city: 'p4q8r1', building: 123 }
// }
```

### 4. Автоматическая генерация тестов

На основе контракта можно генерировать:
- **Позитивные тесты** (валидные данные)
- **Негативные тесты** (невалидные данные)

```javascript
function generateTests(contractName, contract) {
  const tests = [];

  // Позитивный тест
  tests.push({
    name: `${contractName} - should work with valid data`,
    run: async () => {
      const validData = generateTestData(contract.parameters);
      const result = await executeWithContract(contract, ...Object.values(validData));
      assert(checkType(result, contract.returns));
    }
  });

  // Негативные тесты для каждого параметра
  Object.keys(contract.parameters).forEach(paramName => {
    tests.push({
      name: `${contractName} - should fail with invalid ${paramName}`,
      run: async () => {
        const invalidData = generateTestData(contract.parameters);
        invalidData[paramName] = 'INVALID_VALUE';

        await assert.rejects(
          executeWithContract(contract, ...Object.values(invalidData))
        );
      }
    });
  });

  return tests;
}
```

---

## Преимущества контрактного программирования

### 1. Надёжность и безопасность

- ✅ Гарантированная валидация на границах систем
- ✅ Защита от некорректных данных
- ✅ Предотвращение ошибок в runtime
- ✅ Ранняя детекция проблем

### 2. Разделение ответственности

```javascript
// БЕЗ контрактов - смешение логики
function processPayment(amount, currency, userId) {
  // Валидация смешана с бизнес-логикой
  if (typeof amount !== 'number') throw new Error('Invalid amount');
  if (amount <= 0) throw new Error('Amount must be positive');
  if (typeof currency !== 'string') throw new Error('Invalid currency');
  if (!['USD', 'EUR', 'GBP'].includes(currency)) throw new Error('Unsupported currency');
  if (typeof userId !== 'string') throw new Error('Invalid userId');

  // Бизнес-логика теряется среди проверок
  const fee = amount * 0.03;
  const total = amount + fee;
  return { total, fee, currency };
}

// С контрактами - чистая бизнес-логика
const processPaymentContract = {
  method: async (amount, currency, userId) => {
    // Только бизнес-логика!
    const fee = amount * 0.03;
    const total = amount + fee;
    return { total, fee, currency };
  },

  parameters: {
    amount: { type: 'number', min: 0 },
    currency: { type: 'string', enum: ['USD', 'EUR', 'GBP'] },
    userId: 'string'
  },

  returns: {
    total: 'number',
    fee: 'number',
    currency: 'string'
  }
};
```

### 3. Культура разработки

**Контракт как спецификация:**
- Контракт можно передать команде до реализации
- Фронтенд-разработчики могут начать работу, зная контракт
- QA может писать тесты параллельно с разработкой
- Документация генерируется автоматически

**Пример процесса:**
```
1. Backend описывает контракт API
2. Backend передаёт контракт Frontend команде
3. Frontend начинает разработку интерфейса
4. QA пишет тесты на основе контракта
5. Backend реализует методы
6. Интеграция проходит гладко (контракт уже согласован!)
```

### 4. Оптимизация производительности

Контракты можно включать/выключать:

```javascript
const config = {
  enableContracts: process.env.NODE_ENV === 'development'
};

function executeMethod(contract, ...args) {
  if (config.enableContracts) {
    // Проверяем контракт
    validateParameters(args, contract.parameters);
    const result = await contract.method(...args);
    validateResult(result, contract.returns);
    return result;
  } else {
    // Пропускаем проверки в production
    return contract.method(...args);
  }
}
```

**Стратегия:**
1. Включить контракты в development и testing
2. Убедиться, что все работает корректно
3. Отключить контракты в production для производительности
4. Прирост производительности может быть значительным (10-30%)

### 5. Тестирование

**Без ручного написания базовых тестов:**
- Автоматическая генерация тестов на типы
- Автоматическая генерация позитивных/негативных сценариев
- Фокус на бизнес-логике, а не на проверке типов

**Экономия времени:**
- Не нужно писать тесты для проверки типов вручную
- Не нужно тестировать валидацию вручную
- Можно сосредоточиться на сложных edge cases

---

## Ограничения контрактов

### Что контракты НЕ делают

❌ **Не описывают поведение полностью**
- Контракты описывают интерфейсы и типы
- Сложная бизнес-логика требует отдельных тестов
- Контракты не заменяют unit-тесты полностью

❌ **Не проверяют алгоритмы**
```javascript
// Контракт проверит типы, но не логику
const sortContract = {
  method: async (arr) => {
    // Контракт не проверит, ПРАВИЛЬНО ли отсортирован массив!
    return arr.sort(); // Может быть баг в сортировке
  },
  parameters: { arr: 'Array' },
  returns: 'Array'
};
```

❌ **Не гарантируют корректность бизнес-логики**
```javascript
// Контракт не поймает логическую ошибку
const calculateDiscountContract = {
  method: async (price, discountPercent) => {
    // БАГ: должно быть (1 - discountPercent/100)
    return price * discountPercent;  // Неправильная формула!
  },
  parameters: { price: 'number', discountPercent: 'number' },
  returns: 'number'  // Тип правильный, но результат — нет!
};
```

### Когда нужны дополнительные инструменты

Для описания сложных протоколов взаимодействия (таких как Thenable, Iterable) нужны специализированные DSL (Domain Specific Languages) или более продвинутые системы контрактов, которые могут описывать:
- Последовательность вызовов методов
- Состояния объектов
- Временные зависимости
- Сложные инварианты

---

## История и происхождение

### Язык Eiffel

Контрактное программирование как концепция было впервые встроено в язык программирования **Eiffel** (разработан Бертраном Мейером в 1980-х).

**Особенности Eiffel:**
- Контракты как часть синтаксиса языка
- Preconditions (предусловия)
- Postconditions (постусловия)
- Class invariants (инварианты класса)

**Пример на Eiffel (для понимания концепции):**
```eiffel
deposit(amount: INTEGER)
  require
    amount > 0  -- Предусловие
  ensure
    balance = old balance + amount  -- Постусловие
```

### Применение в других языках

С тех пор идеи контрактного программирования распространились на многие языки:
- **C#**: Code Contracts
- **Python**: библиотеки типа `dpcontracts`, `contracts`
- **Java**: различные библиотеки (например, Cofoja)
- **JavaScript/Node.js**: пользовательские реализации (MetarhiaJS и другие)

---

## Резюме

### Ключевые концепции

1. **Протоколы** — это более широкое понятие, чем интерфейсы, включающее поведение
2. **Контракты** — это протоколы с runtime-проверкой
3. **Декларативное описание** контрактов лучше императивного
4. **Контракты незаменимы** на границах систем (API, БД, файлы)
5. **Генерация на основе контрактов** экономит время и снижает ошибки

### Практические рекомендации

✅ **Используйте контракты для:**
- API методов
- Межсистемного взаимодействия
- Работы с внешними данными (БД, файлы, очереди)
- Публичных интерфейсов библиотек

✅ **Описывайте контракты декларативно:**
- Отделяйте контракт от реализации
- Используйте метаданные
- Ссылайтесь на переиспользуемые схемы

✅ **Автоматизируйте:**
- Генерируйте тесты из контрактов
- Генерируйте документацию
- Генерируйте схемы БД

✅ **Оптимизируйте:**
- Включайте контракты в development/testing
- Отключайте в production после проверки
- Оставляйте возможность включить при необходимости

### Дополнительная ценность контрактов

Контракты — это не просто валидация. Это:
- **Документация** (формализованная и точная)
- **Спецификация** (описание требований)
- **Основа для генерации** (тесты, БД, код)
- **Культура разработки** (согласование интерфейсов заранее)
- **Инструмент надёжности** (защита границ системы)

---

## Дополнительные материалы для изучения

### Рекомендуемые темы для углубления

1. **Design by Contract** (концепция Бертрана Мейера)
2. **JSON Schema** — стандарт для описания JSON структур
3. **OpenAPI/Swagger** — спецификация для REST API
4. **gRPC и Protocol Buffers** — альтернативный подход к контрактам
5. **MetarhiaJS** — технологический стек с встроенной поддержкой контрактов

### Практические задания

1. Реализуйте простой runner для проверки контрактов
2. Создайте генератор тестовых данных на основе схем
3. Напишите функцию генерации SQL из контрактов
4. Реализуйте систему включения/выключения контрактов
5. Создайте декоратор для автоматической проверки контрактов функций

---

**Заключение:** Контрактное программирование — это мощный инструмент для создания надёжных, поддерживаемых и хорошо документированных систем, особенно в условиях распределённой архитектуры и межсистемного взаимодействия. Декларативное описание контрактов позволяет отделить бизнес-логику от валидации и открывает возможности для автоматизации множества задач разработки.
